Symtableと配列API
===================

ハッシュテーブルのAPIによってどんな型の値でも扱うことができますが、圧倒的に多くのケースでは値はzvalとなります。 ``zend_hash`` APIをzvalで使うと、zvalのメモリ割り当てや初期化を自身でしなければならないので、少し扱いにくいということが多いでしょう。このために、PHPでは特にこのユースケースを対象とした別のAPIのセットを用意しています。この簡素化されたAPIを紹介する前に、PHPの配列が利用しているハッシュテーブルの特殊な例を見ていくことにしましょう。

Symtable
---------

PHPの言語設計の背後にある核となる概念のうちの1つは、整数と整数を含む文字列は置き換え可能であるべきだということです。これは配列のキーにも当てはまり、 ``42`` と ``"42"`` のキーは同じものと考えられるべきです。しかしこれは、通常のハッシュテーブルには当てはまりません。ハッシュテーブルでは厳密にキーの型が区別され、 ``42`` と ``"42"`` のキーの両方を同じハッシュテーブルで(それぞれ違う値を)持つことは可能です。

このために、別でsymtable(シンボル表) APIというものがあり、これはハッシュテーブルの関数の薄いラッパー関数で、整数の文字列を整数のキーへと変換します。例えば、 ``zend_symtable_find()`` は次のように定義されています。

.. code-block:: c

  static inline int zend_symtable_find(
      HashTable *ht, const char *arKey, uint nKeyLength, void **pData
  ) {
      ZEND_HANDLE_NUMERIC(arKey, nKeyLength, zend_hash_index_find(ht, idx, pData));
      return zend_hash_find(ht, arKey, nKeyLength, pData);
  }

``ZEND_HANDLE_NUMERIC()`` マクロの実装はここでは詳しくふれませんが、その背後にある機能が大事にすぎません。 ``arKey`` に ``LONG_MIN`` と ``LONG_MAX`` の間の十進数の整数を含む場合、その整数が ``idx`` に書き込まれ、それと共に ``zend_hash_index_find()`` が呼ばれます。それ以外の全ての場合は、処理は次の ``zend_hash_find()`` が呼び出されている行へと進みます。

``zend_symtable_find()`` 以外には、下記の関数がsymtable APIの一部で、それぞれハッシュテーブルのAPIと対応しているものと同じ振る舞いですが、それに文字列の整数への正規化の処理が含まれています。

.. code-block:: c

  static inline int zend_symtable_exists(HashTable *ht, const char *arKey, uint nKeyLength);
  static inline int zend_symtable_del(HashTable *ht, const char *arKey, uint nKeyLength);
  static inline int zend_symtable_update(
      HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest
  );
  static inline int zend_symtable_update_current_key_ex(
      HashTable *ht, const char *arKey, uint nKeyLength, int mode, HashPosition *pos
  );

加えて、symtableを生成するために2つのマクロがあります。

.. code-block:: c

  #define ZEND_INIT_SYMTABLE_EX(ht, n, persistent) \
      zend_hash_init(ht, n, NULL, ZVAL_PTR_DTOR, persistent)  

  #define ZEND_INIT_SYMTABLE(ht) \
      ZEND_INIT_SYMTABLE_EX(ht, 2, 0)

ご覧のように、これらのマクロは ``ZVAL_PTR_DTOR`` をデストラクタとして使って ``zend_hash_init()`` を単に呼び出しているだけです。そのため、これらのマクロは、上で述べた、文字列から整数への型変換する振る舞いには直接関係はありません。

では新しい関数を試してみましょう。

.. code-block:: c

  HashTable *myht;
  zval *zv1, *zv2;
  zval **zv_dest;  

  ALLOC_HASHTABLE(myht);
  ZEND_INIT_SYMTABLE(myht);  

  MAKE_STD_ZVAL(zv1);
  ZVAL_STRING(zv1, "zv1", 1);  

  MAKE_STD_ZVAL(zv2);
  ZVAL_STRING(zv2, "zv2", 1);  

  zend_hash_index_update(myht, 42, &zv1, sizeof(zval *), NULL);
  zend_symtable_update(myht, "42", sizeof("42"), &zv2, sizeof(zval *), NULL);  

  if (zend_hash_index_find(myht, 42, (void **) &zv_dest) == SUCCESS) {
      php_printf("Value at key 42 is %Z\n", *zv_dest);
  }  

  if (zend_symtable_find(myht, "42", sizeof("42"), (void **) &zv_dest) == SUCCESS) {
      php_printf("Value at key \"42\" is %Z\n", *zv_dest);
  }  

  zend_hash_destroy(myht);
  FREE_HASHTABLE(myht);

このコードの出力は下記の通りです。 ::

  Value at key 42 is zv2
  Value at key "42" is zv2

つまり、両方の ``update`` 関数は同じ要素に対して書き込みをおこない(2番目の呼び出しでは1番目のを上書いています)、両方の ``find`` 関数もまた同じ要素を取得しています。



配列API
---------

さて、配列APIをみていくために必要な予備知識は全て準備できました。このAPIはハッシュテーブルに対して直接動作するのではなく、むしろ ``Z_ARRVAL_P()`` を使用してハッシュテーブルから取り出されたzvalを引数として受け取ります。

配列APIのうちの最初の2つの関数は ``array_init()`` と ``array_init_size()`` で、ハッシュテーブルを初期化してzvalを設定します。前者は対象のzvalのみを受け取る一方で、後者は追加としてサイズヒントを受け取ります。

.. code-block:: c

  /* return_valueを空の配列にする */
  array_init(return_value);  

  /* return_valueを空の配列として、サイズヒンとして1000000を設定する */
  array_init_size(return_value, 1000000);

このAPIの残りの関数は全て、配列に値を挿入することを扱っています。関数には次の通り、4つのグループがあります。

.. code-block:: c

  /* 次のインデックスに挿入する */
  int add_next_index_*(zval *arg, ...);
  /* 指定したインデックスに挿入する */
  int add_index_*(zval *arg, ulong idx, ...);
  /* 指定したキーに挿入する */
  int add_assoc_*(zval *arg, const char *key, ...);
  /* key_lenの長さの指定したキーに挿入する (バイナリセーフ) */
  int add_assoc_*_ex(zval *arg, const char *key, uint key_len, ...);

上の ``*`` は型のプレースホルダーで、 ``...`` は型特有の引数のプレースホルダーです。プレースホルダーの有効な値は下記の表の通りです。

============  ===============================================
型            追加の引数
============  ===============================================
``null``      なし
``bool``      ``int b``
``long``      ``long n``
``double``    ``double d``
``string``    ``const char *str, int duplicate``
``stringl``   ``const char *str, uint length, int duplicate``
``resource``  ``int r``
``zval``      ``zval *value``
============  ===============================================

これらの関数の使用例として、様々な型の要素をもったダミーの配列をつくってみましょう。

.. code-block:: c

  PHP_FUNCTION(make_array) {
      zval *zv;  

      array_init(return_value);  

      add_index_long(return_value, 10, 100);
      add_index_double(return_value, 20, 3.141);
      add_index_string(return_value, 30, "foo", 1);  

      add_next_index_bool(return_value, 1);
      add_next_index_stringl(return_value, "\0bar", sizeof("\0bar")-1, 1);  

      add_assoc_null(return_value, "foo");
      add_assoc_long(return_value, "bar", 42);  

      add_assoc_double_ex(return_value, "\0bar", sizeof("\0bar"), 1.61);  

      /* zvalを手動で生成するために、必要な処理を行います */
      MAKE_STD_ZVAL(zv);
      object_init(zv);
      add_next_index_zval(return_value, zv);
  }

この配列の ``var_dump()`` の出力は次の通りです(NUL-bytesは ``\0`` で置き換えています)。

.. code-block:: c

  array(9) {
    [10]=>
    int(100)
    [20]=>
    float(3.141)
    [30]=>
    string(3) "foo"
    [31]=>
    bool(true)
    [32]=>
    string(4) "\0bar"
    ["foo"]=>
    NULL
    ["bar"]=>
    int(42)
    ["\0bar"]=>
    float(1.61)
    [33]=>
    object(stdClass)#1 (0) {
    }
  }

上記のコードをみてみると、配列APIは文字列の長さという点に関して全く一貫性がないことに気づくかもしれません。というのも、 ``_ex`` 関数に渡されたキーの長さはヌル文字を含む一方で、 ``stringl`` 関数に渡される文字列の長さはヌル文字を含んでいません。

さらに、これらの関数は ``add`` で始まってはいますが、既に存在しているキーを上書きする点においては ``update`` 関数のようなに振る舞うことにも注意すべきです。

これらとは別にいくつかの ``add_get`` という関数があり、これは値を挿入してそれを再び取得します( ``zend_hash_update`` 関数の最後の引数と似ています)。これらの関数はほとんど使われる事がないのでここではふれませんが、完全をきたすために言及しているに過ぎません。

これでハッシュテーブル、symtable、配列のAPIのそれぞれの説明を終わります。
