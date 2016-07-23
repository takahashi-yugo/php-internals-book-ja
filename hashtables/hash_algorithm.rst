ハッシュアルゴリズムと衝突
===========================

ハッシュテーブルの最後のセクションでは、最悪なケースな衝突シナリオについてと、PHPが採用しているハッシュ関数のいくつかの性質についてより詳しくみていきましょう。この知識はハッシュテーブルのAPIを使うにあたって必要ないものではありますが、ハッシュテーブルの構造とその限界へのより深い理解へと繋がるでしょう。


衝突の解析
-----------

衝突の解析を簡潔にするために、 ``array_collision_info()`` という配列を受け取ってどのキーがどのインデックスで衝突しているかについての情報を教えてくれるヘルパー関数をはじめに書いてみましょう。そのためには、 ``arBuckets`` を順にみていき、全てのインデックスに対して、そのインデックスにおける全てのバケットについてのいくつかの情報を含む配列を作成します。

.. code-block:: c

  PHP_FUNCTION(array_collision_info) {
      HashTable *hash;
      zend_uint i;  

      if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_DC, "h", &hash) == FAILURE) {
          return;
      }  

      array_init(return_value);  

      /* 空のハッシュテーブルはまだ初期化されていないかもしれない */
      if (hash->nNumOfElements == 0) {
          return;
      }  

      for (i = 0; i < hash->nTableSize; ++i) {
          /* この nIndex に要素を作成する */
          zval *elements;
          Bucket *bucket;  

          MAKE_STD_ZVAL(elements);
          array_init(elements);
          add_next_index_zval(return_value, elements);  

          bucket = hash->arBuckets[i];
          while (bucket != NULL) {
              zval *element;  

              MAKE_STD_ZVAL(element);
              array_init(element);
              add_next_index_zval(elements, element);  

              add_assoc_long(element, "hash", bucket->h);  

              if (bucket->nKeyLength == 0) {
                  add_assoc_long(element, "key", bucket->h);
              } else {
                  add_assoc_stringl(
                      element, "key", (char *) bucket->arKey, bucket->nKeyLength - 1, 1
                  );
              }  

              {
                  zval **data = (zval **) bucket->pData;
                  Z_ADDREF_PP(data);
                  add_assoc_zval(element, "value", *data);
              }  

              bucket = bucket->pNext;
          }
      }
  }

このコードは前回のセクションでの ``add_`` 関数の良い使用例でもあります。ではこの関数を試してみましょう。

.. code-block:: c

  var_dump(array_collision_info([2 => 0, 5 => 1, 10 => 2]));  

  // Output (reformatted a bit):  

  array(8) {
    [0] => array(0) {}
    [1] => array(0) {}
    [2] => array(2) {
      [0] => array(3) {
        ["hash"]  => int(10)
        ["key"]   => int(10)
        ["value"] => int(2)
      }
      [1] => array(3) {
        ["hash"]  => int(2)
        ["key"]   => int(2)
        ["value"] => int(0)
      }
    }
    [3] => array(0) {}
    [4] => array(0) {}
    [5] => array(1) {
      [0] => array(3) {
        ["hash"]  => int(5)
        ["key"]   => int(5)
        ["value"] => int(1)
      }
    }
    [6] => array(0) {}
    [7] => array(0) {}
  }


この出力結果からいくつかの事が分かります(既に認識済みのものがほとんどでしょう)。

- 3つの要素しか挿入していないのにも関わらず、外側の配列は8の要素を持っています。これは8がデフォルトでのハッシュテーブルの初期サイズだからです。
- 整数の場合には、ハッシュとキーは常に同じです。
- それぞれのハッシュは全て異なっているのにも関わらず、 ``nIndex == 2`` で衝突がおきており、これは 2 % 8 が2で10 % 8もまた2だからです。
- 衝突解決した連結リストには挿入順とは逆の順序で要素が格納されています(これが実装するのに最も簡単な方法だからです)。

インデックス衝突
----------------

現状の目的は、全てのハッシュキーが衝突するという最悪の衝突シナリオをつくることです。これを達成するには2つの方法がありますが、まずより簡単な方法から始めて見ましょう。その方法とは、ハッシュ関数で衝突を発生させるのではなくむしろ、インデックス(これはハッシュ値のハッシュテーブルのサイズにおける剰余です)で衝突を発生させるというものです。

整数のキーの場合には、これは特に簡単です。なぜなら整数のキーには、何のハッシュ演算も適用されないからです。インデックスは単に ``key % nTableSize`` となります。この式から衝突を見つけることは些細なことです。例えば、テーブルサイズの倍数であればどんなキーも衝突します。テーブルサイズが8であれば、インデックスは0 % 8 = 0、 8 % 8 = 0、 16 % 8 = 0、 24 % 8 = 0のようになります。

このシナリオをPHPのスクリプトで実演してみましょう。

.. code-block:: php

  <?php  

  $size = pow(2, 16); // 2の何乗でもかまわない 

  $startTime = microtime(true);  

  //  [0, $size, 2 * $size, 3 * $size, ..., ($size - 1) * $size]  のキーを挿入していく

  $array = array();
  for ($key = 0, $maxKey = ($size - 1) * $size; $key <= $maxKey; $key += $size) {
      $array[$key] = 0;
  }  

  $endTime = microtime(true);  

  printf("Inserted %d elements in %.2f seconds\n", $size, $endTime - $startTime);
  printf("There are %d collisions at index 0\n", count(array_collision_info($array)[0]));

このコードの出力は次の通りです(結果は使用するマシーンによって異なるでしょうが、同じ規模となるでしょう)。 ::

  Inserted 65536 elements in 34.05 seconds
  There are 65536 collisions at index 0

勿論、わずかな量の要素を挿入するのに30秒掛かるのは大変遅いです。何が起きたのでしょう。全てのハッシュキーが衝突するシナリオをつくったために、挿入のパフォーマンスがO(1) から O(n)へと悪化したのです。挿入の度に、PHPは既に同じキーの要素が存在するかどうかを確認するために、そのインデックスの衝突リストをひとつひとつみていかなければなりません。大抵はこれは衝突のリストが1つや2つのバケットのみを含んでいるだけなので問題とはなりません。一方でパフォーマンスが悪化するケースでは、全ての要素がこの衝突リストに入っています。

そのため、PHPはn回の挿入をO(n)の計算量で実行しなければならず、これは全体の計算量がO(n^2)となります。つまり、2^16回の処理をおこなうどころか、約2^32回の処理を行わなければなりません。

ハッシュ衝突
---------------

さて、インデックスの衝突を使って最悪のケースであるシナリオを作ることに成功しましたので、次は同じことをハッシュ衝突を使ってやってみましょう。これは整数キーを使ってはできないので、次の通りに定義されているPHPの文字列のハッシュ関数をみる必要があります。

.. code-block:: c

  static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
  {
      register ulong hash = 5381;  

      /* variant with the hash unrolled eight times */
      for (; nKeyLength >= 8; nKeyLength -= 8) {
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
          hash = ((hash << 5) + hash) + *arKey++;
      }
      switch (nKeyLength) {
          case 7: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
          case 6: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
          case 5: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
          case 4: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
          case 3: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
          case 2: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
          case 1: hash = ((hash << 5) + hash) + *arKey++; break;
          case 0: break;
          EMPTY_SWITCH_DEFAULT_CASE()
      }
      return hash;
  }


手動ループ展開を取り除くと、関数は次のようになります。

.. code-block:: c

  static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
  {
      register ulong hash = 5381;  

      for (uint i = 0; i < nKeyLength; ++i) {
          hash = ((hash << 5) + hash) + arKey[i];
      }  

      return hash;
  }

``hash << 5 + hash`` の式は ``hash * 32 + hash`` か、単に ``hash * 33`` と同じ意味です。これでより関数を簡潔にできます。

.. code-block:: c

  static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
  {
      register ulong hash = 5381;  

      for (uint i = 0; i < nKeyLength; ++i) {
          hash = hash * 33 + arKey[i];
      }  

      return hash;
  }

このハッシュ関数はDJBX33Aと呼ばれるもので、"Daniel J. Bernstein, Times 33 with Addition"表しています。これは文字列ハッシュ関数の中で最もシンプル(そして最も高速)なものの1つです。

ハッシュ関数がシンプルなおかげで、衝突をみつけることは難しくありません。2文字の衝突から始めてみましょう。例えば、同じハッシュを持つ ``ab`` と ``cd`` をみていきましょう。 ::

      hash(ab) = hash(cd)
  <=> (5381 * 33 + a) * 33 + b = (5381 * 33 + c) * 33 + d
  <=> a * 33 + b = c * 33 + d
  <=> c = a + n
      d = b - 33 * n
      where n is an integer

これにより、2文字の文字列を受け取って、1文字目を1つインクリメントして、2文字目を33デクリメントすることで衝突が得られることが分かります。このテクニックを使って、8つの文字列全てが衝突するグループを作ることが出来ます。衝突がおこるグループの一例は次の通りです。

.. code-block:: php

  <?php
  $array = [
      "E" . chr(122)  => 0,
      "F" . chr(89)   => 1,
      "G" . chr(56)   => 2,
      "H" . chr(23)   => 3,
      "I" . chr(-10)  => 4,
      "J" . chr(-43)  => 5,
      "K" . chr(-76)  => 6,
      "L" . chr(-109) => 7,
  ];  

  var_dump(array_collision_info($array));

この出力は完全に全てのキーが ``193456164`` というハッシュで衝突していることを示しています。

.. code-block:: php

  array(8) {
    [0] => array(0) {}
    [1] => array(0) {}
    [2] => array(0) {}
    [3] => array(0) {}
    [4] => array(8) {
      [0] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "L\x93"
        ["value"] => int(7)
      }
      [1] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "K´"
        ["value"] => int(6)
      }
      [2] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "JÕ"
        ["value"] => int(5)
      }
      [3] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "Iö"
        ["value"] => int(4)
      }
      [4] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "H\x17"
        ["value"] => int(3)
      }
      [5] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "G8"
        ["value"] => int(2)
      }
      [6] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "FY"
        ["value"] => int(1)
      }
      [7] => array(3) {
        ["hash"]  => int(193456164)
        ["key"]   => string(2) "Ez"
        ["value"] => int(0)
      }
    }
    [5] => array(0) {}
    [6] => array(0) {}
    [7] => array(0) {}
  }

一旦ひとつの衝突グループを作ってしまえば、さらなる衝突をつくることはずっと簡単になります。そのためには、DJBX33Aの次の性質を利用します。つまり、もし同じ長さの文字列の ``$str1`` と ``$str2`` が衝突する場合、 ``$prefix.$str1.$postfix`` と ``$prefix.$str2.$postfix`` も同様に衝突します。これが完全に真であると証明することは簡単です。 ::

    hash(prefix . str1 . postfix)
  = hash(prefix) * 33^a + hash(str1) * 33^b + hash(postfix)
  = hash(prefix) * 33^a + hash(str2) * 33^b + hash(postfix)
  = hash(prefix . str2 . postfix)  

    where a = strlen(str1 . postfix) and b = strlen(postfix)

つまり、 ``Ez`` と ``FY`` が衝突すれば、 ``abcEzefg`` と ``abcFYefg`` も衝突します。これは、異なるハッシュではあるけども、衝突が発生してしまうという前の考察において、ハッシュの一部でもある末尾のヌルバイトを無視することができた理由でもあります。

この性質を利用して、既知の衝突セットを使い、それらをあの手この手で連結させることで、大きな衝突セットを生成することができます。例えば、 ``Ez`` と ``FY`` が衝突すると分かっていれば、 ``EzEzEz`` 、 ``EzEzFY`` 、 ``EzFYEz`` 、 ``EzFYFY`` 、 ``FYEzEz`` 、 ``FYEzFY`` 、 ``FYFYEz`` 、 ``FYFYFY`` の全てが衝突するということが分かります。このやり方によって、任意の衝突セットを作ることが出来ます。

