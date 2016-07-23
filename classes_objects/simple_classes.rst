シンプルなクラス
================

基本概念
----------

zvalは ``IS_OBJECT`` の型のタグと、下記のように定義されている共有体の ``zend_object_value`` 構造体を使ってオブジェクトを保持します。

.. code-block:: c

  typedef struct _zend_object_value {
      zend_object_handle handle;
      const zend_object_handlers *handlers;
  } zend_object_value;

この構造体の最初の部分の ``zend_object_handle`` は単にunsigned integer のtypedefです。これは単にオブジェクトを識別するための固有のIDで、オブジェクトストアから実際のオブジェクトのデータを取得するために使用されます。

2番目の部分はオブジェクトハンドラーの構造体へのポインターです。ハンドラーはオブジェクトの実際の振る舞いを定義しています。それらは、プロパティの取得やメソッドの呼び出しから、独自の比較処理まで、あるいは特別なガーベッジコレクションのセマンティクスといった全てを扱います。

呼び出される際、個々のハンドラーはオブジェクトのzvalを最初の引数として受け取り、続いて様々なハンドラー特有の情報を受け取ります。その後、ハンドラーはオブジェクトストアからオブジェクトのデータを取得するためにオブジェクトハンドルを使用することができ、それにおける処理を実行できます。

オブジェクトの値のための補足的な構造体で、クラスエントリ( ``zend_class_entry`` )というものがあります。クラスエントリは、クラスメソッドや静的プロパティだけでなく、様々なハンドラー、特にクラスからオブジェクトを生成するためのハンドラーをも含む、大量の情報を保持しています。

クラスの登録
--------------

関数とちょうど同じように、クラスはエクステンションの ``MINIT`` ハンドラーで登録されます。下記は ``Test`` という空のクラスを宣言するためのスニペットです。

.. code-block:: c

  zend_class_entry *test_ce;  

  const zend_function_entry test_functions[] = {
      PHP_FE_END
  };  

  PHP_MINIT_FUNCTION(test)
  {
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "Test", test_functions);  

      test_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);  

      return SUCCESS;
  }

1行目でグローバル変数 ``test_ce`` を宣言しており、これは ``Test`` クラスのクラスエントリを格納します。他のエクステンションからクラスが使えるようにするためには、それを「真」のグローバル変数とするのに加え、ヘッダーファイルを通して外部に宣言する必要があります。その次の3行はクラスメソッドのための配列を宣言しており、これはちょうど通常の関数でおこなうのと同じことです。

その後にメインの処理が続きます。まず初めに、一時的なクラスエントリの ``tmp_ce`` が定義され、 ``INIT_CLASS_ENTRY`` を使用して初期化されています。その後、``zend_register_internal_class`` を使用し、ZendEngineにクラスを登録します。この関数はまた、最終的なクラスエントリを返すので、それを上で宣言されているグローバル変数に格納します。

このクラスが適切に登録されたかを検証するためには、 ``php --rc Test`` を実行します。この実行で、次のような行に沿った結果が得られるはずです。

.. code-block:: c

  Class [ <internal:test> class Test ] {
    - Constants [0] {
    }
    - Static properties [0] {
    }
    - Static methods [0] {
    }
    - Properties [0] {
    }
    - Methods [0] {
    }
  }

期待していたように、得られた結果は完全な空クラスです。

メソッドの定義と宣言
----------------------

クラスに命を吹き込むために、メソッドを追加してみましょう。

.. code-block:: c

  PHP_METHOD(Test, helloWorld) /* {{{ */
  {
      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      RETURN_STRING("Hello World\n", 1);
  }
  /* }}} */  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_void, 0, 0, 0)
  ZEND_END_ARG_INFO()  

  const zend_function_entry test_functions[] = {
      PHP_ME(Test, helloWorld, arginfo_void, ZEND_ACC_PUBLIC)
      PHP_FE_END
  };

ご覧のように、メソッドの宣言は関数の宣言と非常によく似ています。 ``PHP_FUNCTION`` の代わりに ``PHP_METHOD`` を使い、それにクラス名とメソッド名の両方を渡します。 ``zend_function_entry`` の配列には、 ``PHP_FE`` の代わりに ``PHP_ME`` が使われています。ここでもクラス名とメソッド名、 arginfoの構造体や、それに加えて一連のフラグを受け取ります。

フラグのパラメーターを使うことで、 ``ZEND_ACC_PUBLIC`` 、 ``ZEND_ACC_PROTECTED`` 、 ``ZEND_ACC_PRIVATE`` 、 ``ZEND_ACC_STATIC`` 、 ``ZEND_ACC_FINAL`` 、 ``ZEND_ACC_ABSTRACT`` の組み合わせによって普段のPHPのメソッドの修飾子を指定することが出来ます。例えば、protected final static のメソッドの宣言は次の通りです。

.. code-block:: c

  PHP_ME(
      Test, protectedFinalStaticMethod, arginfo_xyz,
      ZEND_ACC_PROTECTED | ZEND_ACC_FINAL | ZEND_ACC_STATIC
  )

abstractのメソッドというのはそれに関連する実装をもたないので、 ``ZEND_ACC_ABSTRACT`` フラグは直接使用されることはありません。代わりに特別なマクロが提供されています。

.. code-block:: c 

  PHP_ABSTRACT_ME(Test, abstractMethod, arginfo_abc)


``PHP_FUNCTION`` の働きと同じように、 ``PHP_METHOD`` マクロは特別な名前をもつ関数宣言に展開され、この名前はメソッド呼び出しのバックトレースで見つけることが出来るでしょう。

.. code-block :: c

  PHP_METHOD(ClassName, methodName) { }
  /* 上記を展開すると次のようになる */
  void zim_ClassName_methodName(INTERNAL_FUNCTION_PARAMETERS) { }

しかしまずは、メソッドを書くことに戻りましょう。下記は別の例です。

.. code-block:: c

  PHP_METHOD(Test, getOwnObjectHandle)
  {
      zval *obj;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      obj = getThis();  

      RETURN_LONG(Z_OBJ_HANDLE_P(obj));
  }  

  //...
      PHP_ME(Test, getOwnObjectHandle, arginfo_void, ZEND_ACC_PUBLIC)
  //...

このメソッドは自身のオブジェクトが所有しているオブジェクトハンドルを返しているに過ぎません。そのためには、まず ``getThis()`` マクロを使って ``$this`` のzvalを取得し、それから ``Z_OBJ_HANDLE_P`` によって提供されるオブジェクトハンドルを返しています。実際に試してみましょう。

.. code-block:: php

  <?php  

  $t1 = new Test;
  $other = new stdClass;
  $t2 = new Test;
  echo $t1, "\n", $t2, "\n";

このコードは(おそらく)1と3を出力するでしょう。このことから、オブジェクトハンドルは基本的にはオブジェクトが新しく作られる度にインクリメントされる数字に過ぎないということが分かります(関連オブジェクトが破棄されると、オブジェクトハンドルが再利用することが出来るので、これは正確には正しくはありません)。


プロパティと定数
------------------

もっと役立つことを出来るようにするために、プロパティの読み書きをするための2つのメソッドを作ってみましょう。

.. code-block:: c

  PHP_METHOD(Test, getFoo)
  {
      zval *obj, *foo_value;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      obj = getThis();  

      foo_value = zend_read_property(test_ce, obj, "foo", sizeof("foo") - 1, 1 TSRMLS_CC);  

      RETURN_ZVAL(foo_value, 1, 0);
  }  

  PHP_METHOD(Test, setFoo)
  {
      zval *obj, *new_foo_value;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &new_foo_value) == FAILURE) {
          return;
      }  

      obj = getThis();  

      zend_update_property(test_ce, obj, "foo", sizeof("foo") - 1, new_foo_value TSRMLS_CC);
  }  

  // ...  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_void, 0, 0, 0)
  ZEND_END_ARG_INFO()  

  ZEND_BEGIN_ARG_INFO_EX(arginfo_set, 0, 0, 1)
      ZEND_ARG_INFO(0, value)
  ZEND_END_ARG_INFO()  

  // ...
      PHP_ME(Test, getFoo, arginfo_void, ZEND_ACC_PUBLIC)
      PHP_ME(Test, setFoo, arginfo_set, ZEND_ACC_PUBLIC)
  // ...

上記のコードでの新しい2つの関数は ``zend_read_property()`` と ``zend_update_property()`` です。両方の関数とも、最初のパラメーターとしてスコープを受け取り、2番目のパラメーターでそのオブジェクトを、その後にプロパティの名前や長さを受け取ります。ここでの"スコープ"とはクラスエントリのことで、アクセス権の制御で必要となります。 ``foo`` がpublicなプロパティであれば、使用されるスコープは問題とはなりません( ``NULL`` でも差し支え無いでしょう)が、privateなプロパティの場合は、そのクラスに属するクラスエントリでしかアクセス出来ません。

``zend_update_property()`` は追加でプロパティの新しい値を最後のパラメーターとして受け取ります。一方で、 ``zend_read_property()`` は真偽値の ``silent`` パラメーターを追加で受け取ります。これはPHPが"Undefined property xyz"のようなnoticeを抑制すべきかどうかを指定するためのものです。このケースでは、前もってそのプロパティが存在するかどうか分からないので、 ``1`` を渡しています(この場合、そのnoticeは出力しないという意味になります)。

では新しい機能を使ってみましょう。

.. code-block:: php

  <?php  

  $t = new Test;
  var_dump($t->getFoo()); // NULL (silent=1としているので、noticeは出力されない)  

  $t->setFoo("abc");
  var_dump($t->foo);      // "abc"
  var_dump($t->getFoo()); // "abc"  

  $t->foo = "def";
  var_dump($t->foo);      // "def"
  var_dump($t->getFoo()); // "def"

``zend_update_property()`` には特定の値をより簡単に(つまり手動でzvalを作らなくても)設定できるように様々な種類があります。

 - ``zend_update_property_null(... TSRMLS_DC)``
 - ``zend_update_property_bool(..., long value TSRMLS_DC)``
 - ``zend_update_property_long(..., long value TSRMLS_DC)``
 - ``zend_update_property_double(..., double value TSRMLS_DC)``
 - ``zend_update_property_string(..., const char *value TSRMLS_DC)``
 - ``zend_update_property_stringl(..., const char *value, int value_len TSRMLS_DC)``

上の例では、 ``silent=1`` パラメーターを使わなければなりませんでした。なぜなら ``foo`` プロパティを読む際に存在しているかの保証がなかったからです。これのより良い解決法は、クラスの登録時に ちょうどPHPで ``public $foo = DEFAULT_VALUE;`` とするように、そのプロパティを適切に宣言しておくということです。

これは ``zend_declare_property()`` 関数グループでおこなうことができ、この関数には ``zend_update_property()`` と同じ種類があります。例として、 ``foo`` というデフォルト値が ``null`` でpublicプロパティを宣言するには、 ``MINIT`` のクラス登録の後に下記の行を追加する必要があります。

.. code-block:: c

  zend_declare_property_null(test_ce, "foo", sizeof("foo") - 1, ZEND_ACC_PUBLIC TSRMLS_CC);

デフォルト値が ``"bar"`` でprotectedなプロパティをつくる場合は、代わりに次のように書いて下さい。

.. code-block:: c

  zend_declare_property_string(
      test_ce, "foo", sizeof("foo") - 1, "bar", ZEND_ACC_PROTECTED TSRMLS_CC
  );

プロパティを使いたい場合(そして内部のクラスにとってほとんど必要のないものと分かった場合)、プロパティを適切に宣言しておくことは、常に良い方法です。この方法では明確なアクセス権のレベルやデフォルト値を指定できるので、宣言されたプロパティのメモリ最適化の恩恵も受けられます。

staticのプロパティもまた同じ関数のグループで ``ZEND_ACC_STATIC`` のフラグを追加で指定することで宣言できます。public staticのプロパティ ``$pi`` の例は次の通りです。

.. code-block:: c

  zend_declare_property_double(
      test_ce, "pi", sizeof("pi") - 1, 3.141, ZEND_ACC_PUBLIC | ZEND_ACC_STATIC TSRMLS_CC
  );
  /* 私が記憶しているpiの全ての数値 :( */


staticのプロパティを読み書きするために、 ``zend_read_static_property()`` 関数や ``zend_update_static_property()`` 関数のグループがあります。それらの関数は通常のプロパティの関数と同じインターフェイスですが、オブジェクトを受け取らない(スコープのみです)という点のみが異なります。

定数を宣言するためには ``zend_declare_class_constant_*()`` の関数グループが使われます。これらの関数は ``zend_declare_property_*()`` の関数と同じ種類とシグネチャを持っていますが、フラグの引数だけ受け取りません。"Test::PI" という定数を宣言するためには次のように書きます。

.. code-block:: c

  zend_declare_class_constant_double(test_ce, "PI", sizeof("PI") - 1, 3.141 TSRMLS_CC);

継承とインターフェイス
-----------------------

ユーザーランドとちょうど同じように、内部のクラスも他のクラスを継承したりインターフェイスを実装することが出来ます。

PHPの継承のとてもシンプル(そして非常に一般的)な例として、 ``Exception`` クラスのサブクラスを作ってみます。

.. code-block:: c

  zend_class_entry *custom_exception_ce;  

  PHP_MINIT_FUNCTION(Test)
  {
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "CustomException", NULL);
      custom_exception_ce = zend_register_internal_class_ex(
          &tmp_ce, zend_exception_get_default(TSRMLS_C), NULL TSRMLS_CC
      );  

      return SUCCESS;
  }

ここでの新しい部分は ``zend_register_internal_class_ex()`` ( ``_ex`` が付きます)の使用で、これは ``zend_register_internal_class()`` と同じものですが、追加で親クラスエントリを指定することが出来ます。ここでは ``zend_exception_get_default(TSRMLS_C)`` を使って親クラスエントリを取得しています。注目に値する別のポイントは、関数の構造体を何も宣言しておらず、代わりにただ ``INIT_CLASS_ENTRY`` の最後の引数に ``NULL`` を渡しているだけだということです。これは ``Exception`` クラスから継承したものを除いて、追加で何のメソッドも必要ないということを意味しています。

``RuntimeException`` のようなより具体的なSPL拡張クラスを拡張したい場合には、次のようにします。

.. code-block:: c

  #ifdef HAVE_SPL
  #include "ext/spl/spl_exceptions.h"
  #endif  

  zend_class_entry *custom_exception_ce;  

  PHP_MINIT_FUNCTION(Test)
  {
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "CustomException", NULL);  

  #ifdef HAVE_SPL
      custom_exception_ce = zend_register_internal_class_ex(
          &tmp_ce, spl_ce_RuntimeException, NULL TSRMLS_CC
      );
  #else
      custom_exception_ce = zend_register_internal_class_ex(
          &tmp_ce, zend_exception_get_default(TSRMLS_C), NULL TSRMLS_CC
      );
  #endif  

      return SUCCESS;
  }


上記のコードは条件的には ``RuntimeException`` を継承するか、SPLがコンパイルされていない場合には単に ``Exception`` を継承のどちらかになります。 ``RuntimeException`` のクラスエントリは ``ext/spl/spl_exceptions.h`` でexternされているので、同様にincludeしなければなりません。

上のコードで ``NULL`` で設定されている ``zend_register_internal_class_ex()`` の最後のパラメーターは親クラスを指定するための別の方法です。もし利用可能なクラスエントリがない場合には、クラス名を指定することが出来ます。

.. code-block:: c

  custom_exception_ce = zend_register_internal_class_ex(
      &tmp_ce, spl_ce_RuntimeException, NULL TSRMLS_CC
  );
  /* can also be written as */
  custom_exception_ce = zend_register_internal_class_ex(
      &tmp_ce, NULL, "RuntimeException" TSRMLS_CC
  );

実際問題、最初の方の例を好んで使うべきではあります。2番目の方は、クラスエントリを宣言するのを忘れた無作法なエクステンションを使う場合のみしか有効ではありません。

他のクラスを継承することが出来るように、インターフェイスの実装もまた可能です。このためには、可変長引数の ``zend_class_implements()`` 関数が使用されます。

.. code-block ::c

  #include "ext/spl/spl_iterators.h"
  #include "zend_interfaces.h"  

  zend_class_entry *data_class_ce;  

  PHP_METHOD(DataClass, count) { /* ... */ }  

  const zend_function_entry data_class_functions[] = {
      PHP_ME(DataClass, count, arginfo_void, ZEND_ACC_PUBLIC)
      /* ... */
      PHP_FE_END
  };  

  PHP_MINIT_FUNCTION(test)
  {
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "DataClass", data_class_functions);
      data_class_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);  

      /* DataClass は Countable, ArrayAccess, IteratorAggregateを実装している */
      zend_class_implements(
          data_class_ce TSRMLS_CC, 3, spl_ce_Countable, zend_ce_arrayaccess, zend_ce_aggregate
      );  

      return SUCCESS;
  }


ご覧のように、 ``zend_class_implements()`` はクラスエントリ、TSRMLS_CC、実装するインターフェイスの数、インターフェイスのクラスエントリを受け取ります。例えば、 ``Serializable`` を追加で実装したいとしましょう。

.. code-block:: c

  zend_class_implements(
      data_class_ce TSRMLS_CC, 4,
      spl_ce_Countable, zend_ce_arrayaccess, zend_ce_aggregate, zend_ce_serializable
  );


当然、自身でインタフェースを作成することも出来ます。インターフェイスはクラスと同じ方法で登録されますが、 ``zend_register_internal_interface()`` 関数を使って全てのメソッドがabstractとして宣言します。例えば、 ``Iterator`` を拡張して ``ReversibleIterator`` という新しいインターフェイスを作って、追加で ``prev`` メソッドを指定したい場合、次のような方法で行います。

.. code-block:: c

  #include "zend_interfaces.h"  

  zend_class_entry *reversible_iterator_ce;  

  const zend_function_entry reversible_iterator_functions[] = {
      PHP_ABSTRACT_ME(ReversibleIterator, prev, arginfo_void)
      PHP_FE_END
  };  

  PHP_MINIT_FUNCTION(test)
  {
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "ReversibleIterator", reversible_iterator_functions);
      reversible_iterator_ce = zend_register_internal_interface(&tmp_ce TSRMLS_CC);  

      /* ReversibleIterator は Iteratorを継承している。 インターフェイスの継承には
       * zend_class_implements()関数が使用される */
      zend_class_implements(reversible_iterator_ce TSRMLS_CC, 1, zend_ce_iterator);  

      return SUCCESS;
  }

内部のインターフェイスにはユーザーランドのインターフェイスには無いちょっとした追加の機能があります。しかしこれは後に残しておきます。
