ハッシュテーブルAPI
===================

ハッシュテーブルを扱うための2つのAPIセットがあります。1つはこのセクションで説明する、低レベルの ``zend_hash`` APIです。もう1つは配列のAPIで、一般的な動作をおこなうための高レベルのAPIで、こちらは次のセクションで扱います。

ハッシュテーブルの生成と破棄
------------------------------

ハッシュテーブルは ``ALLOC_HASHTABLE()`` を使用してメモリが割り当てられ、 ``zend_hash_init()`` で初期化されます。

.. code-block:: c

  HashTable *myht;

  /* myht = emalloc(sizeof(HashTable)); と同じ */
  ALLOC_HASHTABLE(myht);

  zend_hash_init(myht, 1000000, NULL, NULL, 0);

``zend_hash_init()`` の２番めの引数はサイズヒントで、これはハッシュテーブルがどれくらい多くの要素を保持すべきかを指定します。 ``1000000`` が渡されると、PHPは最初の挿入時に ``2^20 = 1048576`` の要素分のメモリを確保します。サイズヒントがない場合は、PHPはまず8要素分のメモリを確保し、それからそれ以上の要素が挿入される際に、1度サイズを2倍にリサイズしていきます(最初は16、次が32、その次が64という具合です)。リサイズの度に ``arBuckets`` の再度メモリ割り当てをおこない、ハッシュ値を取得しなおす必要があります(衝突のリストを再計算します)。

サイズヒントを指定するとこのような不要なリサイズ作業が避けられますし、それによりパフォーマンスの向上にもつながります。しかしこれは大きなハッシュテーブルの場合のみ有効で、小さなハッシュテーブルには0を渡すので十分でしょう。特に、8が最低限のテーブルサイズなので、0や2や7を渡してもそれらに違いはないことに注意して下さい。

``zend_hash_init()`` の3番目の引数は常に ``NULL`` であるべきです。というのも、その引数は以前に独自のハッシュ関数を指定するために使用されていましたが、この機能は今はもう使えなくなっているからです。4番目の引数は保持している値を破棄するための関数で、次のような定義になっています。

.. code-block:: c

  typedef void (*dtor_func_t)(void *pDest);

多くの場合、この破棄するための関数は ``ZVAL_PTR_DTOR`` (zvalを格納している場合)になるでしょう。これは単に通常の ``zval_ptr_dtor()`` 関数ですが、``dtor_func_t`` の定義に合うようにしています。

``zend_hash_init()`` の最後の引数は永続的なメモリ割り当てをするかどうかを指定するためのものです。ハッシュテーブルがリクエストが終わっても残り続けるようにするためには、この引数を1にすべきです。初期化の関数は ``zend_hash_init_ex()`` という種類もあり、これは ``bApplyProtection`` という真偽値の引数が追加で渡せます。これに0を設定すると再帰保護機能を無効にできます(デフォルトでは有効になっています)。この関数は滅多に使われないで、大抵はZendEngineの内部構造(関数やクラスなど)で使用されるだけです。

ハッシュテーブルは ``zend_hash_destroy()`` で破棄することができ、 ``FREE_HASHTABLE()`` でメモリ解放できます。

.. code-block:: c

  zend_hash_destroy(myht);

  /* efree(myht); と同じ */
  FREE_HASHTABLE(myht);

``zend_hash_destroy()`` は全てのバケットにおいて破棄のための関数を呼び出し、メモリを解放します。この関数が実行されている間は、ハッシュテーブルは不整合な状態となり使用できません。これは大抵は問題ないですが、いくつかのレアケースで(特に破棄の関数がユーザーランドのコードを呼び出せる場合)、ハッシュテーブルは破棄している間も使用できる状態にしておく必要があるかもしれません。この場合、 ``zend_hash_graceful_destroy()`` や ``zend_hash_graceful_reverse_destroy()`` 関数が有効です。前者はバケットを挿入された順番で破棄していき、後者は逆順に破棄していきます。

ハッシュテーブルの全要素を削除したいが破棄はしたくない場合は ``zend_hash_clean()`` が使用できます。

整数キー
------------

ハッシュテーブルの整数のキーを挿入、取得、削除するための関数をみていく前に、まずそれらの関数がどのような引数を受け取るのか明確にしておきましょう。

実際のデータへのポインターを保持している、バケットの ``pData`` メンバーを思い出してください。ハッシュテーブルに ``zval *`` の値を格納している場合は、 ``pData`` は ``zval **`` となります。そのため、ハッシュテーブルへの挿入では、データ型として ``zval *`` を指定したとしても、 ``zval **`` を渡さなければいけません。

ハッシュテーブルから値を取り出す際には、``pData`` の書き込み先となるポインターの ``pDest`` を渡します。 ``*pDest = pData`` としてポインターに書き込むために、ポインターの無駄な参照がさらにもう1つ必要になるのです。 そのため、``zval *`` がデータ型なら、値の取得するための関数には ``zval ****`` を渡さないといけません。

これが具体的にどのようになるかの例として、整数のキーを挿入したり更新したりすることができる ``zend_hash_index_update()`` 関数についてみていきましょう。

.. code-block:: c

  HashTable *myht;
  zval *zv;  

  ALLOC_HASHTABLE(myht);
  zend_hash_init(myht, 0, NULL, ZVAL_PTR_DTOR, 0);  

  MAKE_STD_ZVAL(zv);
  ZVAL_STRING(zv, "foo", 1);  

  /* 下記はPHPのコードで表現すると $array[42] = "foo" となる */
  zend_hash_index_update(myht, 42, &zv, sizeof(zval *), NULL);  

  zend_hash_destroy(myht);
  FREE_HASHTABLE(myht);

上の例では、``42`` のキーに ``foo`` という文字列の ``zval *`` を挿入しています。4番目の引数には使用しているデータ型を指定して ``sizeof(zval *)`` とします。そのため、挿入する値である3番目の引数は ``zval **`` の型である必要があります。

最後の引数は、値を挿入し同時にその値を再び取り出すために使用されます。

.. code-block:: c

  zval **zv_dest

  zend_hash_index_update(myht, 42, &zv, sizeof(zval *), (void **) &zv_dest);

なぜこのような処理が必要になるのでしょう？挿入した値は既に知っているはずなのに、なぜ再び取得しなすのでしょうか？ハッシュテーブルは常に渡された値のコピーを扱っていることを思い出して下さい。つまり、 ``zval *`` がハッシュテーブルで保持されている限りは ``zv`` と同じで、異なるメモリ空間に格納されています。参照渡しのハッシュテーブルの値を変更するためには、そのための新しい場所である、書き込み先の ``zv_dest`` のアドレスが必要になります。

``zval *`` の値を保持している際には、この関数の最後の引数は滅多に必要となりません。一方で、ポインター型でない値を使用している場合に非常によく見られるパターンは、まず一時的な構造体がつくられ、その後ハッシュテーブルに挿入され、ポインターが参照しているその値がその後の全ての処理で使用されるというものです(その一時的な構造体を変更してもハッシュテーブルの値には何も影響しないため)。

どのインデックスにも値は入れたくないが、ハッシュテーブルの要素数のサイズを大きくしたい事はよくあるでしょう。これは ``zend_hash_next_index_insert()`` の関数を使うことで可能です。

.. code-block:: c

  if (zend_hash_next_index_insert(myht, &zv, sizeof(zval *), NULL) == SUCCESS) {
      Z_ADDREF_P(zv);
  }

この関数は次に使用できる整数のキーに ``zv`` を挿入しています。つまり、使用されている一番大きいの整数のキーが ``42`` であれば、新しい値は ``43`` のキーに対して挿入されます。 ``zend_hash_index_update()`` とは違って、この関数は失敗を返す場合があるので、返り値が ``SUCCESS`` / ``FAILURE`` なのかをチェックする必要があります。

どのような場合に失敗を返すのか、次の例を見てみましょう。

.. code-block:: c

  zend_hash_index_update(myht, LONG_MAX, &zv, sizeof(zval *), NULL);  

  php_printf("Next \"free\" key: %ld\n", zend_hash_next_free_element(myht));
  if (zend_hash_next_index_insert(myht, &zv, sizeof(zval *), NULL) == FAILURE) {
      php_printf("next_index_insert failed\n");
  }
  php_printf("Number of elements in hashtable: %ld\n", zend_hash_num_elements(myht));


このコードの出力は下記の通りです。

.. code-block:: c

  Next "free" key: 2147483647 [or 9223372036854775807 on 64 bit]
  next_index_insert failed
  Number of elements in hashtable: 1

一体何が起きたのでしょう？最初の値は ``LONG_MAX`` のキーで挿入されています。この場合、次の整数のキーは ``LONG_MAX + 1`` となり、オーバーフローして ``LONG_MIN`` となります。この動作は望ましくないので、PHPはこの特殊なケースをチェックし、 ``LONG_MAX`` の ``nNextFreeElement`` を ``LONG_MAX+1`` とせずにそのままにしておきます。その状態で ``zend_hash_next_index_insert()`` が実行されると、 ``LONG_MAX`` のキーに対して値を挿入することを試みますが、そのキーは既に使用されているので、処理は失敗となります。

上の最後のコードでは2つの関数の紹介もしています。1つは、次に使用できる整数のキー(既にご覧のように、実際には必ずしも使用できなくてもよいです)を返す関数と、もう1つはハッシュテーブルの要素数を返す関数です。特に ``zend_hash_num_elements()`` はかなり頻繁に使用されます。

これまでの知識をもってすれば、整数のキーにまつわる残りの3つのAPIはかなり分かりやすいでしょう。 ``zend_hash_index_find()`` はインデックスの値を取得し、 ``zend_hash_index_exists()`` は値を取り出さずにインデックスが存在するかチェックし、 ``zend_hash_index_del()`` は要素を削除します。下記はこの3つの関数の例です。

.. code-block:: c

  zval **zv_dest;  

  if (zend_hash_index_exists(myht, 42)) {
      php_printf("Index 42 exists\n");
  } else {
      php_printf("Index 42 doesn't exist\n");
  }  

  if (zend_hash_index_find(myht, 42, (void **) &zv_dest) == SUCCESS) {
      php_printf("Fetched value of index 42 into zv_dest\n");
  } else {
      php_printf("Couldn't fetch value of index 42 as it doesn't exist :(\n");
  }  

  if (zend_hash_index_del(myht, 42) == SUCCESS) {
      php_printf("Removed value at index 42\n");
  } else {
      php_printf("Couldn't remove value at index 42 as it doesn't exist :(\n");
  }

``zend_hash_index_exists()`` はインデックスが存在していると1を返し、そうでなければ0を返します。 ``find`` と ``del`` の関数は、値が存在すれば ``SUCCESS`` を返し、そうでなければ ``FAILURE`` を返します。


文字列キー
-----------

文字列キーの扱いは整数キーと非常によく似ています。主な違いは全ての関数名に ``index`` という語が使用されていない点です。勿論、それらの関数はパラメーターとして、インデックスの代わりに文字列とその長さを受け取ります。

唯一警戒すべき点は、この文脈で"文字列の長さ"が何を意味するかです。ハッシュテーブルのAPIでは"文字列の長さ"には **ヌル終端文字も含みます** 。この点で、その ``zend_hash`` のAPIは、文字列の長さにヌル終端文字を含まない他の殆ど全てのZendAPIと異なります。

実際のところ、これは何を意味するのでしょう。引数に文字列リテラルを渡した場合、文字列の長さは ``sizeof("foo")-1`` よりもむしろ ``sizeof("foo")`` となります。zvalから文字列を渡した場合は、文字列の長さは ``Z_STRVAL_P(zv)`` ではなく ``Z_STRVAL_P(zv)+1`` となります。

この点を除けば、これらの関数はindexの関数と全く同じ方法で使用できます。

.. code-block:: c

  HashTable *myht;
  zval *zv;
  zval **zv_dest;  

  ALLOC_HASHTABLE(myht);
  zend_hash_init(myht, 0, NULL, ZVAL_PTR_DTOR, 0);  

  MAKE_STD_ZVAL(zv);
  ZVAL_STRING(zv, "bar", 1);  

  /* 下記はPHPのコードで表現すると $array["foo"] = "bar" となる */
  zend_hash_update(myht, "foo", sizeof("foo"), &zv, sizeof(zval *), NULL);  

  if (zend_hash_exists(myht, "foo", sizeof("foo"))) {
      php_printf("Key \"foo\" exists\n");
  }  

  if (zend_hash_find(myht, "foo", sizeof("foo"), (void **) &zv_dest) == SUCCESS) {
      php_printf("Fetched value at key \"foo\" into zv_dest\n");
  }  

  if (zend_hash_del(myht, "foo", sizeof("foo")) == SUCCESS) {
      php_printf("Removed value at key \"foo\"\n");
  }  

  if (!zend_hash_exists(myht, "foo", sizeof("foo"))) {
      php_printf("Key \"foo\" no longer exists\n");
  }  

  if (zend_hash_find(myht, "foo", sizeof("foo"), (void **) &zv_dest) == FAILURE) {
      php_printf("As key \"foo\" no longer exists, zend_hash_find returns FAILURE\n");
  }  

  zend_hash_destroy(myht);
  FREE_HASHTABLE(myht);

上のスニペットは次のように出力します。

.. code-block:: c

  Key "foo" exists
  Fetched value at key "foo" into zv_dest
  Removed value at key "foo"
  Key "foo" no longer exists
  As key "foo" no longer exists, zend_hash_find returns FAILURE

文字列キーで挿入するための関数として ``zend_hash_update()`` の他に、 ``zend_hash_add()`` という関数があります。両者の関数の違いは既にキーが存在していた場合の挙動です。 ``zend_hash_update()`` は値を上書きするのに対して、 ``zend_hash_add()`` は代わりに ``FAILURE`` を返します。

下記が ``zend_hash_update()`` がキーを上書く際の振る舞いです。

.. code-block:: c

  zval *zv1, *zv2;
  zval **zv_dest;  

  /* ... zval の初期化 */  

  zend_hash_update(myht, "foo", sizeof("foo"), &zv1, sizeof(zval *), NULL);
  zend_hash_update(myht, "foo", sizeof("foo"), &zv2, sizeof(zval *), NULL);  

  if (zend_hash_find(myht, "foo", sizeof("foo"), (void **) &zv_dest) == SUCCESS) {
      if (*zv_dest == zv1) {
          php_printf("Key \"foo\" contains zv1\n");
      }
      if (*zv_dest == zv2) {
          php_printf("Key \"foo\" contains zv2\n");
      }
  }

上記のコードは ``Key "foo" contains zv2`` を出力します。つまり、値が上書きされています。では ``zend_hash_add()`` と比べてみましょう。

.. code-block:: c

  zval *zv1, *zv2;
  zval **zv_dest;  

  /* ... zvalの初期化 */  

  if (zend_hash_add(myht, "bar", sizeof("bar"), &zv1, sizeof(zval *), NULL) == FAILURE) {
      zval_ptr_dtor(&zv1);
  } else {
      php_printf("zend_hash_add returned SUCCESS as key \"bar\" was unused\n");
  }  

  if (zend_hash_add(myht, "bar", sizeof("bar"), &zv2, sizeof(zval *), NULL) == FAILURE) {
      zval_ptr_dtor(&zv2);
      php_printf("zend_hash_add returned FAILURE as key \"bar\" is already taken\n");
  }  

  if (zend_hash_find(myht, "bar", sizeof("bar"), (void **) &zv_dest) == SUCCESS) {
      if (*zv_dest == zv1) {
          php_printf("Key \"bar\" contains zv1\n");
      }
      if (*zv_dest == zv2) {
          php_printf("Key \"bar\" contains zv2\n");
      }
  }

上のコードは次のように出力します。

.. code-block:: c

  zend_hash_add returned SUCCESS as key "bar" was unused
  zend_hash_add returned FAILURE as key "bar" is already taken
  Key "bar" contains zv1


``zend_hash_add()`` の2回目の呼び出しは ``FAILURE`` を返し、値は ``zv1`` のままです。

文字列キーのための `` zend_hash_add()`` 関数がありますが、整数キーには同等の関数がないことに注意が必要です。このような類の振る舞いが必要な場合は、 ``exists`` の関数をまず呼び出さないといけないか、下記のように低レベルのAPIを利用しないといけないかのどちらかになります。

.. code-block:: c

  _zend_hash_index_update_or_next_insert(
      myht, 42, &zv, sizeof(zval *), NULL, HASH_ADD ZEND_FILE_LINE_CC
  )

これまでの全ての関数には関数名に ``quick`` がつく種類があり、これは文字列の長さの引数の後に事前に計算済みのハッシュ値を受け取ります。これにより、文字列のハッシュを一度計算すれば、その後それを複数の呼び出しの間で使い回すことができます。

.. code-block:: c

  ulong h; /* hash値 */  

  /* ... zvalの初期化 */  

  h = zend_get_hash_value("foo", sizeof("foo"));  

  zend_hash_quick_update(myht, "foo", sizeof("foo"), h, &zv, sizeof(zval *), NULL);  

  if (zend_hash_quick_find(myht, "foo", sizeof("foo"), h, (void **) &zv_dest) == SUCCESS) {
      php_printf("Fetched value at key \"foo\" into zv_dest\n");
  }  

  if (zend_hash_quick_del(myht, "foo", sizeof("foo"), h) == SUCCESS) {
      php_printf("Removed value at key \"foo\"\n");
  }

``quick`` のAPIを使うことで、関数の呼び出しの度にハッシュを再計算しなくてよいので、パフォーマンスが向上します。これは多くのキーにアクセスする際(例えばループなど)には顕著になります。 ``quick`` 関数は主に、様々なキャッシュや最適化を通して、計算済みのハッシュ値が使用できるZendEngineで使用されます。


Apply関数
----------

ハッシュテーブルの特定のキーに対してではなく、全ての値を扱いたいことが多いでしょう。PHPはこのために、2つの仕組みを用意しています。1つは、 ``zend_hash_apply_*()`` ファミリーで、これはハッシュテーブルの全ての要素において指定した関数を呼び出していきます。この関数には3種類あります。

.. code-block:: c

  void zend_hash_apply(HashTable *ht, apply_func_t apply_func TSRMLS_DC);
  void zend_hash_apply_with_argument(
      HashTable *ht, apply_func_arg_t apply_func, void *argument TSRMLS_DC
  );
  void zend_hash_apply_with_arguments(
      HashTable *ht TSRMLS_DC, apply_func_args_t apply_func, int num_args, ...
  );

基本的にこれら3つの関数は同じ事をおこないますが、 ``apply_func`` に渡す引数の数が異なります。下記がそれぞれの ``apply_func`` の定義です。

.. code-block:: c

  typedef int (*apply_func_t)(void *pDest TSRMLS_DC);
  typedef int (*apply_func_arg_t)(void *pDest, void *argument TSRMLS_DC);
  typedef int (*apply_func_args_t)(
      void *pDest TSRMLS_DC, int num_args, va_list args, zend_hash_key *hash_key
  );

ご覧のように、 ``zend_hash_apply()`` はコールバック関数に何も引数を渡せません。 ``zend_hash_apply_with_argument`` は1つ引数が渡せ、 ``zend_hash_apply_with_arguments()`` は任意の数の引数(これは ``va_list args`` で表されています)を渡すことができます。さらに、最後の関数では ``void *pDest`` だけでなく、それに対応する ``hash_key`` も渡すことができます。 ``zend_hash_key`` の構造は下記の通りです。

.. code-block:: c

  typedef struct _zend_hash_key {
      const char *arKey;
      uint nKeyLength;
      ulong h;
  } zend_hash_key;

これらのメンバーは ``Bucket`` のメンバーと同じ意味をもっています。 ``nKeyLength == 0`` の場合は、 ``h`` は整数キーです。そうでなければ、 ``h`` は ``nKeyLength`` の長さの文字列の ``arKey`` のハッシュとなります。

これらの関数の使用例として、 ``var_dump`` とよく似た配列をdumpする関数を実装してみましょう。 ``zend_hash_apply_with_arguments()`` を使用しますが、これは多くの引数を渡したいからではなく、配列が必要だからです。ではまずdumpをおこなうメインの関数から見ていきましょう。

.. code-block:: c

  static void dump_value(zval *zv, int depth) {
      if (Z_TYPE_P(zv) == IS_ARRAY) {
          php_printf("%*carray(%d) {\n", depth * 2, ' ', zend_hash_num_elements(Z_ARRVAL_P(zv)));
          zend_hash_apply_with_arguments(Z_ARRVAL_P(zv), dump_array_values, 1, depth + 1);
          php_printf("%*c}\n", depth * 2, ' ');
      } else {
          php_printf("%*c%Z\n", depth * 2, ' ', zv);
      }
  }  

  PHP_FUNCTION(dump_array) {
      zval *array;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "a", &array) == FAILURE) {
          return;
      }  

      dump_value(array, 0);
  }

上の例では、 ``php_printf()`` のあまり一般的でないオプションをいくつか使っています。 ``%*c`` はある文字を複数回繰り返すためのものです。つまり、 ``php_printf("%*c", depth * 2, ' ')`` はホワイトスペースを ``depth * 2`` 回繰り返し、これはdepthが増える度に毎回ホワイトスペース2つのインデントをしているという事を表現しています。 ``%Z`` はzvalを文字列に変換して出力します。

このように、上のコードは ``%Z`` を使って直接値を出力していますが、配列は特別な処理をしています。つまり、配列の要素がdumpされ、その外側に ``array(n) { ... }`` と出力されています。ここでapply関数が登場します。

.. code-block:: c

  zend_hash_apply_with_arguments(Z_ARRVAL_P(zv), dump_array_values, 1, depth + 1);

``dump_array_values`` はコールバック関数で、全ての要素に対して呼び出されます。 ``1`` は関数に渡す引数の数で、``depth + 1`` がその1つだけの引数になります。下記がその関数です。

.. code-block:: c

  static int dump_array_values(
      void *pDest TSRMLS_DC, int num_args, va_list args, zend_hash_key *hash_key
  ) {
      zval **zv = (zval **) pDest;
      int depth = va_arg(args, int);  

      if (hash_key->nKeyLength == 0) {
          php_printf("%*c[%ld]=>\n", depth * 2, ' ', hash_key->h);
      } else {
          php_printf("%*c[\"", depth * 2, ' ');
          PHPWRITE(hash_key->arKey, hash_key->nKeyLength - 1);
          php_printf("\"]=>\n");
      }  

      dump_value(*zv, depth);  

      return ZEND_HASH_APPLY_KEEP;
  }

``depth = va_arg(args, int)`` として、渡された ``depth`` を受け取っています。同じようにして、他のそれ以外の引数も受け取る事ができます。それ以降のコードではキーの出力のフォーマットを整えたり、再帰的に ``dump_value`` を呼び出して値を出力しています。

さらに、この関数は ``ZEND_HASH_APPLY_KEEP`` を返していますが、これはapply関数に対しての有効な4種類の返り値のうちの1つです。

``ZEND_HASH_APPLY_KEEP`` 
  今走査している要素をそのままにしておき、ハッシュテーブルの走査を続ける  

``ZEND_HASH_APPLY_REMOVE``
  今走査している要素を取り除き、ハッシュテーブルの走査を続ける

``ZEND_HASH_APPLY_STOP``
  今走査している要素をそのままにしておき、ハッシュテーブルの走査を中止する

``ZEND_HASH_APPLY_REMOVE | ZEND_HASH_APPLY_STOP``
  今走査している要素を取り除き、ハッシュテーブルの走査を中止する。

このように ``zend_hash_apply_*()`` は簡単な ``array_map()`` のように動作しますが、 ``array_filter()`` のようでもあり、イテレーションの任意のタイミングで中断するという追加的な機能もあります。

ではこのdump関数を試してみましょう。

.. code-block:: c

  dump_array([1, [2, "foo" => 3]]);
  // 出力:
  array(2) {
    [0]=>
    1
    [1]=>
    array(2) {
      [0]=>
      2
      ["foo"]=>
      3
    }
  }


結果は ``var_dump`` の出力と非常によく似ています。 ``php_var_dump()`` の関数をみてみると、実装のために同じ方法が使われていることが分かるでしょう。

イテレーション
----------------

ハッシュテーブルの全ての値を取り扱う2つ目の方法は、ハッシュテーブルをイテレートするということです。C言語でのハッシュテーブルのイテレーションはPHPでの配列のイテレーションのやり方と非常によく似ています。

.. code-block:: php

  <?php  

  for (reset($array);
       null !== $data = current($array);
       next($array)
  ) {
      // $data を使って何らかの処理を行う
  }


上のコードでのループ処理はC言語だと次のようになります。

.. code-block:: c

  zval **data;  

  for (zend_hash_internal_pointer_reset(myht);
       zend_hash_get_current_data(myht, (void **) &data) == SUCCESS;
       zend_hash_move_forward(myht)
  ) {
      /* data を使って何らかの処理を行う */
  }

上のコードスニペットでは内部的な配列のポインター( ``pInternalPointer`` )を利用していますが、これは大抵は悪いアイデアです。というのも、このポインターはハッシュテーブルの一部であり、そのためハッシュテーブルを使う全てのコードで共有されます。例えば、ハッシュテーブルのネストされたイテレーションはその内部的なポインターを使う場合は不可能となります(1つのループが他のループで使用しているポインターを変更してしまうからです)。

そのため、全てのイテレーションの関数は ``_ex`` で終わる名前の種類のものがあり、これは外部ポインターを扱っています。このAPIを使う際、現在のハッシュテーブルの位置は ``HashPosition`` (これは単に ``Bucket *`` の型となっています)に保持されており、この構造体へのポインターは全ての関数の最後の引数として渡されます。

.. code-block:: c

  HashPosition pos;
  zval **data;  

  for (zend_hash_internal_pointer_reset_ex(myht, &pos);
       zend_hash_get_current_data_ex(myht, (void **) &data, &pos) == SUCCESS;
       zend_hash_move_forward_ex(myht, &pos)
  ) {
      /* data を使って何らかの処理を行う */
  }

``reset`` の代わりに ``end`` の関数を、また ``move_forward`` の代わりに ``move_backwards`` の関数を使う事で、反対方向へのイテレーションも可能です。

.. code-block:: c

  HashPosition pos;
  zval **data;  

  for (zend_hash_internal_pointer_end_ex(myht, &pos);
       zend_hash_get_current_data_ex(myht, (void **) &data, &pos) == SUCCESS;
       zend_hash_move_backwards_ex(myht, &pos)
  ) {
      /* data を使って何らかの処理を行う */
  }

下記のように定義されている ``zend_hash_get_current_key_ex()`` を使うことで、キーを取得することができます。

.. code-block:: c

  int zend_hash_get_current_key_ex(
      const HashTable *ht, char **str_index, uint *str_length,
      ulong *num_index, zend_bool duplicate, HashPosition *pos
  );


この関数の返り値はキーの型を表し、値の種類は次の通りです。

``HASH_KEY_IS_LONG``
  キーは整数で、 ``num_index`` に書き込まれます。

``HASH_KEY_IS_STRING``
  キーは文字列で、 ``str_index`` に書き込まれます。 ``duplicate`` パラメーターは直接書き込むのか、コピーを書き込むかを指定するためのものです。最後に、文字列の長さ(前にあったように、NUL byteを含みます)は ``str_length.`` に代入されます。

``HASH_KEY_NON_EXISTANT``
  これはハッシュテーブルの最後までイテレートして、他に要素がないことを意味します。上で使用されているループでは、このケースは発生しないでしょう。

異なる返り値を判断するために、この関数は一般的に ``switch`` 文で使われる事が多いです。

.. code-block:: c

  char *str_index;
  uint str_length;
  ulong num_index;  

  switch (zend_hash_get_current_key_ex(myht, &str_index, &str_length, &num_index, 0, &pos)) {
      case HASH_KEY_IS_LONG:
          php_printf("%ld", num_index);
          break;
      case HASH_KEY_IS_STRING:
          /* NUL byte を含むハッシュテーブルの長さから1を減算する */
          PHPWRITE(str_index, str_length - 1);
          break;
  }

PHP 5.5では、追加で ``zend_hash_get_current_key_zval_ex()`` という関数があり、これはキーをzvalに書き込みたい場合に使用します。

.. code-block:: c

  zval *key;
  MAKE_STD_ZVAL(key);
  zend_hash_get_current_key_zval_ex(myht, key, &pos);

コピーとマージ
----------------

とても一般的な操作としてハッシュテーブルのコピーがあります。これは手動でする必要はほとんどないですが、PHPでは配列のコピーオンライトが起きる時には必ずコピーする必要があります。コピーは ``zend_hash_copy()`` の関数を使っておこないます。

.. code-block:: c

  HashTable *ht_source = get_ht_from_somewhere();
  HashTable *ht_target;  

  ALLOC_HASHTABLE(ht_target);
  zend_hash_init(ht_target, zend_hash_num_elements(ht_source), NULL, ZVAL_PTR_DTOR, 0);
  zend_hash_copy(ht_target, ht_source, (copy_ctor_func_t) zval_add_ref, NULL, sizeof(zval *));

``zend_hash_copy()`` の4番目の引数は今は使われていないので、常に ``NULL`` としなければなりません。3番目の引数は要素をコピーする度に呼び出されるコピーコンストラクタの関数を指定します。zvalの場合、この関数は ``zval_add_ref`` となり、単に全ての要素に参照を追加します。

``zend_hash_copy()`` はコピー先のハッシュテーブルに既に要素があっても動作します。``ht_source`` の要素のキーが ``ht_target`` に既に存在している場合は、上書きされます。この振る舞いを制御するために、 ``zend_hash_merge()`` の関数が使うことができます。この関数は ``zend_hash_copy()`` と同じシグネチャですが、その上書きをするかしないかを指定する引数ももっています。

``zend_hash_merge(..., 0)`` はコピー先のハッシュテーブルに存在しない要素のみコピーします。一方、 ``zend_hash_merge(..., 1)`` は ``zend_hash_copy()`` の呼び出しとほとんど同じ動作となります。唯一の違いは、 ``merge`` の方では、内部的な配列のポインターを最初の要素( ``pListHead`` )に設定しますが、 ``copy`` の方ではコピー元のハッシュテーブルと同じ要素にポインターを設定します。

マージの動作において、よりきめ細かい制御をするためには、 ``zend_hash_merge_ex`` 関数が有効で、これはコピーすべき要素をチェック関数を使用することで選択することができます。

.. code-block:: c

  typedef zend_bool (*merge_checker_func_t)(
      HashTable *target_ht, void *source_data, zend_hash_key *hash_key, void *pParam
  );

このチェック関数はコピー先のハッシュテーブル、コピー元のデータ、そのデータのハッシュキー、追加のパラメーター( ``zend_hash_apply_with_argument()`` と同じです)を受け取ります。例として、2つの配列を受け取り、それらをマージし、もしキーが衝突した場合は大きい方の値を使用するという関数を実装してみましょう。

.. code-block:: c

  static int merge_greater(
      HashTable *target_ht, zval **source_zv, zend_hash_key *hash_key, void *dummy
  ) {
      zval **target_zv;
      zval compare_result;  

      if (zend_hash_quick_find(
              target_ht, hash_key->arKey, hash_key->nKeyLength, hash_key->h, (void **) &target_zv
          ) == FAILURE
      ) {
          /* キーがコピー先のハッシュテーブルに存在しないため必ずコピー */
          return 1;
      }  

      /* コピー元のzvalよりもコピー先のzvalの方が大きい場合 (compare == 1) のみコピー */
      compare_function(&compare_result, *source_zv, *target_zv);
      return Z_LVAL(compare_result) == 1;
  }  

  PHP_FUNCTION(array_merge_greater) {
      zval *array1, *array2;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "aa", &array1, &array2) == FAILURE) {
          return;
      }  

      /* array1をreturn_valueにコピー */
      RETVAL_ZVAL(array1, 1, 0);  

      zend_hash_merge_ex(
          Z_ARRVAL_P(return_value), Z_ARRVAL_P(array2), (copy_ctor_func_t) zval_add_ref,
          sizeof(zval *), (merge_checker_func_t) merge_greater, NULL
      );
  }

メインの関数では、まず ``array1`` をreturn_valueにコピーして、その後 ``array2`` とマージしています。その後、チェック関数の ``merge_greater()`` が2番目の要素の全てに対して呼び出されます。まず、1番目の配列から同じキーの要素の取得を試みます。もし要素が取得できなければ、2番目の配列の要素は常にコピーします。要素が存在すれば、2番目の配列の要素の値が1番目の要素の値よりも大きい場合のみコピーします。

ではこの新しい関数を試してみましょう。 ::

  var_dump(array_merge_greater(
      [3 => 0, "bar" => -5],
      ["bar" => 5, "foo" => -10, 3 => -42]
  ));
  // 出力:
  array(3) {
    [3]=>
    int(0)
    ["bar"]=>
    int(5)
    ["foo"]=>
    int(-10)
  }


比較、ソート、極値
----------------------

ハッシュテーブルのAPIで最後の3つは全て、何らかの方法によるハッシュテーブルの要素の比較に関連するものです。比較関数は次のように定義されています。

.. code-block:: c

  typedef int (*compare_func_t)(const void *left, const void *right TSRMLS_DC);

この関数は2つのハッシュテーブルの要素を受け取り、そのお互いの関係を返します。返り値が負の場合は ``left < right`` 、正の場合は ``left > right`` 、0の場合は両者の値が等しいことを意味します。

まず初めにみていく関数は ``zend_hash_compare()`` で、これは2つのハッシュテーブルを比較します。

.. code-block:: c

  int zend_hash_compare(
      HashTable *ht1, HashTable *ht2, compare_func_t compar, zend_bool ordered TSRMLS_DC
  );

返り値の意味は ``compare_func_t`` と同じです。この関数はまず初めに、配列の長さを比較します。2の配列の長さが異なる場合、長い方の配列を大きいと見なします。配列の長さが同じの場合の挙動は ``ordered`` のパラメーターによって決まります。

``ordered=0`` (順序を考慮にいれない)の場合、この関数は1番目のハッシュテーブルのバケットを1つ1つ見ていき、それぞれ2番目のハッシュテーブルに同じキーの要素が存在するかどうか探索します。2番目の配列に存在しなければ、1番目の配列の方が大きいと見なします。2番目の配列に存在すれば、それらの値で ``compar`` 関数が呼び出されます。

``ordered=1`` (順序を考慮にいれる)の場合、両方のハッシュテーブルを同時に見ていきます。それぞれの要素で、まずキーを比較して、同じであれば ``compar`` を使って値を比較します。

このの処理は比較の結果で0以外の値が返ってくるか(この場合は比較の結果の返り値が ``zend_hash_compare()`` の返り値となります)、あるいは比較すべき要素がなくなるまで続けられます。後者の場合は、2つのハッシュテーブルは等しいと見なされます。

これら2つの比較方法はPHPの2つの等価演算子のそれぞれの振る舞いに対応しています。

.. code-block:: c

  /* $ar1 == $ar2 は要素を == で比較し、順序を考慮にいれない */
  zend_hash_compare(ht1, ht2, (compare_func_t) hash_zval_compare_function, 0 TSRMLS_CC);  

  /* $ar1 === $ar2 は要素を === で比較し、 順序を考慮にいれる */
  zend_hash_compare(ht1, ht2, (compare_func_t) hash_zval_identical_function, 1 TSRMLS_CC);

次にみていくのは ``zend_hash_sort()`` という関数で、これはハッシュテーブルのソートで使用されます。

.. code-block:: c

  int zend_hash_sort(HashTable *ht, sort_func_t sort_func, compare_func_t compar, int renumber TSRMLS_DC);

この関数はハッシュテーブルの前処理や後処理をおこなうだけで、実際のソート処理は ``sort_func`` に委譲しています。

.. code-block:: c

  typedef void (*sort_func_t)(
      void *buckets, size_t num_of_buckets, register size_t size_of_bucket,
      compare_func_t compare_func TSRMLS_DC
  );

この関数は比較関数は勿論、バケットの配列、バケットの数、バケットのサイズ(常に ``sizeof(Bucket *)`` です)を受け取ります。ここでの「バケットの配列」とは通常のC言語の配列のことを指しており、ハッシュテーブルのことではありません。ソート関数は配列のバケットを見ていき、そうすることで新しい順序を指定します。

ソート関数の処理が終わると、 ``zend_hash_sort()`` はそのC言語の配列からハッシュテーブルを再構築します。 ``renumber=0`` の場合、値のそれぞれのキーは維持され、単に順序が変わるだけです。 ``renumber=1`` ではキーが振り直されるので、結果としてのハッシュテーブルのキーは0から順に増えていくかたちとなっています。

自分自身でソートアルゴリズムを実装したいのでない限り、ソート関数は ``zend_qsort`` を指定すべきで、これはPHPで定義済みのクイックソートの実装です。

最後の比較に関連する関数はハッシュテーブルの中の要素のうちの最小値や最大値を探すために使用されます。

.. code-block:: c

  int zend_hash_minmax(
      const HashTable *ht, compare_func_t compar, int flag, void **pData TSRMLS_DC
  );

``flag=0`` では最小値が、 ``flag=1`` では最大値が ``pData`` に書き込まれます。ハッシュテーブルが空の場合には、この関数は ``FAILURE`` (空の配列には最小値と最大値は明確に定義されていないため)を返します。
