型付き配列の実装
================

前の2つのセクションではクラスシステムについて少々抽象的な説明してきました。一方このセクションでは、"実際の"クラス、つまり型付き配列の実装をご案内したいと思います。この例を使うのには2つの理由があります。1つ目の理由は、型付き配列は内部で実装することは非常に有意義だからです。型付き配列をPHPのユーザーランドで実装する方がむしろ難しく、PHP内部で実装することでメモリの節約と処理の高速化につながります。2つ目に、型付き配列はPHPのオブジェクトやクラスのハンドラーを引き立たせるのに良い例だからです。型付き配列には、例えばオフセットのアクセス、要素のカウント、イテレーション、シリアライズ、デバッグ情報が必要になってきます。

配列のバッファとビュー
-----------------------

このセクションで実装するのはJavaScriptのArrayBufferシステムの縮小版です。 ``ArrayBuffer`` は単に固定サイズのメモリの塊です。 ``ArrayBuffer`` はそれ自体、読み書きは出来ず、単なるオブジェクトを表現しているメモリーです。

データをバッファから読んだり、書き込んだりするためには、バッファのビューを作成しなければなりません。例えば、符号付き32bitの整数の配列としてバッファを解釈するためには、 ``Int32Array`` ビューを作成します。代わりに、符号なし8ビットの数字の配列として解釈するには ``UInt8Array`` が使用できます。同じデータに対して、いくつかのビューを持つことは可能なので、データをint32とuint8の両方として解釈可能です。

簡単な使用法の例です。

.. code-block:: php

  <?php  

  // 256 byteのバッファを割り当てる
  $buffer = new ArrayBuffer(256);  

  // そのバッファ上に 256 / 8 = 32 要素で int32 のビューを生成する
  $int32 = new Int32Array($buffer);  

  // 同じバッファを使って 256/ 1 = 256 要素でuint8 ビューを生成する 
  $uint8 = new UInt8Array($buffer);  

  //  0 から 255 のuint8のビューを埋める
  for ($i = 0; $i < 256; ++$i) {
      $uint8[$i] = $i;
  }  

  // signed 32 bitの整数として解釈しながら埋められたバッファを読む
  for ($i = 0; $i < 32; ++$i) {
      echo $int32[$i], "\n";
  }

この種のバッファとビューのシステムは多くの目的にとって便利ですので、ここでもこのシステムを実装することにします。実装が長くなり過ぎないように、JS APIの全てではなく、その重要な部分のみを実装します。さらに、オーバーフローの振る舞いやエンディアンのような、実装の詳細についてそれほど多くの時間をかけて考察することはしません。それらは"実際の"実装について重要な考察ではありますが、ここでの目的にとってはそれほど関連はないので、単に"デフォルトによる"振る舞い(つまり最小のコードで)を使い続けるようにします。

ArrayBuffer
------------

``ArrayBuffer`` はとてもシンプルなオブジェクトで、必要なのは単にメモリ割り当てとバッファと長さを保持するだけです。それ故、内部の構造体は下記のようになっています。

.. code-block:: c

  typedef struct _buffer_object {
      zend_object std;  

      void *buffer;
      size_t length;
  } buffer_object;  

  /* クラスエントリーとハンドラーの宣言もおこなっておく */
  zend_class_entry *array_buffer_ce;
  zend_object_handlers array_buffer_handlers;

生成と解放のハンドラーも同様にシンプルで、前のセクションのものとほとんど同じです。

.. code-block:: c

  static void array_buffer_free_object_storage(buffer_object *intern TSRMLS_DC)
  {
      zend_object_std_dtor(&intern->std TSRMLS_CC);  

      if (intern->buffer) {
          efree(intern->buffer);
      }  

      efree(intern);
  }  

  zend_object_value array_buffer_create_object(zend_class_entry *class_type TSRMLS_DC)
  {
      zend_object_value retval;  

      buffer_object *intern = emalloc(sizeof(buffer_object));
      memset(intern, 0, sizeof(buffer_object));  

      zend_object_std_init(&intern->std, class_type TSRMLS_CC);
      object_properties_init(&intern->std, class_type);  

      retval.handle = zend_objects_store_put(intern,
          (zend_objects_store_dtor_t) zend_objects_destroy_object,
          (zend_objects_free_object_storage_t) array_buffer_free_object_storage,
          NULL TSRMLS_CC
      );
      retval.handlers = &array_buffer_handlers;  

      return retval;
  }

``create_object`` のハンドラーはバッファまだメモリ割り当てはせず、これはコンストラクターで行われます(メモリ割り当てがコンストラクターのパラメーターであるバッファの長さに依存するからです)。

.. code-block:: c

  PHP_METHOD(ArrayBuffer, __construct)
  {
      buffer_object *intern;
      long length;
      zend_error_handling error_handling;  

      zend_replace_error_handling(EH_THROW, NULL, &error_handling TSRMLS_CC);
      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "l", &length) == FAILURE) {
          zend_restore_error_handling(&error_handling TSRMLS_CC);
          return;
      }
      zend_restore_error_handling(&error_handling TSRMLS_CC);  

      if (length <= 0) {
          zend_throw_exception(NULL, "Buffer length must be positive", 0 TSRMLS_CC);
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);  

      intern->buffer = emalloc(length);
      intern->length = length;  

      memset(intern->buffer, 0, length);
  }

ここではオブジェクト指向のコードを書いているので、もうエラーを投げるようにするのではなく、むしろ例外を投げるようにします。例外を投げるのは ``zend_throw_exception`` を使い、これは例外クラスエントリ、例外メッセージ、エラーコードを受け取ります。もし例外クラスエントリに ``NULL`` を渡した場合は、デフォルトの、つまり ``Exception`` クラスとなります。

``__construct`` メソッドでは特に、エラーが起きた場合には不完全な状態のオブジェクトとなってしまわないように、例外を投げることが重要です。そのため、上記のコードではパラメーターの解析中にエラーハンドリングモードを変更しています。通常、 ``zend_parse_parameters`` は不正なパラメーターの場合には単に警告を投げるだけで、この場合これでは十分ではありません。エラーモードに ``EH_THROW`` を設定することで、警告は自動的に例外に変換されます。

エラーハンドリングモードは ``zend_replace_error_handling`` を使うことで変更できます。これは ``EH_NORMAL`` (デフォルトのエラーレポート)、 ``EH_SUPPRESS`` (全てのエラーを抑制)、あるいは ``EH_THROW`` (エラーを例外として投げる) のうちどれかを最初の引数として受け取ります。2番目の引数には、 ``EH_THROW`` モードの際の例外クラスエントリを指定できます。もし ``NULL`` の場合には、デフォルトの ``Exception`` クラスが使用されます。最後の引数には ``zend_error_handling`` 構造体のポインターが渡され、変更前のエラーハンドリングモードが保存されます。後で変更前のモードに戻すためには、この構造体を ``zend_restore_error_handling`` に渡します。

生成のためのハンドラーとは別に、クローンのハンドラーも用意しなければなりません。 ``ArrayBuffer`` の場合、クローンは割り当て済みのバッファのコピーするのと同じくらい簡単です。

.. code-block:: c

  static zend_object_value array_buffer_clone(zval *object TSRMLS_DC)
  {
      buffer_object *old_object = zend_object_store_get_object(object TSRMLS_CC);
      zend_object_value new_object_val = array_buffer_create_object(Z_OBJCE_P(object) TSRMLS_CC);
      buffer_object *new_object = zend_object_store_get_object_by_handle(
          new_object_val.handle TSRMLS_CC
      );  

      zend_objects_clone_members(
          &new_object->std, new_object_val,
          &old_object->std, Z_OBJ_HANDLE_P(object) TSRMLS_CC
      );  

      new_object->buffer = old_object->buffer;
      new_object->length = old_object->length;  

      if (old_object->buffer) {
          new_object->buffer = emalloc(old_object->length);
          memcpy(new_object->buffer, old_object->buffer, old_object->length);
      }  

      memcpy(new_object->buffer, old_object->buffer, old_object->length);  

      return new_object_val;
  }

そして最後に ``MINIT`` に全てをそろえます。

.. code-block:: c

  ZEND_BEGIN_ARG_INFO_EX(arginfo_buffer_ctor, 0, 0, 1)
      ZEND_ARG_INFO(0, length)
  ZEND_END_ARG_INFO()  

  const zend_function_entry array_buffer_functions[] = {
      PHP_ME(ArrayBuffer, __construct, arginfo_buffer_ctor, ZEND_ACC_PUBLIC)
      PHP_FE_END
  };  

  MINIT_FUNCTION(buffer)
  {
      zend_class_entry tmp_ce;  

      INIT_CLASS_ENTRY(tmp_ce, "ArrayBuffer", array_buffer_functions);
      array_buffer_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);
      array_buffer_ce->create_object = array_buffer_create_object;  

      memcpy(&array_buffer_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
      array_buffer_handlers.clone_obj = array_buffer_clone;  

      return SUCCESS;
  }


バッファビュー
--------------

バッファビューはもう少し大変となるでしょう。1つの同じ実装を共有する8つの異なるビュークラス、すなわち ``Int8Array`` 、 ``UInt8Array`` 、 ``Int16Array`` 、 ``UInt16Array`` 、 ``Int32Array`` 、 ``UInt32Array`` 、 ``FloatArray`` 、 ``DoubleArray`` を実装していきます。クラス登録のコードは次の通りです。

.. code-block:: c

  zend_class_entry *int8_array_ce;
  zend_class_entry *uint8_array_ce;
  zend_class_entry *int16_array_ce;
  zend_class_entry *uint16_array_ce;
  zend_class_entry *int32_array_ce;
  zend_class_entry *uint32_array_ce;
  zend_class_entry *float_array_ce;
  zend_class_entry *double_array_ce;  

  zend_object_handlers array_buffer_view_handlers;  

  /* ... ここには沢山のコードが記述されることになる ... */  

  PHP_MINIT_FUNCTION(buffer)
  {
      zend_class_entry tmp_ce;  

      /* ... ここでArrayBufferの処理を記述する ... */  

  #define DEFINE_ARRAY_BUFFER_VIEW_CLASS(class_name, type)                      \
      INIT_CLASS_ENTRY(tmp_ce, #class_name, array_buffer_view_functions);       \
      type##_array_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);        \
      type##_array_ce->create_object = array_buffer_view_create_object;         \
      zend_class_implements(type##_array_ce TSRMLS_CC, 1, zend_ce_arrayaccess);  

      DEFINE_ARRAY_BUFFER_VIEW_CLASS(Int8Array,   int8);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(UInt8Array,  uint8);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(Int16Array,  int16);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(Uint16Array, uint16);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(Int32Array,  int32);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(UInt32Array, uint32);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(FloatArray,  float);
      DEFINE_ARRAY_BUFFER_VIEW_CLASS(DoubleArray, double);  

  #undef DEFINE_ARRAY_BUFFER_VIEW_CLASS  

      memcpy(&array_buffer_view_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
      array_buffer_view_handlers.clone_obj = array_buffer_view_clone;  

      return SUCCESS;
  }


何度も同じコードを書くのを避けるために、一時的なマクロを使用します。そのマクロはクラスエントリを初期化(常に同じ関数で行われます)、クラスの登録、生成ハンドラーの割り当て(これも全てのクラスにとって同じ処理です)、 ``ArrayAccess`` のインターフェイスの実装をおこないます。マクロは ``#`` (文字列化演算子)と ``##`` (トークン連結演算子)の演算子を使っています。[注: これらの演算子はどこかで説明しなければなりませんが、それらの演算子の意味が分からない場合はとりあえずググって下さい]

``array_buffer_view_functions`` 関数は次のように宣言されます。

.. code-block:: c

  ZEND_BEGIN_ARG_INFO_EX(arginfo_buffer_view_ctor, 0, 0, 1)
      ZEND_ARG_INFO(0, buffer)
  ZEND_END_ARG_INFO()  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_buffer_view_offset, 0, 0, 1)
      ZEND_ARG_INFO(0, offset)
  ZEND_END_ARG_INFO()  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_buffer_view_offset_set, 0, 0, 2)
      ZEND_ARG_INFO(0, offset)
      ZEND_ARG_INFO(0, value)
  ZEND_END_ARG_INFO()  

  const zend_function_entry array_buffer_view_functions[] = {
      PHP_ME_MAPPING(__construct, array_buffer_view_ctor, arginfo_buffer_view_ctor, ZEND_ACC_PUBLIC)  

      /* 配列のアクセス */
      PHP_ME_MAPPING(
          offsetGet, array_buffer_view_offset_get, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC
      )
      PHP_ME_MAPPING(
          offsetSet, array_buffer_view_offset_set, arginfo_buffer_view_offset_set, ZEND_ACC_PUBLIC
      )
      PHP_ME_MAPPING(
          offsetExists, array_buffer_view_offset_exists, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC
      )
      PHP_ME_MAPPING(
          offsetUnset, array_buffer_view_offset_unset, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC
      )  

      PHP_FE_END
  };

ここで新しいのは ``PHP_ME`` の代わりに、 ``PHP_ME_MAPPING`` が使われているということです。違いは、 ``PHP_ME`` は ``PHP_METHOD`` にマッピングするのに対して、 ``PHP_ME_MAPPING`` は ``PHP_FUNCTION`` にマッピングします。下記の例をご覧ください。

.. code-block:: c

  PHP_ME(ArrayBufferView, offsetGet, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC)
  /* PHP_MEはPHP_METHODにマッピングする */
  PHP_METHOD(ArrayBufferView, offsetGet) { ... }  

  PHP_ME_MAPPING(
      offsetGet, array_buffer_view_offset_get, arginfo_buffer_view_offset, ZEND_ACC_PUBLIC
  )
  /* PHP_ME_MAPPINGはPHP_FUNCTIONにマッピングする */
  PHP_FUNCTION(array_buffer_view_offset_get) { ... }

ここで ``PHP_FUNCTION`` と ``PHP_METHOD`` はPHPの関数やメソッドに何ら関係のないということを理解しておかなければなりません。それらは関数名や一連のパラメーターで関数を定義する単なるマクロです。そのため、"関数"をメソッドとして登録することが出来ます(同じ1つの名前で定義することも出来ますが、異なる名前で登録もできます)。これはオブジェクト指向なインターフェイスと手続き型のAPIの両方に対応する際には非常に役に立ちます。

この場合では、 ``ArrayBufferView`` という実際のクラスはなく、むしろいくつかのクラスによって共有される一連のクラスがあるということを示すために ``PHP_ME_MAPPING`` を使うようにしました。

実装部分に話を戻すと、バッファビューのための内部の構造体が何を保持する必要があるかを考えなければなりません。まずは、異なるビュークラスを区別するための方法、つまり何らかの型のタグが必要です。2つ目に、動作するバッファのzvalを保持する必要があります。そして3つ目に、それぞれ異なる型としてバッファにアクセスするために使用できるメンバーを保持しなければなりません。

追加で、ここでの実装ではオフセットやビューの長さを保持するようにします。それらはバッファの全体使わないビューを作成するために使用されます。例えば、 ``new Int32Array($buffer, 18, 24)`` はバッファの18 bytesから開始して、全体で24要素を持ったビューを作成します。

結果として構造体は次のようになるでしょう。

.. code-block:: c

  typedef enum _buffer_view_type {
      buffer_view_int8,
      buffer_view_uint8,
      buffer_view_int16,
      buffer_view_uint16,
      buffer_view_int32,
      buffer_view_uint32,
      buffer_view_float,
      buffer_view_double
  } buffer_view_type;  

  typedef struct _buffer_view_object {
      zend_object std;  

      zval *buffer_zval;  

      union {
          int8_t   *as_int8;
          uint8_t  *as_uint8;
          int16_t  *as_int16;
          uint16_t *as_uint16;
          int32_t  *as_int32;
          uint32_t *as_uint32;
          float    *as_float;
          double   *as_double;
      } buf;  

      size_t offset;
      size_t length;  

      buffer_view_type type;
  } buffer_view_object;


上で使用されている固定長整数型( ``int8_t`` , ..)は ``stdint.h`` のヘッダーの一部です。残念なことに、このヘッダーはWindowsでは常に利用可能とは限らないので、この場合ヘッダーの取り換え(PHPでは元々提供されています)を行わなければなりません。

.. code-block:: c

  #if defined(PHP_WIN32)
  # include "win32/php_stdint.h"
  #elif defined(HAVE_STDINT_H)
  # include <stdint.h>
  #endif

上述のデータ構造の生成と解放のハンドラーに関してはここでもかなり単純です。

.. code-block:: c

  static void array_buffer_view_free_object_storage(buffer_view_object *intern TSRMLS_DC)
  {
      zend_object_std_dtor(&intern->std TSRMLS_CC);  

      if (intern->buffer_zval) {
          zval_ptr_dtor(&intern->buffer_zval);
      }  

      efree(intern);
  }  

  zend_object_value array_buffer_view_create_object(zend_class_entry *class_type TSRMLS_DC)
  {
      zend_object_value retval;  

      buffer_view_object *intern = emalloc(sizeof(buffer_view_object));
      memset(intern, 0, sizeof(buffer_view_object));  

      zend_object_std_init(&intern->std, class_type TSRMLS_CC);
      object_properties_init(&intern->std, class_type);  

      {
          zend_class_entry *base_class_type = class_type;  

          while (base_class_type->parent) {
              base_class_type = base_class_type->parent;
          }  

          if (base_class_type == int8_array_ce) {
              intern->type = buffer_view_int8;
          } else if (base_class_type == uint8_array_ce) {
              intern->type = buffer_view_uint8;
          } else if (base_class_type == int16_array_ce) {
              intern->type = buffer_view_uint16;
          } else if (base_class_type == int32_array_ce) {
              intern->type = buffer_view_int32;
          } else if (base_class_type == uint32_array_ce) {
              intern->type = buffer_view_uint32;
          } else if (base_class_type == float_array_ce) {
              intern->type = buffer_view_float;
          } else if (base_class_type == double_array_ce) {
              intern->type = buffer_view_double;
          } else {
              /* ここに来るべきではない */
              zend_error(E_ERROR, "Buffer view does not have a valid base class");
          }
      }  

      retval.handle = zend_objects_store_put(intern,
          (zend_objects_store_dtor_t) zend_objects_destroy_object,
          (zend_objects_free_object_storage_t) array_buffer_view_free_object_storage,
          NULL TSRMLS_CC
      );
      retval.handlers = &array_buffer_view_handlers;  

      return retval;
  }

``create_object`` ハンドラーはまずインスタンス化された基底クラスを探し、それからどのバッファビューが対応しているのかを導き出します。クラスのうちいずれかが拡張されている場合に、全てが問題なく動作するように ``parent`` チェインを見ていく必要があります。生成のためのハンドラーはとりわけ多くのことはしておらず、メインの処理はコンストラクターで行われています。

.. code-block:: c

  PHP_FUNCTION(array_buffer_view_ctor)
  {
      zval *buffer_zval;
      long offset = 0, length = 0;
      buffer_view_object *view_intern;
      buffer_object *buffer_intern;
      zend_error_handling error_handling;  

      zend_replace_error_handling(EH_THROW, NULL, &error_handling TSRMLS_CC);
      if (zend_parse_parameters(
              ZEND_NUM_ARGS() TSRMLS_CC, "O|ll", &buffer_zval, array_buffer_ce, &offset, &length
          ) == FAILURE
      ) {
          zend_restore_error_handling(&error_handling TSRMLS_CC);
          return;
      }
      zend_restore_error_handling(&error_handling TSRMLS_CC);  

      view_intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      buffer_intern = zend_object_store_get_object(buffer_zval TSRMLS_CC);  

      if (offset < 0) {
          zend_throw_exception(NULL, "Offset must be non-negative", 0 TSRMLS_CC);
          return;
      }
      if (offset >= buffer_intern->length) {
          zend_throw_exception(NULL, "Offset has to be smaller than the buffer length", 0 TSRMLS_CC);
          return;
      }
      if (length < 0) {
          zend_throw_exception(NULL, "Length must be positive or zero", 0 TSRMLS_CC);
          return;
      }  

      view_intern->offset = offset;
      view_intern->buffer_zval = buffer_zval;
      Z_ADDREF_P(buffer_zval);  

      {
          size_t bytes_per_element = buffer_view_get_bytes_per_element(view_intern);
          size_t max_length = (buffer_intern->length - offset) / bytes_per_element;  

          if (length == 0) {
              view_intern->length = max_length;
          } else if (length > max_length) {
              zend_throw_exception(NULL, "Length is larger than the buffer", 0 TSRMLS_CC);
              return;
          } else {
              view_intern->length = length;
          }
      }  

      view_intern->buf.as_int8 = buffer_intern->buffer;
      view_intern->buf.as_int8 += offset;
  }


このコードはほとんどエラーチェックと、その合間の所々で内部の構造体にいくつかの割り当てをおこなっています。またコードでは、 ``buffer_view_get_bytes_per_element`` というヘルパー関数を使用しており、これは名前の通りの事をおこなっています。

.. code-block:: c

  size_t buffer_view_get_bytes_per_element(buffer_view_object *intern)
  {
      switch (intern->type)
      {
          case buffer_view_int8:
          case buffer_view_uint8:
              return 1;
          case buffer_view_int16:
          case buffer_view_uint16:
              return 2;
          case buffer_view_int32:
          case buffer_view_uint32:
          case buffer_view_float:
              return 4;
          case buffer_view_double:
              return 8;
          default:
              /* ここに来るべきではない */
              zend_error_noreturn(E_ERROR, "Invalid buffer view type");
      }
  }

生成ロジックで唯一欠けているピースはクローンハンドラーで、これは全ての内部のメンバーをコピーし、バッファのzvalの参照を追加します。

.. code-block:: c

  static zend_object_value array_buffer_view_clone(zval *object TSRMLS_DC)
  {
      buffer_view_object *old_object = zend_object_store_get_object(object TSRMLS_CC);
      zend_object_value new_object_val = array_buffer_view_create_object(
          Z_OBJCE_P(object) TSRMLS_CC
      );
      buffer_view_object *new_object = zend_object_store_get_object_by_handle(
          new_object_val.handle TSRMLS_CC
      );  

      zend_objects_clone_members(
          &new_object->std, new_object_val,
          &old_object->std, Z_OBJ_HANDLE_P(object) TSRMLS_CC
      );  

      new_object->buffer_zval = old_object->buffer_zval;
      if (new_object->buffer_zval) {
          Z_ADDREF_P(new_object->buffer_zval);
      }  

      new_object->buf.as_int8 = old_object->buf.as_int8;
      new_object->offset = old_object->offset;
      new_object->length = old_object->length;
      new_object->type   = old_object->type;  

      return new_object_val;
  }

さて、形式的なやり方からは離れて、実際の機能である、あるオフセットにおいて値にアクセスの仕組みに取り掛かることが出来ます。そのために、ビューの型に応じたオフセットを取得したり設定したりするための2つのヘルパー関数が必要になります。これは基本的には結局、全ての異なる型をみていくswitch文やバッファ共有体のそれぞれのメンバーを使っていくということになります。

.. code-block:: c

  zval *buffer_view_offset_get(buffer_view_object *intern, size_t offset)
  {
      zval *retval;
      MAKE_STD_ZVAL(retval);  

      switch (intern->type) {
          case buffer_view_int8:
              ZVAL_LONG(retval, intern->buf.as_int8[offset]); break;
          case buffer_view_uint8:
              ZVAL_LONG(retval, intern->buf.as_uint8[offset]); break;
          case buffer_view_int16:
              ZVAL_LONG(retval, intern->buf.as_int16[offset]); break;
          case buffer_view_uint16:
              ZVAL_LONG(retval, intern->buf.as_uint16[offset]); break;
          case buffer_view_int32:
              ZVAL_LONG(retval, intern->buf.as_int32[offset]); break;
          case buffer_view_uint32: {
              uint32_t value = intern->buf.as_uint32[offset];
              if (value <= LONG_MAX) {
                  ZVAL_LONG(retval, value);
              } else {
                  ZVAL_DOUBLE(retval, value);
              }
              break;
          }
          case buffer_view_float:
              ZVAL_DOUBLE(retval, intern->buf.as_float[offset]); break;
          case buffer_view_double:
              ZVAL_DOUBLE(retval, intern->buf.as_double[offset]); break;
          default:
              /* ここに来るべきではない */
              zend_error_noreturn(E_ERROR, "Invalid buffer view type");
      }  

      return retval;
  }  

  void buffer_view_offset_set(buffer_view_object *intern, long offset, zval *value)
  {
      if (intern->type == buffer_view_float || intern->type == buffer_view_double) {
          Z_ADDREF_P(value);
          convert_to_double_ex(&value);  

          if (intern->type == buffer_view_float) {
              intern->buf.as_float[offset] = Z_DVAL_P(value);
          } else {
              intern->buf.as_double[offset] = Z_DVAL_P(value);
          }  

          zval_ptr_dtor(&value);
      } else {
          Z_ADDREF_P(value);
          convert_to_long_ex(&value);  

          switch (intern->type) {
              case buffer_view_int8:
                  intern->buf.as_int8[offset] = Z_LVAL_P(value); break;
              case buffer_view_uint8:
                  intern->buf.as_uint8[offset] = Z_LVAL_P(value); break;
              case buffer_view_int16:
                  intern->buf.as_int16[offset] = Z_LVAL_P(value); break;
              case buffer_view_uint16:
                  intern->buf.as_uint16[offset] = Z_LVAL_P(value); break;
              case buffer_view_int32:
                  intern->buf.as_int32[offset] = Z_LVAL_P(value); break;
              case buffer_view_uint32:
                  intern->buf.as_uint32[offset] = Z_LVAL_P(value); break;
              default:
                  /* ここに来るべきではない */
                  zend_error(E_ERROR, "Invalid buffer view type");
          }  

          zval_ptr_dtor(&value);
      }
  }

``ArrayAccess`` インターフェイスを実装は、単にちょっとした境界チェックと上のヘルパー関数へのディスパッチ(通常のメソッドでの決まり文句と同様にして)だけになります。 ``offsetGet`` メソッドの実装は次のようになるでしょう。

.. code-block:: c

  PHP_FUNCTION(array_buffer_view_offset_get)
  {
      buffer_view_object *intern;
      long offset;
      zval *retval;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "l", &offset) == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);  

      if (offset < 0 || offset >= intern->length) {
          zend_throw_exception(NULL, "Offset is outside the buffer range", 0 TSRMLS_CC);
          return;
      }  

      retval = buffer_view_offset_get(intern, offset);
      RETURN_ZVAL(retval, 1, 1);
  }


残りの ``offsetSet`` 、 ``offsetExists`` 、 ``offsetUnset`` メソッドはほとんど同じになるので、読者の方への課題として残しておきます。

実装のコードは約600行程度でJavaScriptでのバッファ/ビューシステムの極めて重要な部分を実装できます。

しかし現状の実装ではまだPHPに上手く統合できません。単に ``ArrayAccess`` の実装はしていますが、イテレーションやカウント等はできません。これらについては次のセクションで取り上げます。
