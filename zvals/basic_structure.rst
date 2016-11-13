基本構造
=========

zval("Zend value"を短くしたもの) はPHPの任意の値を表現してます。PHP全体の中で最も重要な構造であるため、zvalを使う機会は多くなるでしょう。このセクションではzvalの背後にある基本コンセプトと使用方法を説明します。

型と値
-------

とりわけ、zvalは値とその型を保持しています。PHPは動的型付け言語で、コンパイル時ではなく実行時に変数の型が決まるのでこれは必要なことです。さらに、型はzvalが使われている間に変更することができるため、以前には整数を保持していても後には文字列を保持しているかもしれません。

型は整数のタグ(unsigned char)として保持されています。これには8つの値の種類があり、これはPHPで使用できる8つの型に対応しています。この値は ``IS_TYPE`` の定数を使用する際に参照されています。例えば ``IS_NULL`` はNULL型に対応していますし、 ``IS_STRING`` は文字列型に対応しています。

変数の実際の値は共有体で保持され、下記のように定義されています。

.. code-block:: c

  typedef union _zvalue_value {
      long lval;
      double dval;
      struct {
          char *val;
          int len;
      } str;
      HashTable *ht;
      zend_object_value obj;
  } zvalue_value;

共有体に馴染みがない人のために説明しておくと、共有体は複数のメンバーを定義できますが、1度に使用できるメンバーはひとつだけです。例えば、 ``value.lval`` メンバーがセットされると他のメンバーではなく ``value.lval`` を使って値を参照する必要があります(さもないと"厳密な別名規約(Strict Aliasing)"の違反になり、未定義の動作につながります)。というのも、共有体が全てのメンバーを同じメモリ領域に保持しており、そこに保持されている値をあなたがアクセスするメンバーによってそれぞれ解釈しているだけからです。共有体のサイズはメンバーの中で一番大きいメンバーのサイズとなります。

zvalを使用する際には型のタグを使う事で、共有体のどのメンバーが現在使用されているかを確かめることができます。そのためのAPIを見て行く前に、PHPがサポートする型とそれらがどのように保持されているかを見ていくことにしましょう。

最もシンプルな型は ``IS_NULL`` です。これは値を実際保持する必要はなく ``null`` 値だけです。

数値を保持するためのものとして、PHPには ``IS_LONG`` と ``IS_DOUBLE`` という型が提供されており、それぞれ ``long lval`` と ``double dval`` メンバーを利用します。前者は整数、後者は浮動小数点数を保持するために使用されます。

``long`` 型について意識しておくべきことがいくつかあります。第一に、この型は符号付き整数型であるということです。つまり正と負の数の両方を保持できますが、これは一般的にビット演算には適していません。第二に、 ``long`` 型はプラットフォームによってサイズが異なるということです。つまり、 32bitシステムのOSであれば32 bits / 4 bytesの大きさですし、64bitシステムのOSであれば4あるいは8bytesのどちらかになるでしょう。特に64bitのUnixであれば一般的には8byteですが、64bitのWindowsであれば4bytesとなります。

こういった理由から、 ``long`` 型のいかなるサイズにも依存してはいけません。 ``long`` 型が保持できる値の最小値と最大値は ``LONG_MIN`` と ``LONG_MAX`` から取得でき、型のサイズは ``SIZEOF_LONG`` を使ってアクセス可能です(これは ``sizeof(long)`` と違って ``#if`` ディレクティブの中で使用可能です)。

浮動小数点数を保持するために使用される ``double`` 型はIEEE-754の仕様に従って一般的に8byteの値となります。この形式についてここで詳細は述べませんが、少なくともこの型の精度は限定されており正確な値を保持することはできないことに注意してください。

真偽値は ``IS_BOOL`` フラグを使用し、 ``long lval`` メンバーに 0(偽) と1(真)の値で保持されます。2通りの値しかないので、理論上は代わりにもっと小さな型(例えばunsigned charの ``zend_bool`` など)が使えますが ``zvalue_value`` の共有体はメンバーの中で一番大きなサイズとなるため、そのようにしても実際メモリーの節約にはなりません。そのため、``lval`` メンバーを再利用しています。

文字列( ``IS_STRING`` )は ``struct { char *val; int len; } str`` 構造体に保持されます。つまり、 ``char*`` の文字列と ``int`` の文字列の長さで構成されています。PHPの文字列はその中でNUL bytes( ``'\0'`` )を使用できるよう(バイナリセーフ)に、明確な文字列の長さを保持している必要があります。それにもかかわらず、PHPの文字列はヌルバイト文字で終わるようになっていますが、これは引数に文字列の長さを受け取らない代わりにヌルバイト文字で終わる文字列を期待しているようなライブラリーの関数との相互運用を容易にするためです。もちろんこの場合、文字列はバイナリセーフではなくなり、最初のヌル文字のところで切り取られることになります。例えば、libc の文字列の関数と同様、多くのファイルシステムに関連する関数ではこのように振る舞うでしょう。

文字列の長さはbyteであり(Unicodeの符号点ではありません)、ヌルバイト文字を含んでは **いません** 。つまり、``foo`` の場合、実際に保持する際には4byte使用されますが、文字列の長さは3となります。もし ``sizeof`` を使用して定数文字列の長さを取得した場合は1を減算する必要があることに注意してください。つまり ``strlen("foo") == sizeof("foo") - 1`` となります。

さらに、文字列の長さは ``long`` や他の型ではなく ``int`` で保持される事を理解することも大事な点です。これは不幸な歴史の産物であり、文字列の長さは2147483647 byteに限定されます。これよりも大きい文字列はオーバーフローを引き起こすでしょう(つまり負の長さになります)。

残りの3つの型についてはここでは簡単にだけ述べ、後ほどそれぞれの章で詳しくみていきましょう。

配列型は ``IS_ARRAY`` を使用し、``HashTable *ht`` メンバーに保持されます。 ``HashTable`` 構造体がどのように動作していくかは :doc:`/hashtables` で説明します。

オブジェクト型( ``IS_OBJECT`` )は ``zend_object_value obj`` メンバーを使用し、これはオブジェクトの実際の値を参照する際に使用される整数のIDである“object handle”と、オブジェクトがどのように振る舞うかを定義する“object handlers”の集合によって構成されています。PHPのクラスとオブジェクトの仕組みは :doc:`/classes_objects` の章で説明します。

リソース型( ``IS_RESOURCE`` )は、実際の値を参照するための一意のIDをもっているという点ではオブジェクト型と似ています。このIDは ``long lval`` メンバーで保持されます。リソース型はリソースの章(まだありません)で扱っていきます。

まとめると、利用可能な型のタグとその値が保持される場所は以下の表の通りになります。


======================   ======================================
Type tag                  Storage location
======================   ======================================
``IS_NULL``              none
``IS_BOOL``              ``long lval`` 
``IS_LONG``              ``long lval``
``IS_DOUBLE``            ``double dval``
``IS_STRING``            ``struct { char *val; int len; } str``
``IS_ARRAY``             ``HashTable *ht``
``IS_OBJECT``            ``zend_object_value obj``
``IS_RESOURCE``          ``long lval``
======================   ======================================

アクセスマクロ
---------------

では ``zval`` 構造体が実際どのようなものかみていきましょう。

.. code-block:: c

  typedef struct _zval_struct {
      zvalue_value value;
      zend_uint refcount__gc;
      zend_uchar type;
      zend_uchar is_ref__gc;
  } zval;


既に述べたように、zvalは ``value`` と その ``type`` を保持するためのメンバーを持っています。前述したように、値は ``zvalue_value`` 共有体で保持され、型のタグは ``zend_uchar`` でもっています。さらに、構造体は ``__gc`` で終わるプロパティがありますが、これはPHPが採用しているガーベッジコレクションの仕組みのために使用されます。今のところはこれらは一旦扱いませんが、次のセクションで説明致します。

これまでの説明を踏まえると、zvalは次のように使うことが出来ます。

.. code-block:: c

  zval *zv_ptr = /* ... zvalを何らかの方法で取得する */;

  if (zv_ptr->type == IS_LONG) {
      php_printf("Zval is a long with value %ld\n", zv_ptr->value.lval);
  } else /* ... 他の型の時の処理 */

上記のコードは問題なく動作しますが、このような処理としてはあまり一般的な書き方ではありません。zvalのメンバーにアクセスするためのマクロを使用せずに直接メンバーを参照してしまっています。

.. code-block:: c

  zval *zv_ptr = /* ... */;

  if (Z_TYPE_P(zv_ptr) == IS_LONG) {
      php_printf("Zval is a long with value %ld\n", Z_LVAL_P(zv_ptr));
  } else /* ... */

上記のコードでは型のタグを取得するために ``Z_TYPE_P()`` マクロを、long型の整数の値を取得するために ``Z_LVAL_P()`` マクロを使用しています。アクセスマクロには接尾辞の ``_P`` 、``_PP`` がつくものと、接尾辞なしの種類があります。どれを使用するかは ``zval`` 、``zval*`` 、``zval**`` のどれを扱うかで変わってきます。

.. code-block:: c

  zval zv;
  zval *zv_ptr;
  zval **zv_ptr_ptr;
  zval ***zv_ptr_ptr_ptr;

  Z_TYPE(zv);                 // = zv.type
  Z_TYPE_P(zv_ptr);           // = zv_ptr->type
  Z_TYPE_PP(zv_ptr_ptr);      // = (*zv_ptr_ptr)->type
  Z_TYPE_PP(*zv_ptr_ptr_ptr); // = (**zv_ptr_ptr_ptr)->type

基本的には ``P`` の数は型の ``*`` の数と一致するはずです。しかしこれは ``zval**`` までしか有効でなく、``zval***`` へアクセスするための特別なマクロは、実際滅多に必要にならないので、用意されていません( ``*`` 演算子を使ってポインターからその値を取得しなければなりません)。

``Z_LVAL`` と同様に、他の全ての型の値を取得するためのマクロが用意されています。それらのデモのために、zvalをdumpする簡単な関数をつくってみましょう。

.. code-block:: c

  PHP_FUNCTION(dump)
  {
      zval *zv_ptr;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &zv_ptr) == FAILURE) {
          return;
      }  

      switch (Z_TYPE_P(zv_ptr)) {
          case IS_NULL:
              php_printf("NULL: null\n");
              break;
          case IS_BOOL:
              if (Z_BVAL_P(zv_ptr)) {
                  php_printf("BOOL: true\n");
              } else {
                  php_printf("BOOL: false\n");
              }
              break;
          case IS_LONG:
              php_printf("LONG: %ld\n", Z_LVAL_P(zv_ptr));
              break;
          case IS_DOUBLE:
              php_printf("DOUBLE: %g\n", Z_DVAL_P(zv_ptr));
              break;
          case IS_STRING:
              php_printf("STRING: value=\"");
              PHPWRITE(Z_STRVAL_P(zv_ptr), Z_STRLEN_P(zv_ptr));
              php_printf("\", length=%d\n", Z_STRLEN_P(zv_ptr));
              break;
          case IS_RESOURCE:
              php_printf("RESOURCE: id=%ld\n", Z_RESVAL_P(zv_ptr));
              break;
          case IS_ARRAY:
              php_printf("ARRAY: hashtable=%p\n", Z_ARRVAL_P(zv_ptr));
              break;
          case IS_OBJECT:
              php_printf("OBJECT: ???\n");
              break;
      }
  }  

  const zend_function_entry funcs[] = {
      PHP_FE(dump, NULL)
      PHP_FE_END
  };


では実行してみます。

.. code-block:: c

  dump(null);                 // NULL: null
  dump(true);                 // BOOL: true
  dump(false);                // BOOL: false
  dump(42);                   // LONG: 42
  dump(4.2);                  // DOUBLE: 4.2
  dump("foo");                // STRING: value="foo", length=3
  dump(fopen(__FILE__, "r")); // RESOURCE: id=???
  dump(array(1, 2, 3));       // ARRAY: hashtable=0x???
  dump(new stdClass);         // OBJECT: ???

値にアクセスするためのマクロは非常に簡単です。 ``Z_BVAL`` は真偽値、 ``Z_LVAL`` はlong型、 ``Z_DVAL`` はdouble型です。文字列では ``Z_STRVAL`` が実際の ``char*`` の文字列の値を返し、``Z_STRLEN`` はその長さを返します。リソース型のIDは ``Z_RESVAL`` から取得でき、配列の ``HashTable*`` は ``Z_ARRVAL`` からアクセスできます。オブジェクト型へのアクセスについては、もう少し予備知識が必要なため、ここではふれません。

zvalのメンバーにアクセスする際には直接参照するのではなく、常にマクロを通しておこなうようにしてください。これによって抽象性が維持されますし、コードの意図が明確になります。例えば、 ``lval`` メンバーに直接アクセスするといった場合、それは真偽値としての値、整数の値、リソース型用のIDを取得する場合のいずれかでしょう。代わりに ``Z_BVAL`` 、 ``Z_LVAL`` 、 ``Z_RESVAL`` を使うことで、コードの意図が曖昧とならないようになるのです。また、マクロを使うと将来zvalの内部構造が変わっても、その変化に強くなります。

値を設定する
-------------

上で説明したマクロのほとんどは単にzvalのメンバーにアクセスしているだけですので、これらのマクロでそれぞれの値の読みと書きの両方が可能です。"hello world!"の文字列を単に返すだけの関数を例にみてみましょう。

.. code-block:: c

  PHP_FUNCTION(hello_world) {
      Z_TYPE_P(return_value) = IS_STRING;
      Z_STRVAL_P(return_value) = estrdup("hello world!");
      Z_STRLEN_P(return_value) = strlen("hello world!");
  };

  /* ... */
      PHP_FE(hello_world, NULL)
  /* ... */

``php -r "echo hello_world();"`` と実行すればターミナルに ``hello world!`` と表示されるはずです。

上の例では ``PHP_FUNCTION`` によって提供される ``zval*`` の ``return_value`` という変数を設定しています。この変数に関しては次の章でより詳しく見ていきますが、今のところ、この変数の値が関数での返り値となるということを理解すれば十分です。これはデフォルトでは ``IS_NULL`` の型で初期化されています。

アクセスマクロを使ってzvalに値を設定していくことは非常に簡単ですが、注意すべきことがいくつかあります。まず第一に、型のタグがzvalの型を決定していることを念頭に置く必要があります。単に値を設定(ここでは ``Z_STRVAL`` や ``Z_STRLEN`` を通して)するだけでは十分でなく、常に型のタグも同様に設定する必要があります。

これに加え、多くの場合にzvalはその値を"保持"しているということと、zvalはその値をセットしたスコープよりも広い生存期間をもつことになるということに注意しておく必要があります。一時的なzvalを扱う場合にはこれが当てはまらないこともありますが、ほとんどのケースでこれが当てはまります。

上の例では、 ``return_value`` がその関数を抜けても生き続けるということを意味しますが(明らかですが、そうでなければreturn valueが使えません)、関数内での一時的な値に関しては使うことは出来ません。例えば、 ``Z_STRVAL_P(return_value) = "hello world!"`` と書くだけでは、 ``"hello world!"`` の文字列は関数を抜けると存在しなくなる(これはC言語でスタックに割り当てられたどの値にとっても同様です)ので、上手く動きません。

このため、 ``estrdup()`` を使って文字列をコピーする必要があります。これによってヒープ上にその文字列の別のコピーがつくられます。zvalが値を"保持"しているので、zvalが破棄される際には忘れずにこのコピーのメモリーを解放するようにします。これはzvalの他の複雑な値に関しても同様で、例えば配列型のための ``HashTable*`` などに値を設定した場合には、zvalがそれを保持していて破棄されるタイミングで解放します。interger型やdoubles型などのプリミティブな型を使用する場合には、常にコピーされるので気にする必要はありません。

最後に、全てのアクセスマクロが直接メンバーを返すわけではないという事を指摘しておかなければなりません。例えば ``Z_BVAL`` マクロは次のように定義されています。

.. code-block:: c

  #define Z_BVAL(zval) ((zend_bool)(zval).value.lval)

このマクロでは型変換されてますので ``Z_BVAL_P(return_value) = 1`` と書くことはできません。オブジェクト型に関連するマクロを除いて、これは唯一の例外です。他の全てのマクロで値を設定することができます。

実際のところ最後に指摘した点に関しては心配する必要はありません。というのも、zvalに値を設定するというような一般的なことについては、PHPではそれ専用に別のマクロが用意されています。そのマクロを使えば型のタグと値が同時に設定できます。前の例をそのマクロを使って書きなおしてみましょう。

.. code-block:: c

  PHP_FUNCTION(hello_world) {
      ZVAL_STRINGL(return_value, estrdup("hello world!"), strlen("hello world!"), 0);
  }

zvalを割り当てる際に文字列をコピーをとる必要があるのはよくあることですので、 ``ZVAL_STRINGL`` の最後の引数(boolean)によってコピーをするかどうかの制御ができるようになっています。 ``0`` を渡すと文字列はそのまま使用され、 ``1`` を渡すと ``estrndup()`` でコピーされます。さきほどの例を書きなおすと次のようになります。

.. code-block:: c

  PHP_FUNCTION(hello_world) {
      ZVAL_STRINGL(return_value, "hello world!", strlen("hello world!"), 1);
  }

さらに、わざわざ ``strlen`` で長さを取らなくとも、代わりに ``ZVAL_STRING`` マクロ(最後に ``L`` がありません)を使うことが出来ます。

.. code-block:: c

  PHP_FUNCTION(hello_world) {
      ZVAL_STRING(return_value, "hello world!", 1);
  }

(何らかの方法で既に受け取っていて)文字列の長さがわかっている場合には、バイナリセーフを保つために常に ``ZVAL_STRINGL`` マクロを通して使うべきです。もし長さがわかっていない場合(あるいはリテラルの場合のように文字列にヌルバイト文字が含まれているかどうかわからない場合)は、代わりに ``ZVAL_STRING`` が使えます。

``ZVAL_STRING(L)`` を除いて、下記の例にあるように、値を設定するためのマクロは他に下記のものがあります。

.. code-block:: c

  ZVAL_NULL(return_value);  

  ZVAL_BOOL(return_value, 0);
  ZVAL_BOOL(return_value, 1);
  /* あるいは、もっと良い書き方としては下記のようにする */
  ZVAL_FALSE(return_value);
  ZVAL_TRUE(return_value);  

  ZVAL_LONG(return_value, 42);
  ZVAL_DOUBLE(return_value, 4.2);
  ZVAL_RESOURCE(return_value, resource_id);  

  ZVAL_EMPTY_STRING(return_value);
  /* = ZVAL_STRING(return_value, "", 1); */  

  ZVAL_STRING(return_value, "string", 1);
  /* = ZVAL_STRING(return_value, estrdup("string"), 0); */  

  ZVAL_STRINGL(return_value, "nul\0string", 10, 1);
  /* = ZVAL_STRINGL(return_value, estrndup("nul\0string", 10), 10, 0); */

これらのマクロは値を設定しますが、zvalが既に保持しているどんな値も削除しないことに注意してください。 ``return_value`` のzvalに関しては ``IS_NULL`` で初期化されています(解放すべき値がなにもありません)ので特に問題はありませんが、他の場合には次のセクションで説明する関数を利用して古い値を破棄する必要があります。

