キャストと演算
===============

基本的な演算
-------------

zvalは複雑な構造なので、それらを ``zv1 + zv2`` のような基本的な演算を直接おこなうようなことは出来ません。このようにしてしまうとエラーとなるか両方の値ではなくむしろポインターが加算されるだけで終わってしまうかのどちらかでしょう。

``+`` のような基本的な演算子はzvalで使う際には、多くの型に対応しないといけないので、かなり複雑です。例えば、PHPではdouble型と整数の文字列との足し算( ``3.14 + "17"`` )や、配列同士の足し算( ``[1, 2, 3] + [4, 5, 6]`` )でさえも可能です。

そのため、PHPにはzvalの演算のための特別なマクロが用意されています。例えば、加算は ``add_function()`` で処理できます。

.. code-block:: c

  zval *a, *b, *result;
  MAKE_STD_ZVAL(a);
  MAKE_STD_ZVAL(b);
  MAKE_STD_ZVAL(result);  

  ZVAL_DOUBLE(a, 3.14);
  ZVAL_STRING(b, "17", 1);  

  /* result = a + b */
  add_function(result, a, b TSRMLS_CC);  

  php_printf("%Z\n", result); /* 20.14 */  

  /* zvals a, b, result need to be dtored */

``add_function()`` とは別に他にもいくつかの二項演算子を実装した関数があり、全ての引数と戻り値は同じです。

.. code-block:: c

  int add_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  +  */
  int sub_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  -  */
  int mul_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  *  */
  int div_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  /  */
  int mod_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  %  */
  int concat_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);              /*  .  */
  int bitwise_or_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);          /*  |  */
  int bitwise_and_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /*  &  */
  int bitwise_xor_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /*  ^  */
  int shift_left_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);          /*  << */
  int shift_right_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /*  >> */
  int boolean_xor_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /* xor */
  int is_equal_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);            /*  == */
  int is_not_equal_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);        /*  != */
  int is_identical_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);        /* === */
  int is_not_identical_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);    /* !== */
  int is_smaller_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);          /*  <  */
  int is_smaller_or_equal_function(zval *result, zval *op1, zval *op2 TSRMLS_DC); /*  <= */

全ての関数は ``op1`` と ``op2`` の演算の結果を格納するための ``result`` のzvalを受け取ります。 ``int`` の戻り値は演算が成功したかを示す ``SUCCESS`` か ``FAILURE`` のどちらかです。 ``result`` はたとえ演算が成功しなくても、何らかの値(例えば ``false`` )が設定されていることに注意してください。

``result`` のzvalは関数を呼び出す前にメモリ割り当てと初期化を済ませておく必要があります。あるいは複合代入演算子の場合には、 ``result`` と ``op1`` は同じに出来ます。

.. code-block:: c

  zval *a, *b;
  MAKE_STD_ZVAL(a);
  MAKE_STD_ZVAL(b);  

  ZVAL_LONG(a, 42);
  ZVAL_STRING(b, "3");  

  /* a += b */
  add_function(a, a, b TSRMLS_CC);  

  php_printf("%Z\n", a); /* 45 */  

  /* aとbのzvalは破棄する必要がある */

いくつかの二項演算子は上の関数のなかにはありません。例えば ``>`` や ``>=`` の関数などです。この理由は ``is_smaller_function()`` や ``is_smaller_or_equal_function()`` の関数でオペランドを入れ替えるて使うだけで実装できるからです。

また ``&&`` や ``||`` のための関数もありません。それらの演算子は主に短絡評価で動作し、簡単な関数では実装できないからです。短絡評価を一旦横においておくと、両方の演算子は単にC言語の演算子の ``&&`` や ``||`` に続いてbooleanのキャストがおこなわれるだけです。

二項演算子以外では、２つの単項演算子用の関数があります。

.. code-block:: c

  int boolean_not_function(zval *result, zval *op1 TSRMLS_DC); /*  !  */
  int bitwise_not_function(zval *result, zval *op1 TSRMLS_DC); /*  ~  */

これらも他の演算のマクロと同様に動作しますが、ひとつのオペランドしか受け取りません。単項の ``+`` や ``-`` 演算がないのは ``add_function()`` や ``sub_function()`` などを使って、それぞれ ``0 + $value`` や ``0 - $value`` とすることで実装できるからです。

最後は ``++`` と ``--`` 演算のマクロです。

.. code-block:: c

  int increment_function(zval *op1); /* ++ */
  int decrement_function(zval *op1); /* -- */

これらは結果を格納するzvalを受け取らず、代わりに渡されたオペランドを直接変更します。これらは ``add_function()`` や ``sub_function()`` を使って ``+ 1`` や ``- 1`` するのとでは違う動作になることに注意してください。例えば ``"a"`` をインクリメントすると ``"b"`` となりますが、 ``"a" + 1`` は ``1`` となるからです。

比較
------

上で紹介した比較のための関数は全て特定の演算子に対応しています。例えば ``is_equal_function()`` は ``==`` に、 ``is_smaller_function()`` は ``<`` に対応しています。それら全ての代わりとして ``compare_function()`` というものがあり、これはより汎用的に比較をおこないます。

.. code-block:: c

  zval *a, *b, *result;
  MAKE_STD_ZVAL(a);
  MAKE_STD_ZVAL(b);
  MAKE_STD_ZVAL(result);  

  ZVAL_LONG(a, 42);
  ZVAL_STRING(b, "24");  

  compare_function(result, a, b TSRMLS_CC);  

  if (Z_LVAL_P(result) < 0) {
      php_printf("a is smaller than b\n");
  } else if (Z_LVAL_P(result) > 0) {
      php_printf("a is greater than b\n");
  } else /*if (Z_LVAL_P(result) == 0)*/ {
      php_printf("a is equal to b\n");
  }  

  /* aとbとresultのzvalは破棄する必要がある */

``compare_function()`` は渡された値同士の関係の" ``op1`` は ``op2`` より小さい"、" ``op1`` は ``op2`` より大きい"、" ``op1`` と ``op2`` は等しい"に対応してそれぞれ ``result`` のzvalに-1、1、0を設定します。この関数もまた比較関数の多くのファミリーのうちの一部です。


.. code-block:: c

  int compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);  

  int numeric_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);  

  int string_compare_function_ex(zval *result, zval *op1, zval *op2, zend_bool case_insensitive TSRMLS_DC);
  int string_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);
  int string_case_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);  

  #ifdef HAVE_STRCOLL
  int string_locale_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);
  #endif

これら全ての関数もまた２つのオペランドとresultのzvalを受け取り ``SUCCESS`` / ``FAILURE`` を返します。

``compare_function()`` は通常のPHPの比較と同じように動作します(つまり ``<`` や ``>`` や ``==`` 演算子と同じように振る舞います)。 ``numeric_compare_function()`` はまずオペランドをdoubleにキャストすることで数字として比較します。

``string_compare_function_ex()`` はオペランドを文字列として比較し、大文字と小文字を区別するかどうかのフラグを指定できます。そのフラグを手動で指定する代わりに、 ``string_compare_function()`` (大文字と小文字の区別しない)や ``string_case_compare_function()`` (大文字と小文字を区別する)という関数を使うことも出来ます。これらの関数によっておこなわれる比較は、追加的に定義済みの数字の文字列を除いて通常の辞書順による文字列比較となります。

``string_locale_compare_function()`` は現在のロケール設定に応じた文字列比較をおこない、 ``HAVE_STRCOLL`` が定義済みの場合のみ利用できます。そのため、この関数を使う時ではいつでも ``#ifdef HAVE_STRCOLL`` を使う必要があります。他のロケールに関連したものと同様に、使用しないのが一番です。

型変換
--------

自身のコードを書く際にはzvalの特定のひとつ型のみをよく扱うかもしれません。例えば、文字列を操作するコードを書いている場合には文字列型のzvalのみを扱いたく、他の別の型には悩まされたくないかもしれません。一方で、PHPの動的型変換の仕組みもサポートしたいときっと思うでしょう。PHPでは数字を文字列として使うこともでき、エクステンションのコードでも同様にこの動作を受け入れなければなりません。

この解決策はzvalのどんな型も扱いたい特定の型に変換するということです。これに対応するため、PHPには全ての型のための ``convert_to_*`` という関数が用意されています( ``(resource)`` という型変換はないのでリソース型のものはありません)。

.. code-block:: c

  void convert_to_null(zval *op);
  void convert_to_boolean(zval *op);
  void convert_to_long(zval *op);
  void convert_to_double(zval *op);
  void convert_to_string(zval *op);
  void convert_to_array(zval *op);
  void convert_to_object(zval *op);  

  void convert_to_long_base(zval *op, int base);
  void convert_to_cstring(zval *op);

最後の２つの関数は標準的な型変換ではありません。というのも ``convert_to_long_base()`` は ``convert_to_long()`` と同じですが文字列をlongへの変換で特定の基数を使います(例: 16進数だと16)。 ``convert_to_cstring()`` は ``convert_to_string()`` のように振る舞いますが、ロケール設定に依存しないdoubleから文字列への変換をおこないます。これは変換の結果、小数点が ``"3,14"`` (ドイツ)のようにロケール設定による特有の文字を使うのではなく、常に ``.`` が使われることを意味します。

``convert_to_*`` 関数は引数で渡されたzvalを直接変更します。

.. code-block:: c

  zval *zv_ptr;
  MAKE_STD_ZVAL(zv_ptr);
  ZVAL_STRING(zv_ptr, "123 foobar", 1);

  convert_to_long(zv_ptr);

  php_printf("%ld\n", Z_LVAL_P(zv_ptr));

  zval_dtor(&zv_ptr);

zvalが複数箇所で共有されている場合(refcount > 1)は、もしかすると直接変更することが正しくない結果となってしまうかもしれません。例えば、値渡しでzvalを受け取って ``convert_to_*`` で直接変更すると、その関数の中でのzvalへの参照が変更されるだけでなく、関数の外からの参照も変更されます。

この問題を解決するために、PHPには ``convert_to_*_ex`` マクロが用意されています。

.. code-block:: c

  void convert_to_null_ex(zval **ppzv);
  void convert_to_boolean_ex(zval **ppzv);
  void convert_to_long_ex(zval **ppzv);
  void convert_to_double_ex(zval **ppzv);
  void convert_to_string_ex(zval **ppzv);
  void convert_to_array_ex(zval **ppzv);
  void convert_to_object_ex(zval **ppzv);

これらのマクロは ``zval**`` を受け取り、型の変換の前に ``SEPARATE_ZVAL_IF_NOT_REF()`` を実行するように実装されています。

.. code-block:: c

  #define convert_to_ex_master(ppzv, lower_type, upper_type)  \
      if (Z_TYPE_PP(ppzv)!=IS_##upper_type) {                 \
          SEPARATE_ZVAL_IF_NOT_REF(ppzv);                     \
          convert_to_##lower_type(*ppzv);                     \
      }

その点を除けば、使用方法は通常の ``convert_to_*`` とよく似ています。

.. code-block:: c

  zval **zv_ptr_ptr = /* 関数の引数を取得する */;  

  convert_to_long_ex(zv_ptr_ptr);  

  php_printf("%ld\n", Z_LVAL_PP(zv_ptr_ptr));  

  /* 関数の引数は自動的に破棄されるので明示的に破棄する必要がない */

しかしこれでも常に上手くいくとは限りません。ではよく似た例で値が配列の場合を考えてみましょう。

.. code-block:: c

  zval *array_zv = /* 何らかの方法で配列を取得 */;  

  /* 42の要素を取り出し、zv_destに代入する (ここで行っている処理そのものには意味はありません) */
  zval **zv_dest;
  if (zend_hash_index_find(Z_ARRVAL_P(array_zv), 42, (void **) &zv_dest) == FAILURE) {
      /* エラー: インデックスが見つからない */
      return;
  }  

  convert_to_long_ex(zv_dest);  

  php_printf("%ld\n", Z_LVAL_PP(zv_dest));  

  /* 配列の値は自動的に破棄される */

上記のコードでの ``convert_to_long_ex()`` は関数の外からの配列の値への参照に対しては変更をしないようにしますが、関数の中から参照している配列は変更されます。いくつかのケースではこれは正しい動作となりますが、一般的には配列から値を取得した際に配列を変更することは避けたいことが多いと思います。

そのようなケースでは、zvalの型変換の前にzvalをコピーすることは避けて通れません。

.. code-block:: c

  zval **zv_dest = /* 配列の値を取得 */;
  zval tmp_zv;  

  ZVAL_COPY_VALUE(&tmp_zv, *zv_dest);
  zval_copy_ctor(&tmp_zv);  

  convert_to_long(&tmp_zv);  

  php_printf("%ld\n", Z_LVAL(tmp_zv));  

  zval_dtor(&tmp_zv);

上記のコードの最後の ``zval_dtor()`` の呼び出しは厳密には必要ありません。なぜなら ``tmp_zv`` の型は ``IS_LONG`` になるであろうことが分かっていますし、long型は値を破棄する必要がないからです。string型やarray型など他の型の変換では、zval_dtorの呼び出しは必要になります。

もしコードの中でlong型やdouble型への変換が多い場合には、zvalの変更なしに型変換するヘルパー関数をつくることは意味があるでしょう。long型への型変換のための実装のサンプルは次の通りです。

.. code-block:: c

  long zval_get_long(zval *zv) {
      switch (Z_TYPE_P(zv)) {
          case IS_NULL:
              return 0;
          case IS_BOOL:
          case IS_LONG:
          case IS_RESOURCE:
              return Z_LVAL_P(zv);
          case IS_DOUBLE:
              return zend_dval_to_lval(Z_DVAL_P(zv));
          case IS_STRING:
              return strtol(Z_STRVAL_P(zv), NULL, 10);
          case IS_ARRAY:
              return zend_hash_num_elements(Z_ARRVAL_P(zv)) ? 1 : 0;
          case IS_OBJECT: {
              zval tmp_zv;
              ZVAL_COPY_VALUE(&tmp_zv, zv);
              zval_copy_ctor(&tmp);
              convert_to_long_base(&tmp, 10);
              return Z_LVAL_P(tmp_zv);
          }
      }
  }

上のコードではzvalのコピーをせずに、型変換の結果を直接返しています( ``IS_OBJECT`` の場合にはコピーが避けられないのでこの場合を除く)。この関数を使うことで配列の値を型変換するサンプルコードがより簡潔になります。

.. code-block:: c

  zval **zv_dest = /* 配列の値を取得 */;
  long lval = zval_get_long(*zv_dest);

  php_printf("%ld\n", lval);


上記のようなタイプの関数はPHPの標準関数に、``zend_is_true()`` というものが既に含まれています。この関数の機能はbool型の変換で値を直接返すのと同等です。

.. code-block:: c

  zval *zv_ptr;
  MAKE_STD_ZVAL(zv_ptr);  

  ZVAL_STRING(zv, "", 1);
  php_printf("%d\n", zend_is_true(zv)); // 0
  zval_dtor(zv);  

  ZVAL_STRING(zv, "foobar", 1);
  php_printf("%d\n", zend_is_true(zv)); // 1
  zval_ptr_dtor(&zv);

型変換の際の不要なコピーを避ける別の関数で ``zend_make_printable_zval()`` というものがあります。この関数は ``convert_to_string()`` と同じstringの変換の働きをしますが、別のAPIを使用しています。一般的な使用例は次の通りです。

.. code-block:: c

  zval *zv_ptr = /* zvalを何らかの方法で取得する */;  

  zval tmp_zval;
  int tmp_zval_used;
  zend_make_printable_zval(zv_ptr, &tmp_zval, &tmp_zval_used);  

  if (tmp_zval_used) {
      zv_ptr = &tmp_zval;
  }  

  PHPWRITE(Z_STRVAL_P(zv_ptr), Z_STRLEN_P(zv_ptr));  

  if (tmp_zval_used) {
      zval_dtor(&tmp_zval);
  }

この関数の２番目の引数は一時的なzvalへのポインターで、３番目の引数は整数へのポインターです。もしこの関数が一時的なzvalを使用する場合はその整数に1がセットされ、他の場合には0となります。

``tmp_zval_used`` に基いて、もとのzvalを使用するか一時的なzvalを使うかを決定できます。通例、 ``zv_ptr = &tmp_zval`` とおこなって、一時的なzvalをもとのzvalに割り当てます。これによって、毎回その条件でどちらを使用するかを選択するのではなく、常に ``zv_ptr`` を使うことが可能になります。

最後に ``zval_dtor(&tmp_zval)`` として一時的なzvalを破棄する必要があります。しかしこれは実際に使用されている場合だけです。

型変換に関連する別の関数で ``is_numeric_string()`` というものがあります。この関数は文字列が数字かどうかを確認し、long型かdouble型のどちらかに変換します。

.. code-block:: c

  long lval;
  double dval;  

  switch (is_numeric_string(Z_STRVAL_P(zv_ptr), Z_STRLEN_P(zv_ptr), &lval, &dval, 0)) {
      case IS_LONG:
          /* 文字列は整数と評価され、その値は `lval` に設定される */
          break;
      case IS_DOUBLE:
          /* 文字列は浮動小数点数型として評価され、 その値は `dval` に設定される */
          break;
      default:
          /* 文字列は数値として評価できない */
  }

この関数の最後の引数は ``allow_errors`` と呼ばれるものです。 ``0`` と設定すれば ``123abc`` のような文字列を受け付けませんが、それに対して ``1`` と設定すればエラーとせず受け入れます(その時の値は ``123`` です)。 その中間的な解決策として ``-1`` と指定すると、そのような文字列を受け付けますがnoticeが出力されるようになります。

この関数は ``0xabc`` の形式の16進数の数字もまた受け入れることが出来ることを知っておくと役立つでしょう。この点で、 ``"0xabc"`` に変換する ``convert_to_long()`` や ``convert_to_double()`` とは異なります。

整数と浮動小数点数のどちらも使用し、両方の場合でdoubleを使用する関係で精度を低下させたくない場合には、 ``is_numeric_string`` は特に有効です。このケースを助けるものとして ``convert_scalar_to_number()`` というものがあり、zvalを受け取って配列でない値をlongかdoubleのどちらかに変換(文字列には ``is_numeric_string()`` を使用)します。これは変換されたzvalの型は ``IS_LONG`` か ``IS_DOUBLE`` か ``IS_ARRAY`` となることを意味します。使用方法は ``convert_to_*()`` の関数と同じです。

.. code-block:: c

  zval *zv_ptr;
  MAKE_STD_ZVAL(zv_ptr);
  ZVAL_STRING(zv_ptr, "3.141", 1);  

  convert_scalar_to_number(zv_ptr);
  switch (Z_TYPE_P(zv_ptr)) {
      case IS_LONG:
          php_printf("Long: %ld\n", Z_LVAL_P(zv_ptr));
          break;
      case IS_DOUBLE:
          php_printf("Double: %G\n", Z_DVAL_P(zv_ptr));
          break;
      case IS_ARRAY:
          /* エラーをなげるようにする */
          break;
  }  

  zval_ptr_dtor(&zv_ptr);  

  /* Double: 3.141 */

この関数にも ``convert_scalar_to_number_ex()`` の種類があり、これは ``zval**`` を受け取り、変換の前にzvalを分離します。
