シリアライゼーション
====================

このセクションでは、PHPのシリアル化のフォーマットについてとオブジェクトのデータをシリアル化するための仕組みについて説明します。いつも通り、型付き配列の実装を例に使っていきます。

PHPのシリアライズ形式
---------------------

もしかすると既に ``serialize()`` の出力がどのようなものかおおよそご存知かもしれません。出力結果では何らかの( ``s`` や ``i`` といった)型の指定子が含まれており、それにコロンが続き、その後にデータとセミコロンが続きます。そのため、基本型のシリアライズ形式は次のようになります。::

  NULL:         N;
  true:         b:1;
  false:        b:0;
  42:           i:42;  

  42.3789:      d:42.378900000000002;
                  ^-- Precision controlled by serialize_precision ini setting (default 17)  

  "foobar":     s:6:"foobar";
                  ^-- strlen("foobar")  

  resource:     i:0;
                ^-- Resources can't really be serialized, so they just get the value int(0)

配列の場合は波括弧の中にキーと値のペアのリストとなります。::

  [10, 11, 12]:     a:3:{i:0;i:10;i:1;i:11;i:2;i:12;}
                      ^-- count([10, 11, 12])
  
                                                   v-- key   v-- value
  ["foo" => 4, "bar" => 2]:     a:2:{s:3:"foo";i:4;s:3:"bar";i:2;}
                                     ^-- key   ^-- value

オブジェクトには2つのシリアライズの仕組みがあります。1つは、単にオブジェクトのプロパティを配列と同じ方法でシリアライズするという方法です。この仕組みは型の指定子に ``O`` を使います。

下記のクラスで考えてみましょう。

.. code-block:: php

 <?php
  
  class Test {
      public $public = 1;
      protected $protected = 2;
      private $private = 3;
  }

シリアライズされると次のようになります。::

    v-- strlen("Test")           v-- property          v-- value
  O:4:"Test":3:{s:6:"public";i:1;s:12:"\0*\0protected";i:2;s:13:"\0Test\0private";i:3;}
                ^-- property ^-- value                     ^-- property           ^-- value

上記のシリアライズの中の ``\0`` という文字列はヌルバイトのことです。見ての通り、privateやprotectedのメンバー変数はそれ固有の名前でシリアル化されます。privateなメンバー変数は ``\0ClassName\0`` 接頭辞が付けられ、protectedなメンバー変数には ``\0*\0`` が付けられます。それらの名前は名前修飾によるものですが、これについては後のセクションで扱います。

2つ目のシリアライズの仕組みでは、独自のシリアライズのフォーマットを使うことが可能です。この仕組みでは実際のシリアライズの処理を ``Serializable`` インターフェイスの``serialize`` メソッドに委譲して、 ``C`` という型指定子を使います。例として次のクラスを考えてみましょう。

.. code-block:: php

  <?php
  
  class Test2 implements Serializable {
      public function serialize() {
          return "foobar";
      }
      public function unserialize($str) {
          // ...
      }
  }

これは次のようにシリアル化されます。::

  C:5:"Test2":6:{foobar}
              ^-- strlen("foobar")

この場合、PHPは単に ``Serializable::serialize()`` の結果を波括弧の中に入れ込みます。

PHPのシリアル化のフォーマットのもうひとつの特徴は、適切に参照を保持するということです。::

  $a = ["foo"];
  $a[1] =& $a[0];
  
  a:2:{i:0;s:3:"foo";i:1;R:2;}

ここで重要な部分は ``R:2;`` という要素です。これは"2番目の値への参照"という意味です。この2番目の値とは何を指すのでしょうか。配列全体が1番目の値となり、最初のインデックス( ``s:3:"foo"`` )が2番目の値で、これが参照されているのです。

PHPのオブジェクトは参照渡しのような振る舞いをおこなうので、 ``serialize`` も 同じオブジェクトが2回登場するとアンシリアライズの際に必ず全く同じオブジェクトとなるようにします。::

  $o = new stdClass;
  $o->foo = $o;
  
  O:8:"stdClass":1:{s:3:"foo";r:1;}

見ての通り、参照の場合と同じ方法でシリアライズしますが、``R`` の代わりに ``r`` を使います。


内部クラスのシリアライズ
------------------------

PHP内部のクラスは普通のプロパティ上にデータを保持していないので、PHPのデフォルトのシリアライズの仕組みは上手く動作しないでしょう。例えば、 ``ArrayBuffer`` をシリアライズしようとすると次のような結果となります。::

  O:11:"ArrayBuffer":0:{}

そのため、シリアライズのための独自のハンドラーを実装しなければなりません。上述のように、オブジェクトは( ``O`` と ``C`` での)2通りの方法でシリアライズが可能です。ここでは両方の仕組みの使い方を実演してみせたいと思います。まずは ``Serializable`` インターフェイスを使う ``C`` のシリアライズ形式からです。この方法の場合、 ``serialize`` で提供されている基本をベースとした独自のシリアライズ形式を作成しなければなりません。そのためには、2つのヘッダーをincludeする必要があります。

.. code-block:: c

  #include "ext/standard/php_var.h"
  #include "ext/standard/php_smart_str.h"

``php_var.h`` ヘッダーによっていくつかのシリアライズ関数がエクスポートされ、 ``php_smart_str.h`` ヘッダーにはPHPの ``smart_str`` APIが含まれています。このAPIは動的にリサイズされる文字列の構造体を提供し、これによって自身でのメモリ割り当てに煩わされることなく文字列を簡単に作成できます。

では ``ArrayBuffer`` の ``serialize`` メソッドがどのようになるか見ていきましょう。

.. code-block:: c

  PHP_METHOD(ArrayBuffer, serialize)
  {
      buffer_object *intern;
      smart_str buf = {0};
      php_serialize_data_t var_hash;
      zval zv, *zv_ptr = &zv;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);
      if (!intern->buffer) {
          return;
      }  

      PHP_VAR_SERIALIZE_INIT(var_hash);  

      INIT_PZVAL(zv_ptr);  

      /* バッファを文字列としてシリアライズする */
      ZVAL_STRINGL(zv_ptr, (char *) intern->buffer, (int) intern->length, 0);
      php_var_serialize(&buf, &zv_ptr, &var_hash TSRMLS_CC);  

      /* プロパティを配列としてシリアライズする */
      Z_ARRVAL_P(zv_ptr) = zend_std_get_properties(getThis() TSRMLS_CC);
      Z_TYPE_P(zv_ptr) = IS_ARRAY;
      php_var_serialize(&buf, &zv_ptr, &var_hash TSRMLS_CC);  

      PHP_VAR_SERIALIZE_DESTROY(var_hash);  

      if (buf.c) {
          RETURN_STRINGL(buf.c, buf.len, 0);
      }
  }

よくあるイディオムは別とすると、このメソッドは興味深い要素がいくつかあります。まずは、``PHP_VAR_SERIALIZE_INIT`` で初期化され、 ``PHP_VAR_SERIALIZE_DESTROY`` で破棄されている ````php_serialize_data_t var_hash```` という変数を宣言している部分です。実際にはこの変数は ``HashTable*`` の型で、 ``R`` / ``r`` の参照を保持する仕組みのために、シリアライズされた値を覚えておくために使用されます。

さらに、 ``smart_str buf = {0}`` としてsmart_strを生成します。 ``= {0}`` は構造体の全てのメンバーを0で初期化します。この構造体は下記の通りです。

.. code-block:: c

  typedef struct {
      char *c;
      size_t len;
      size_t a;
  } smart_str;

``c`` は文字列のバッファ、 ``len`` は現在使用されている長さ、 ``a`` は現在のメモリ割り当てのサイズです(smart stringなので ``len`` と一致する必要はありません)。

シリアライズそれ自体はダミーのzval( ``zv_ptr`` )を使って行われます。まず値をそのzvalに書き込み、それから ``php_var_serialize`` を実行します。まずバッファを(文字列として)シリアライズして、次にプロパティを(配列として)シリアライズします。

``unserialize`` メソッドはもう少し複雑です。

.. code-block:: c

  PHP_METHOD(ArrayBuffer, unserialize)
  {
      buffer_object *intern;
      char *str;
      int str_len;
      php_unserialize_data_t var_hash;
      const unsigned char *p, *max;
      zval zv, *zv_ptr = &zv;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &str, &str_len) == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);  

      if (intern->buffer) {
          zend_throw_exception(
              NULL, "Cannot call unserialize() on an already constructed object", 0 TSRMLS_CC
          );
          return;
      }  

      PHP_VAR_UNSERIALIZE_INIT(var_hash);  

      p = (unsigned char *) str;
      max = (unsigned char *) str + str_len;  

      INIT_ZVAL(zv);
      if (!php_var_unserialize(&zv_ptr, &p, max, &var_hash TSRMLS_CC)
          || Z_TYPE_P(zv_ptr) != IS_STRING || Z_STRLEN_P(zv_ptr) == 0) {
          zend_throw_exception(NULL, "Could not unserialize buffer", 0 TSRMLS_CC);
          goto exit;
      }  

      intern->buffer = Z_STRVAL_P(zv_ptr);
      intern->length = Z_STRLEN_P(zv_ptr);  

      INIT_ZVAL(zv);
      if (!php_var_unserialize(&zv_ptr, &p, max, &var_hash TSRMLS_CC)
          || Z_TYPE_P(zv_ptr) != IS_ARRAY) {
          zend_throw_exception(NULL, "Could not unserialize properties", 0 TSRMLS_CC);
          goto exit;
      }  

      if (zend_hash_num_elements(Z_ARRVAL_P(zv_ptr)) != 0) {
          zend_hash_copy(
              zend_std_get_properties(getThis() TSRMLS_CC), Z_ARRVAL_P(zv_ptr),
              (copy_ctor_func_t) zval_add_ref, NULL, sizeof(zval *)
          );
      }  

  exit:
      zval_dtor(zv_ptr);
      PHP_VAR_UNSERIALIZE_DESTROY(var_hash);
  }

``unserialize`` メソッドでも ``var_hash`` 変数を宣言していますが、今回は ``php_unserialize_data_t`` の型で、 ``PHP_VAR_UNSERIALIZE_INIT`` で初期化されて ``PHP_VAR_UNSERIALIZE_DESTROY`` で破棄されます。これはシリアライズ関数と同様に、 ``R`` / ``r`` のために変数を保持するという全く同じ役割を持っています。

``php_var_unserialize`` 関数を使うためには、シリアライズされた文字列への2つのポインターが必要です。1つ目は、 ``p`` で、シリアライズされた文字列の現在のポジションを表しています。2つ目は、 ``max`` で、これは文字列の末尾を指しています。 ``p`` のポジションは ``php_var_unserialize`` に参照渡しされて、アンシリアライズするために次の値のポジションへと変更されていきます。

アンシリアライズ処理では、最初にバッファを、次にプロパティを読み取ります。コードの大半は様々なエラーハンドリングとなっています。PHPにはシリアライズのクラッシュ(やセキュリティ)に関連するような長い歴史がありますので、全てのデータが正当であるということを確実にしなければなりません。また、 ``unserialize`` は他のメソッドとは違って特殊な意味を持っているのにも関わらず、通常のメソッドのように呼び出せるということを忘れてはいけません。そのような呼び出しを防ぐために、もし ``intern->buffer`` が既にセットされている場合には処理を中断しています。

では次に、2つ目のシリアライズの仕組みを、バッファビューで使用して見ていきましょう。 ``O`` でのシリアライズを実装するためには、独自の ``get_properties`` ハンドラー(シリアライズするためにプロパティを返します)と ``__wakeup`` メソッド(シリアライズされたプロパティの状態を復元します)が必要となってきます。

``get_properties`` ハンドラーはオブジェクトのプロパティをハッシュテーブルとして取得するためのものです。ZendEngineは様々なケースでこの処理を実行していて、そのうちの1つが ``O`` でのシリアライズです。ビューのバッファ、オフセット、長さを取得するためにこのハンドラーを使用し、他のプロパティ同様にシリアライズされます。

.. code-block:: c

  static HashTable *array_buffer_view_get_properties(zval *obj TSRMLS_DC)
  {
      buffer_view_object *intern = zend_object_store_get_object(obj TSRMLS_CC);
      HashTable *ht = zend_std_get_properties(obj TSRMLS_CC);
      zval *zv;  

      if (!intern->buffer_zval) {
          return ht;
      }  

      Z_ADDREF_P(intern->buffer_zval);
      zend_hash_update(ht, "buffer", sizeof("buffer"), &intern->buffer_zval, sizeof(zval *), NULL);  

      MAKE_STD_ZVAL(zv);
      ZVAL_LONG(zv, intern->offset);
      zend_hash_update(ht, "offset", sizeof("offset"), &zv, sizeof(zval *), NULL);  

      MAKE_STD_ZVAL(zv);
      ZVAL_LONG(zv, intern->length);
      zend_hash_update(ht, "length", sizeof("length"), &zv, sizeof(zval *), NULL);  

      return ht;
  }

このような定義済みのプロパティはデバッグ用の出力でも表示されるようになるので、この場合良い方法でしょう。また、これらのプロパティは通常のプロパティとしてアクセスできますが、それはこのハンドラーが実行されてからのみということに注意してください。例えば、オブジェクトをシリアライズした後なら ``$view->buffer`` とアクセスできるでしょう。これによる副作用は(他のシリアライズの方法を利用する以外)避けられません。

アンシリアライズした後にオブジェクトの状態を復元するために、 ``__wakeup`` のマジックメソッドを実装します。このメソッドはアンシリアライズされた直後に呼び出され、オブジェクトのプロパティを読みとる事ができ、オブジェクトの内部的な状態を再構築することが出来ます。

.. code-block:: c

  PHP_FUNCTION(array_buffer_view_wakeup)
  {
      buffer_view_object *intern;
      HashTable *props;
      zval **buffer_zv, **offset_zv, **length_zv;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      intern = zend_object_store_get_object(getThis() TSRMLS_CC);  

      if (intern->buffer_zval) {
          zend_throw_exception(
              NULL, "Cannot call __wakeup() on an already constructed object", 0 TSRMLS_CC
          );
          return;
      }  

      props = zend_std_get_properties(getThis() TSRMLS_CC);  

      if (zend_hash_find(props, "buffer", sizeof("buffer"), (void **) &buffer_zv) == SUCCESS
       && zend_hash_find(props, "offset", sizeof("offset"), (void **) &offset_zv) == SUCCESS
       && zend_hash_find(props, "length", sizeof("length"), (void **) &length_zv) == SUCCESS
       && Z_TYPE_PP(buffer_zv) == IS_OBJECT
       && Z_TYPE_PP(offset_zv) == IS_LONG && Z_LVAL_PP(offset_zv) >= 0
       && Z_TYPE_PP(length_zv) == IS_LONG && Z_LVAL_PP(length_zv) > 0
       && instanceof_function(Z_OBJCE_PP(buffer_zv), array_buffer_ce TSRMLS_CC)
      ) {
          buffer_object *buffer_intern = zend_object_store_get_object(*buffer_zv TSRMLS_CC);
          size_t offset = Z_LVAL_PP(offset_zv), length = Z_LVAL_PP(length_zv);
          size_t bytes_per_element = buffer_view_get_bytes_per_element(intern);
          size_t max_length = (buffer_intern->length - offset) / bytes_per_element;  

          if (offset < buffer_intern->length && length <= max_length) {
              Z_ADDREF_PP(buffer_zv);
              intern->buffer_zval = *buffer_zv;  

              intern->offset = offset;
              intern->length = length;  

              intern->buf.as_int8 = buffer_intern->buffer;
              intern->buf.as_int8 += offset;  

              return;
          }
      }  

      zend_throw_exception(
          NULL, "Invalid serialization data", 0 TSRMLS_CC
      );
  }


程度の差はありますが、このメソッドは(シリアライズ処理にはよくあることですが)お決まりのエラーチェックがされています。実質このメソッドがおこなっていることは、それは3つの定義済みのプロパティを ``zend_hash_find`` で取得して、それらが妥当かどうか検証し、その後オブジェクトを初期化するということです。

シリアライズの禁止
------------------

オブジェクトによっては理論的にシリアライズできないものもあります。その場合は、特別なシリアライズのハンドラーを割り当てることでシリアライズを禁止することができます。::

  ce->serialize = zend_class_serialize_deny;
  ce->unserialize = zend_class_unserialize_deny;

``serialize`` と ``unserialize`` のクラスハンドラーは ``Serializable`` インターフェイス、つまり ``C`` シリアライズ を実装するために使用されます。そのため、このように割り当てることでシリアライズや ``C`` のアンシリアライズを禁止できますが、 ``O`` アンシリアライズはまだ許可されています。これも禁止するためには、 ``__wakeup`` から単にエラーをスローするだけです。

.. code-block:: c

  PHP_METHOD(SomeClass, __wakeup)
  {
      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }
  
      zend_throw_exception(NULL, "Unserialization of SomeClass is not allowed", 0 TSRMLS_CC);
  }

これでこれまで見てきた型付き配列を終わり、次のテーマとして定義済みインターフェイスを取り上げていきたいと思います。
