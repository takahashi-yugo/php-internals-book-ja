イテレーター
============

先ほどのセクションでは、型付き配列のPHPの言語への統合が上手くいくように、いくつかオブジェクトハンドラーを実装しました。しかしまだ、考慮しなければならないことがあり、それはイテレーションです。このセクションでは、PHPの内部でどのようにイテレーションが実装されているかや、それをどのように利用できるかといった点をみていきたいと思います。ここでも例として、型付き配列を使用していきます。

get_iteratorハンドラー
-----------------------

PHPの内部でのイテレーションの動作は、ユーザーランドでの ``IteratorAggregate`` インターフェイスと非常によく似ています。このクラスは ``zend_object_iterator*`` を返す ``get_iterator`` ハンドラーを持っており、以下のようになっています。

.. code-block:: c

  struct _zend_object_iterator {
      void *data;
      zend_object_iterator_funcs *funcs;
      ulong index; /* fe_reset/fe_fetch オペコードでのみ使用 */
  };

``index`` メンバーは内部的に ``foreach`` の実装に使用されています。これが各イテレーションの度にインクリメントされ、もし独自のキー関数を指定しなければ、キーとして使用されます。 ``funcs`` メンバーにはそれぞれのイテレーションでの動作のためのハンドラーが含まれています。

.. code-block:: c

  typedef struct _zend_object_iterator_funcs {
      /* このイテレーターインスタンスに関連する全てのリソースを解放する */
      void (*dtor)(zend_object_iterator *iter TSRMLS_DC);  

      /* イテレーションの最後かどうかをチェックする (データが妥当であれば SUCCESSを、そうでなければFAILUREを返す) */
      int (*valid)(zend_object_iterator *iter TSRMLS_DC);  

      /* 現在の要素からデータを取得する */
      void (*get_current_data)(zend_object_iterator *iter, zval ***data TSRMLS_DC);  

      /* 現在の要素からキーを取得する (オプション。 NULLかもしれない) */
      void (*get_current_key)(zend_object_iterator *iter, zval *key TSRMLS_DC);  

      /* 次の要素にすすむ */
      void (*move_forward)(zend_object_iterator *iter TSRMLS_DC);  

      /* 最初の要素に巻き戻す (オプション。 NULLかもしれない) */
      void (*rewind)(zend_object_iterator *iter TSRMLS_DC);  

      /* 現在の値やキーを無効にする (オプション。 NULLかもしれない) */
      void (*invalidate_current)(zend_object_iterator *iter TSRMLS_DC);
  } zend_object_iterator_funcs;

このハンドラーは ``Iterator`` インターフェイスのものとそれぞれ名前が僅かに違うだけで、非常によく似ています。唯一、 ``invalidate_current`` はユーザーランドの方に対応するものがありませんが、これは現在のキー/値を破棄するために使用されます。しかしこれはほとんど使われておらず、特に ``foreach`` からもこれが呼び出されることすらありません。

イテレーターの構造体の最後に紹介するメンバーは ``data`` で、独自のデータを持ちまわることができます。大抵、1スロットでは十分でないので、そういう時は、 ``zend_object`` での場合と同様に構造体を拡張します。

型付き配列をイテレートするために、いくつかのデータを保持しておかなければなりません。まず最初に、バッファビューオブジェクトへの参照を持っておく必要があります（さもないと、イテレーションの間に破棄されるかもしれません）。その参照は ``data`` メンバーで保持しておくとよいでしょう。さらに、 ``buffer_view_object`` をハンドラーが呼び出される度に取得しなおさなくてもよいように、手元に置いておく必要があります。また、現在のイテレーションの ``offset`` と、現在の値の ``zval*`` を保持しておく必要があります(これが必要な理由は後ほど説明します)。

.. code-block:: c

  typedef struct _buffer_view_iterator {
      zend_object_iterator intern;
      buffer_view_object *view;
      size_t offset;
      zval *current;
  } buffer_view_iterator;

では準備が整ったので、例として ``zend_object_iterator_funcs`` 構造体を宣言してみましょう。

.. code-block:: c

  static zend_object_iterator_funcs buffer_view_iterator_funcs = {
      buffer_view_iterator_dtor,
      buffer_view_iterator_valid,
      buffer_view_iterator_get_current_data,
      buffer_view_iterator_get_current_key,
      buffer_view_iterator_move_forward,
      buffer_view_iterator_rewind
  };

では ``get_iterator`` ハンドラーを実装していきましょう。このハンドラーはクラスエントリとそのオブジェクト、イテレーションが参照でおこなわれているかどうかのフラグを受け取って、 ``zend_object_iterator*`` を返します。ここで必要になってくるのは、イテレーターのメモリ割り当てとそれぞれのメンバーの初期化をおこなうということだけです。

.. code-block:: c

  zend_object_iterator *buffer_view_get_iterator(
      zend_class_entry *ce, zval *object, int by_ref TSRMLS_DC
  ) {
      buffer_view_iterator *iter;  

      if (by_ref) {
          zend_throw_exception(NULL, "Cannot iterate buffer view by reference", 0 TSRMLS_CC);
          return NULL;
      }  

      iter = emalloc(sizeof(buffer_view_iterator));
      iter->intern.funcs = &buffer_view_iterator_funcs;  

      iter->intern.data = object;
      Z_ADDREF_P(object);  

      iter->view = zend_object_store_get_object(object TSRMLS_CC);
      iter->offset = 0;
      iter->current = NULL;  

      return (zend_object_iterator *) iter;
  }

最後に、バッファビュークラスの登録のためにマクロを調整しなければなりません。

.. code-block:: c

  #define DEFINE_ARRAY_BUFFER_VIEW_CLASS(class_name, type)                     \
      INIT_CLASS_ENTRY(tmp_ce, #class_name, array_buffer_view_functions);      \
      type##_array_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);       \
      type##_array_ce->create_object = array_buffer_view_create_object;        \
      type##_array_ce->get_iterator = buffer_view_get_iterator;                \
      type##_array_ce->iterator_funcs.funcs = &buffer_view_iterator_funcs;     \
      zend_class_implements(type##_array_ce TSRMLS_CC, 2,                      \
          zend_ce_arrayaccess, zend_ce_traversable);

これまでと比べて、``Traversable`` の実装を設定している点と、 ``get_iterator`` と ``iterator_funcs.funcs`` への代入部分が加わりました。

イテレーター関数
----------------

では上記で記述している ``buffer_view_iterator_funcs`` を実際に実装していきましょう。

.. code-block:: c

  static void buffer_view_iterator_dtor(zend_object_iterator *intern TSRMLS_DC)
  {
      buffer_view_iterator *iter = (buffer_view_iterator *) intern;  

      if (iter->current) {
          zval_ptr_dtor(&iter->current);
      }  

      zval_ptr_dtor((zval **) &intern->data);
      efree(iter);
  }  

  static int buffer_view_iterator_valid(zend_object_iterator *intern TSRMLS_DC)
  {
      buffer_view_iterator *iter = (buffer_view_iterator *) intern;  

      return iter->offset < iter->view->length ? SUCCESS : FAILURE;
  }  

  static void buffer_view_iterator_get_current_data(
      zend_object_iterator *intern, zval ***data TSRMLS_DC
  ) {
      buffer_view_iterator *iter = (buffer_view_iterator *) intern;  

      if (iter->current) {
          zval_ptr_dtor(&iter->current);
      }  

      if (iter->offset < iter->view->length) {
          iter->current = buffer_view_offset_get(iter->view, iter->offset);
          *data = &iter->current;
      } else {
          *data = NULL;
      }
  }  

  #if ZEND_MODULE_API_NO >= 20121212
  static void buffer_view_iterator_get_current_key(
      zend_object_iterator *intern, zval *key TSRMLS_DC
  ) {
      buffer_view_iterator *iter = (buffer_view_iterator *) intern;
      ZVAL_LONG(key, iter->offset);
  }
  #else
  static int buffer_view_iterator_get_current_key(
      zend_object_iterator *intern, char **str_key, uint *str_key_len, ulong *int_key TSRMLS_DC
  ) {
      buffer_view_iterator *iter = (buffer_view_iterator *) intern;  

      *int_key = (ulong) iter->offset;
      return HASH_KEY_IS_LONG;
  }
  #endif  

  static void buffer_view_iterator_move_forward(zend_object_iterator *intern TSRMLS_DC)
  {
      buffer_view_iterator *iter = (buffer_view_iterator *) intern;  

      iter->offset++;
  }  

  static void buffer_view_iterator_rewind(zend_object_iterator *intern TSRMLS_DC)
  {
      buffer_view_iterator *iter = (buffer_view_iterator *) iter;  

      iter->offset = 0;
      iter->current = NULL;
  }

それぞれの関数はかなり単純なので、簡単な説明にとどめます。

``get_current_data`` は ``zval*** data`` をパラメーターとして受け取り、それに ``*data = ...`` とすることで ``zval**`` を書き込むことを想定しています。イテレーションは参照渡しの可能性もあり、その場合は ``zval*`` だと十分でないので ``zval**`` でなければなりません。 イテレーターで currentのメンバーで ``zval*`` を保持しておかなければならないのは、このように ```zval**` を扱うからです。

``get_current_key`` の実装はPHPのバージョンに依存しています。PHP5.5では単に受け取った ``key`` の変数に ``ZVAL_*`` マクロのいづれかを使ってキーを書き込むだけです。

5.5以前のPHPでの ``get_current_key`` ハンドラーは3つのパラメーターを受け取り、これは返すキーの型に対応してセットされます。もし ``HASH_KEY_NON_EXISTANT`` を返す場合には、結果としてのキーは ``null`` となり、パラメーターには何もセットする必要がありません。 ``HASH_KEY_IS_LONG`` の場合には、 ``int_key`` をセットします。 ``HASH_KEY_IS_STRING`` の場合には、 ```str_key`` と ``str_key_len`` をセットしなければなりません。ここでの ``str_key_len`` は( ``zend_hash`` APIでの場合と同様に)文字列の長さに1を足したものであることに注意してください。

継承について
------------

ここでもユーザーがクラスを継承してイテレーションの振る舞いを変えたいと思う場合について考えなければなりません。現状では個別のイテレーションのハンドラーがユーザーランドからは隠蔽されているので、継承した際にはユーザーがイテレーションの仕組みを手動で再実装しなければなりません。

オブジェクトハンドラーで既におこなってきたように、通常の ``Iterator`` インターフェイスを実装するというかたちで解決します。今回は、PHPが実際にオーバーライドしたメソッドを呼び出すのを保証するために、特別なハンドリングは必要ありません。というのも、PHPはクラスが継承されておらず直接使用されている場合には最初の内部のハンドラーを使用しますが、クラスが拡張されている場合は ``Iterator`` のメソッドを使用するからです。

``Iterator`` メソッドを実装するためには、``buffer_view_object`` に ``size_t current_offset`` メンバーを追加して、イテレーションメソッドのために現在のオフセットを保持するようにしなければなりません(そして ``get_iterator`` のイテレーターで使われるイテレーションの状態とは完全に分離します)。メソッドの実装それ自体は、お決まりの引数のチェック処理が大半です。

.. code-block:: c

  PHP_FUNCTION(array_buffer_view_rewind)
  {
      buffer_view_object *intern;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      intern->current_offset = 0;
  }  

  PHP_FUNCTION(array_buffer_view_next)
  {
      buffer_view_object *intern;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      intern->current_offset++;
  }  

  PHP_FUNCTION(array_buffer_view_valid)
  {
      buffer_view_object *intern;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      RETURN_BOOL(intern->current_offset < intern->length);
  }  

  PHP_FUNCTION(array_buffer_view_key)
  {
      buffer_view_object *intern;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      RETURN_LONG((long) intern->current_offset);
  }  

  PHP_FUNCTION(array_buffer_view_current)
  {
      buffer_view_object *intern;
      zval *value;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      value = buffer_view_offset_get(intern, intern->current_offset);
      RETURN_ZVAL(value, 1, 1);
  }  

  /* ... */  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_buffer_view_void, 0, 0, 0)
  ZEND_END_ARG_INFO()  

  /* ... */  

  PHP_ME_MAPPING(rewind, array_buffer_view_rewind, arginfo_buffer_view_void, ZEND_ACC_PUBLIC)
  PHP_ME_MAPPING(next, array_buffer_view_next, arginfo_buffer_view_void, ZEND_ACC_PUBLIC)
  PHP_ME_MAPPING(valid, array_buffer_view_valid, arginfo_buffer_view_void, ZEND_ACC_PUBLIC)
  PHP_ME_MAPPING(key, array_buffer_view_key, arginfo_buffer_view_void, ZEND_ACC_PUBLIC)
  PHP_ME_MAPPING(current, array_buffer_view_current, arginfo_buffer_view_void, ZEND_ACC_PUBLIC)

当然ですが、 ``Traversable`` ではなく ``Iterator`` の実装を設定しなければなりません。

.. code-block:: c

  #define DEFINE_ARRAY_BUFFER_VIEW_CLASS(class_name, type)                     \
      INIT_CLASS_ENTRY(tmp_ce, #class_name, array_buffer_view_functions);      \
      type##_array_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);       \
      type##_array_ce->create_object = array_buffer_view_create_object;        \
      type##_array_ce->get_iterator = buffer_view_get_iterator;                \
      type##_array_ce->iterator_funcs.funcs = &buffer_view_iterator_funcs;     \
      zend_class_implements(type##_array_ce TSRMLS_CC, 2,                      \
          zend_ce_arrayaccess, zend_ce_iterator);

ここで最後に注意しなければならないことがあります。一般的には、 ``Iterator`` よりも ``IteratorAggregate`` を実装する方が良いでしょう。なぜなら ``IteratorAggregate`` だとイテレーターの状態をメインのオブジェクトから分離できるからです。このデザイン設計が良いのは明らかで、ネストしてもそれぞれを独立したイテレーションとして振る舞う事も可能となります。それでもここで ``Iterator`` を選んでいるのは ``IteratorAggregate`` の方が(独立したオブジェクトとやり取りをおこなう別々のクラスが必要となってくるために)実装上のオーバーヘッドが大きいからです。
