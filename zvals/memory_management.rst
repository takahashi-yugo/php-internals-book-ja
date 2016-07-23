メモリ管理
===========

zvalの構造体には2つの役割があります。1つ目は前のセクションで説明したように、値と型を保持するということです。このセクションで説明する2つ目の役割は、値の効率的なメモリー管理です。

以下では、参照カウントとコピーオンラインのコンセプトについて、そしてクステンションでのコードの中でどのようにそれらを利用していくかを見ていきたいと思います。

値セマンティクスと参照セマンティクス
---------------------------------------

PHPでは全ての値が、明示的に参照を使わない限り、常に値のセマンティクスをもっています。これは関数に値渡しで渡そうが、ある変数を別の変数に対して代入しようが、それぞれ値の別々のコピーが作成されるということを意味します。いくつかの例で実際に確認してみましょう。

.. code-block:: php

  <?php  

  $a = 1;
  $b = $a;
  $a++;  

  // $aのみがインクリメントされており, $bはそのまま:
  var_dump($a, $b); // int(2), int(1)  

  function inc($n) {
      $n++;
  }  

  $c = 1;
  inc($c);  

  // 関数の外側の$cの値と関数の内側の$nは別物
  var_dump($c); // int(1)

上記の例で十分明らかではありますが、このルールが常に当てはまることを認識することは重要です。特にオブジェクトでも同様です。

.. code-block:: php

  <?php  

  $obj = (object) ['value' => 1];  

  function fnByVal($val) {
      $val = 100;
  }  

  function fnByRef(&$ref) {
      $ref = 100;
  }  

  // 値渡しの関数では$objを変更しないが、参照渡しだと変更される:  

  fnByVal($obj);
  var_dump($obj); // stdClass(value => 1)
  fnByRef($obj);
  var_dump($obj); // int(100)

PHP5からは自動的にオブジェクトが参照渡しになっているということがよく言われていますが、上記の例のようにそれは間違いです。値渡しの関数では引数で受け取った変数の値の書き換えは出来ず、参照渡しでのみ出来ます。

しかしながら、オブジェクトが"参照渡しのような"振る舞いをおこなうことは事実です。引数で与えられたオブジェクト型の変数の全く異なる別のコピーが作成されるわけではないのですが、そのオブジェクトのプロパティを変更することはできるようになっています。これはオブジェクトの値というのは、単にオブジェクトの"実際の中身"を参照するためのIDにすぎないためです。値セマンティクスはこのオブジェクトのIDを変更して別のオブジェクトにしたり完全に型を変更したりということは禁止しますが、オブジェクトの"実際の中身"を変更することは禁止はしません。

これはリソース型にも同様に当てはまります。なぜならオブジェクト型もまた、実際の中身を参照するためのIDを保持しているだけだからです。なので値セマンティクスによりリソースIDを変更したりzvalの型をすることは出来ませんが、リソースの中身(例えばファイル内の位置を進めたりといったようなこと)を変更することは可能です。

参照カウントとコピーオンライト
-------------------------------

上記のことを少し考えてみると、PHPは大変な量をコピーする必要があるのではないかと思われるかもしれません。関数に引数を渡す度に、その値をコピーする必要があるからです。これはとりわけintegerやdoubleであれば問題とはならないでしょうが、1000万の要素をもった配列を関数に渡す場合を想像してみてください。何百万もの要素を関数が呼ばれる度にコピーしていると手がつけられないほどに処理が遅くなってしまいます。

これを避けるため、PHPではコピーオンライトという方法論を採用しています。つまりzvalの値が読まれるだけで変更されない限りは、複数の変数や関数などの間で共有されるのです。もしそのzvalを利用しているうちのいづれかが値を変更したければ、変更を加える前にzvalをコピーしなければなりません。

ひとつのzvalが複数の場所で共有されうる場合には、そのzvalを破棄して解放するために、PHPは何らかの方法によってzvalがどこからも利用されていない状況を把握する必要があります。PHPではこれを単純にzvalが参照されている回数を把握し続けることで可能にしています。ここでの"参照"はPHPの参照( ``&`` で表現されるもの)とは何も関係なく、zvalを利用しているもの(変数や関数など)についてのことを意味していることに注意してください。参照されている数はrefcountと呼ばれ、zvalの ``refcount__gc`` のメンバーで保持されています。

このことを理解するためにある例を考えてみましょう。

.. code-block:: php

  <?php  

  $a = 1;    // $a =           zval_1(value=1, refcount=1)
  $b = $a;   // $a = $b =      zval_1(value=1, refcount=2)
  $c = $b;   // $a = $b = $c = zval_1(value=1, refcount=3)  

  $a++;      // $b = $c = zval_1(value=1, refcount=2)
             // $a =      zval_2(value=2, refcount=1)  

  unset($b); // $c = zval_1(value=1, refcount=1)
             // $a = zval_2(value=2, refcount=1)  

  unset($c); // refcount=0 なのでzval_1 は破棄される
             // $a = zval_2(value=2, refcount=1)


動作は非常に簡単です。参照が増えるとrefcountもインクリメントされ、参照が減るとデクリメントされます。refcountが0になると、zvalが破棄されます。

この方法が上手くいかなくなるのは循環参照のケースです。

.. code-block:: php

  <?php  

  $a = []; // $a = zval_1(value=[], refcount=1)
  $b = []; // $b = zval_2(value=[], refcount=1)  

  $a[0] = $b; // $a = zval_1(value=[0 => zval_2], refcount=1)
              // $b = zval_2(value=[], refcount=2)
              // zval_2がzval_1の配列の中で使用されているので、
              // zval_2のrefcountがインクリメントされる

  $b[0] = $a; // $a = zval_1(value=[0 => zval_2], refcount=2)
              // $b = zval_2(value=[0 => zval_1], refcount=2)
              // zval_1がzval_2の配列の中で使用されているので、
              // zval_1のrefcountがインクリメントされる

  unset($a);  //      zval_1(value=[0 => zval_2], refcount=1)
              // $b = zval_2(value=[0 => zval_1], refcount=2)
              // zval_1のrefcountはデクリメントされるが、
              // zval_2からまだ参照されているのでzvalは破棄されない  

  unset($b);  //      zval_1(value=[0 => zval_2], refcount=1)
              //      zval_2(value=[0 => zval_1], refcount=1)
              // zval_2のrefcountはデクリメントされるが、
              // zval_1からまだ参照されているのでzvalは破棄されない 

上記のコードが実行すると、どの変数からも利用されないzvalが2つ出来上がり、お互い参照しているので破棄されないまま残り続けてしまうという状況となります。これは参照カウントの失敗としての古典的な例です。

この問題に取り掛かるためにPHPではガーベッジコレクションの第二の仕組みであるサイクルコレクターを利用します。今のところこの仕組みを無視しても(参照カウントのそれとは違って)、エクステンションを書く際にはほとんど透過的となっていますので無視しても問題ありません。このトピックについてもう少し詳細に学びたい方は、PHPマニュアルにある `そのアルゴリズムについての簡単な説明 <http://php.net/manual/ja/features.gc.collecting-cycles.php>`_ を参照してください。

別のケースとして、実際のPHPの参照(上で述べたPHP内部で使用されているzvalの参照ではなく ``&$var`` としての参照)についても考慮されなくてはなりません。zvalがPHPの参照を利用していることを示すために、is_refという真偽値のフラグを利用しており、これはzval構造体の ``is_ref__gc`` メンバーで保持されています。

zvalで ``is_ref=1`` のフラグの場合、変更の前にzvalをコピーすべき **でない** ことを表しています。代わりに、次のように値を直接変更します。

.. code-block:: php

  <?php  

  $a = 1;   // $a =      zval_1(value=1, refcount=1, is_ref=0)
  $b =& $a; // $a = $b = zval_1(value=1, refcount=2, is_ref=1)  

  $b++;     // $a = $b = zval_1(value=2, refcount=2, is_ref=1)
            // is_ref=1となっているため、 PHPはzvalのコピーを作るのではなく
            // zvalを直接変更する

上の例では参照がつくられる前のzvalのrefcountは1です。では同じような例で、参照しているzvalのrefcountが1よりも大きい場合にどうなるか考えてみましょう。

.. code-block:: php 

  <?php  

  $a = 1;   // $a =           zval_1(value=1, refcount=1, is_ref=0)
  $b = $a;  // $a = $b =      zval_1(value=1, refcount=2, is_ref=0)
  $c = $b   // $a = $b = $c = zval_1(value=1, refcount=3, is_ref=0)  

  $d =& $c; // $a = $b = zval_1(value=1, refcount=2, is_ref=0)
            // $c = $d = zval_2(value=1, refcount=2, is_ref=1)
            // $dは、$aや$b **ではなく** $cの参照となるので、
            // zvalのコピーが必要になる。 ここでは、同じ内容のzvalが出来上がり、
            // ひとつはis_ref=0、ひとつはis_ref=1となっている

  $d++;     // $a = $b = zval_1(value=1, refcount=2, is_ref=0)
            // $c = $d = zval_2(value=2, refcount=2, is_ref=1)
            // 両者は別々のzvalなので、$d++としても
            // (予想通り)$aと$bは変更されない

ご覧のように、``&`` で参照しているzvalがis_refが0かつrefcount > 1の場合、コピーが必要になるのです。同様に、値渡しでis_ref=1かつrefcount > 1のzvalを使用しようとするとコピーが必要になります。この理由から、PHPの参照を利用すると大抵処理が遅くなります。というのもPHPのほぼ全ての関数が値渡しであるため、is_ref=1なzvalを引数で渡すとたいていコピーを引き起こすことになるからです。

zvalのメモリ割り当てと初期化
-----------------------------

さて、zvalのメモリ管理の背後にある一般的なコンセプトは理解されたと思います。ではそれらの実際の実装をみていきましょう。まずはzvalのメモリ割り当てです。

.. code-block:: c

  zval *zv_ptr;
  ALLOC_ZVAL(zv_ptr);

このコードスニペットはメモリ割り当てをしていますが、zvalのメンバーの初期化はしていません。リクエストの終了時でも破棄されない永続的なzvalにするためのメモリを割り当てのマクロもあります。

.. code-block:: c

  zval *zv_ptr;
  ALLOC_PERMANENT_ZVAL(zv_ptr);

この２つのマクロの違いは前者が ``emalloc()`` を使っていて、後者が ``malloc()`` を使っている点です。もっとも、まずはマクロを使わず直接zvalのメモリを割り当てようとしても上手く動作しないであろうことを知っておくことが重要です。

.. code-block:: c

  /* このコードは間違い */
  zval *zv_ptr = emalloc(sizeof(zval));

上手く動作しない理由としては、サイクルコレクターがzval上のいくつか情報を必要としていて、メモリ割り当てが必要な構造体は、実際には ``zval`` ではなく ``zval_gc_info`` だからです。

.. code-block:: c

  typedef struct _zval_gc_info {
      zval z;
      union {
          gc_root_buffer       *buffered;
          struct _zval_gc_info *next;
      } u;
  } zval_gc_info;


``ALLOC_*`` マクロは ``zval_gc_info`` のメモリ割り当てをおこないそのメンバーを初期化しますが、それによってzvalが使用できるようになります(なぜなら構造体の最初のメンバーに ``zval`` が含まれているからです)。

zvalのメモリが割り当てられたら、次は初期化が必要になります。そのためのマクロは２つ用意されています。ひとつが ``INIT_PZVAL`` で、refcount=1かつis_ref=0で初期化しますが値は初期化されないままです。

.. code-block:: c

  zval *zv_ptr;
  ALLOC_ZVAL(zv_ptr);
  INIT_PZVAL(zv_ptr);
  /* ここではzv_ptrのtypeとvalueは不定の値となっている */

もうひとつのマクロは ``INIT_ZVAL`` で、refcount=1かつis_ref=0で初期化し、それに加え、型が ``IS_NULL`` で設定されます。

.. code-block:: c

  zval *zv_ptr;
  ALLOC_ZVAL(zv_ptr);
  INIT_ZVAL(*zv_ptr);
  /* zv_ptrのtypeはIS_NULLとなる */


``INIT_PZVAL()`` は ``zval*`` を引数で受け取り(だから名前に ``P`` が含まれています)、一方 ``INIT_ZVAL()`` は ``zval`` を受け取ります。後者のマクロに ``zval*`` を渡す際、参照先の値をまず取得する必要があります。

zvalのメモリ割り当てと初期化を一度にすることは非常に一般的なことなので、両方のステップをおこなってくれるマクロが２つあります。

.. code-block:: c

  zval *zv_ptr;
  MAKE_STD_ZVAL(zv_ptr);
  /* ここではzv_ptrのtypeとvalueは不定の値となっている */

  zval *zv_ptr;
  ALLOC_INIT_ZVAL(zv_ptr);
  /* zv_ptrのtypeはIS_NULLとなる */

``MAKE_STD_ZVAL()`` は ``INIT_PZVAL()`` でのメモリ割り当てで、 ``ALLOC_INIT_ZVAL()`` は ``INIT_ZVAL()`` でのメモリ割り当てをおこないます。


refcountの管理とzvalの破棄
----------------------------

zvalのメモリ割り当てと初期化が終わると、上で紹介した参照カウントの仕組みが利用できるようになります。refcountを管理するために、PHPにはいくつかのマクロが用意されています。

.. code-block:: c

  Z_REFCOUNT_P(zv_ptr)      /* refcountを取得 */
  Z_ADDREF_P(zv_ptr)        /* refcountをインクリメント */
  Z_DELREF_P(zv_ptr)        /* refcountをデクリメント */
  Z_SET_REFCOUNT(zv_ptr, 1) /* refcountに特定の数を設定する (ここでは1) */

他の ``Z_`` のマクロのように、 ``_P`` や ``_PP`` の接尾辞がつく種類があり、それぞれ ``zval`` 、 ``zval*`` 、 ``zval**`` を受け取るようになっています。

よく使うことになるであろうマクロは ``Z_ADDREF_P()`` です。簡単な例を見てみましょう。

.. code-block:: c

  zval *zv_ptr;
  MAKE_STD_ZVAL(zv_ptr);
  ZVAL_LONG(zv_ptr, 42);  

  add_index_zval(some_array, 0, zv_ptr);
  add_assoc_zval(some_array, "num", zv_ptr);
  Z_ADDREF_P(zv_ptr);


このコードでは、42の整数を配列の0番目の要素に追加しており、そのzvalは２箇所で使われています。 ``MAKE_STD_ZVAL()`` によってのメモリ割り当てと初期化の後、zvalのrefcountは1ではじまります。同じzvalを２箇所で使うということはrefcountは2である必要があるので、 ``Z_ADDREF_P()`` でrefcountをインクリメントしなければならないのです。

一方、その補足的なマクロである ``Z_DELREF_P()`` は滅多に使われることはありません。というのも、``refcount==0`` の時にzvalを破棄してメモリ解放するというケースをチェックする必要があるので、単にrefcountをデクリメントするだけでは十分ではないのです。

.. code-block:: c

  Z_DELREF_P(zv_ptr);
  if (Z_REFCOUNT_P(zv_ptr) == 0) {
      zval_dtor(zv_ptr);
      efree(zv_ptr);
  }

``zval_dtor()`` マクロは ``zval*`` を受け取り、その値を破棄します。値が文字列ならその文字列を解放し、配列であればHashTableが破棄して解放します。オブジェクトやリソースであれば実際の値のrefcountがデクリメントされます(それによってrefcount=0となれば、破棄と解放されるでしょう)。

自身でrefcountのチェックをしている上記のコードを書き変えて、 ``zval_ptr_dtor()`` と呼ばれる２つ目のマクロを使うべきです。

.. code-block:: c

  zval_ptr_dtor(&zv_ptr);

このマクロは ``zval**`` を受け取り(歴史的な理由から、 ``zval*`` も同様に受け取れます)、refcountをデクリメントしてzvalの破棄と解放が必要かどうかチェックします。しかし上で書いた手動でのチェックとは違って、サークルコレクションもサポートしています。下記がその実装のうちの関連部分です。

.. code-block:: c 

  static zend_always_inline void i_zval_ptr_dtor(zval *zval_ptr ZEND_FILE_LINE_DC TSRMLS_DC)
  {
      if (!Z_DELREF_P(zval_ptr)) {
          ZEND_ASSERT(zval_ptr != &EG(uninitialized_zval));
          GC_REMOVE_ZVAL_FROM_BUFFER(zval_ptr);
          zval_dtor(zval_ptr);
          efree_rel(zval_ptr);
      } else {
          if (Z_REFCOUNT_P(zval_ptr) == 1) {
              Z_UNSET_ISREF_P(zval_ptr);
          }  

          GC_ZVAL_CHECK_POSSIBLE_ROOT(zval_ptr);
      }
  }

``Z_DELREF_P()`` はデクリメント後の新しいrefcountを返すので、 ``!Z_DELREF_P(zval_ptr)`` と書くことは ``Z_DELREF_P(zval_ptr)`` してから ``Z_REFCOUNT_P(zval_ptr) == 0`` かどうかのチェックをすることと同じ意味です。

``zval_dtor()`` と ``efree()`` をすることとは別に、サイクルコレクションを制御するための２つの ``GC_*`` マクロを呼び出し、また ``&EG(uninitialized_zval)`` が決して解放されないように条件付けしています(これはZendEngineによって使われている定義済みのzvalです)。

さらに、コードではzvalの参照されている数が1であればis_ref=0と設定しています。この場合、is_ref=1のままにしておくことは意味をなしません。なぜならPHPの参照 ``&`` は複数の間でzvalが共有されてはじめて意味をもつからです。

これらのマクロの使用法における秘訣がいくつかあります。 ``Z_DELREF_P()`` は決して使ってはいけません(使っていいとすれば、zvalを破棄する必要がなく、ガベージサイクルの可能性があるルートでないという保証がある状況でのみです)。refcountをデクリメントしたい時はいつでも、代わりに ``zval_ptr_dtor()`` を使用すべきです。 ``zval_dtor()`` マクロは一般的にスタックに割り当てられた一時的なzvalに使います。

.. code-block:: c 

  zval zv;
  INIT_ZVAL(zv);

  /* zvを使って何らかの処理を行う */

  zval_dtor(&zv);

スタック上に割り当てられた一時的なzvalはそのブロックが終わると解放されるので共有できません。そのため、refcountは使うことはなく、無差別に ``zval_dtor()`` を使って破棄することが出来ます。


zvalのコピー
-------------

コピーオンライトの仕組みによって多くのzvalのコピーを節約できますが、zvalの値を変更したい時や別の場所に移動したい時など、いくつかの点ではコピーする必要がでてきます。

PHPには様々なケースのための数多くのコピー用のマクロが用意されています。最も単純なもののひとつは ``ZVAL_COPY_VALUE()`` で、単にzvalの ``value`` と ``type`` メンバーをコピーするだけです。

.. code-block:: c

  zval *zv_src;
  MAKE_STD_ZVAL(zv_src);
  ZVAL_STRING(zv_src, "test", 1);  

  zval *zv_dest;
  ALLOC_ZVAL(zv_dest);
  ZVAL_COPY_VALUE(zv_dest, zv_src);

この段階では ``zv_dest`` は ``zv_src`` と同じ型と値を持っているでしょう。ここでの"同じ値"というのは、両方のzvalの値が同じ文字列( ``char*`` )であるということです。例えば、 ``zv_src`` のzvalが破棄されると、文字列が解放され、 ``zv_dest`` は解放された文字列へとぶら下がり続けるポインターをもったzvalとなってしまうでしょう。これを避けるために、zvalのコピーコンストラクタである ``zval_copy_ctor()`` を呼びださなければなりません。

.. code-block:: c 

  zval *zv_dest;
  ALLOC_ZVAL(zv_dest);
  ZVAL_COPY_VALUE(zv_dest, zv_src);
  zval_copy_ctor(zv_dest);

``zval_copy_ctor()`` はzvalの値を完全にコピーします。もし値が文字列なら、その ``char*`` がコピーされ、配列なら ``HashTable*`` がコピーされます。オブジェクトやリソースであれば内部で使用している参照カウントがインクリメントされます。

抜けているものとしてはrefcountとis_refの初期化です。これは ``ALLOC_ZVAL()`` の代わりに ``INIT_PZVAL()`` マクロや ``MAKE_STD_ZVAL()``  マクロを使うことでおこなわれます。別の方法としては、 ``ZVAL_COPY_VALUE()`` の代わりに、コピーに加えてrefcount/is_refの初期化が一緒になった ``INIT_PZVAL_COPY()`` を使います。

.. code-block:: c

  zval *zv_dest;
  ALLOC_ZVAL(zv_dest);
  INIT_PZVAL_COPY(zv_dest, zv_src);
  zval_copy_ctor(zv_dest);

``INIT_PZVAL_COPY()`` と ``zval_copy_ctor()`` をあわせておこなうことは一般的であるので、それらを一緒にした ``MAKE_COPY_ZVAL()`` マクロがあります。

.. code-block:: c 

  zval *zv_dest;
  ALLOC_ZVAL(zv_dest);
  MAKE_COPY_ZVAL(&zv_src, zv_dest);

このマクロは少々変わった定義となっています。なぜなら引数の順番がさきほどと入れ替わっていますし(コピー先のzvalの引数が２番目となっています)、またコピー元となるzvalの方は ``zval**`` でなければならないのです。これもまた歴史的な産物にすぎず、技術的な意味は何もありません。

これらの基本的なコピー用のマクロとは別に、より複雑なマクロも用意されています。最も重要なのは ``ZVAL_ZVAL`` で、特に関数からzvalを返す際にはこれを使うのが一般的です。このマクロの定義は次の通りです。

.. code-block:: c

  ZVAL_ZVAL(zv_dest, zv_src, copy, dtor)

``copy`` パラメーターはコピー先のzvalに対して ``zval_copy_ctor()`` を実行するかどうかの指定で、 ``dtor`` はコピー元のzvalに対して ``zval_ptr_dtor()`` を実行するかの指定です。ではこの４通りの振る舞いを順に見ていきましょう。

.. code-block:: c 

  ZVAL_ZVAL(zv_dest, zv_src, 0, 0);
  /* 上の方法は下の方法と等しい */
  ZVAL_COPY_VALUE(zv_dest, zv_src)

この場合、 ``ZVAL_ZVAL()`` は単に ``ZVAL_COPY_VALUE()`` となります。このマクロで 0,0の引数で呼び出すことはあまり意味がありません。より役立つのはcopy=1, dtor=0の場合です。

.. code-block:: c

  ZVAL_ZVAL(zv_dest, zv_src, 1, 0);
  /* 上の方法は下の方法と等しい */
  ZVAL_COPY_VALUE(zv_dest, zv_src);
  zval_copy_ctor(&zv_src);

これはありふれた ``MAKE_COPY_ZVAL()`` によるzvalのコピーと基本的に同じですが、 ``INIT_PZVAL()`` のステップだけがありません。これは既に初期化済みのzvalにコピーする際には有効です(例: ``return_value`` )。これにdtor=1とするのは ``zval_ptr_dtor()`` の呼び出しが増えるだけです。

.. code-block:: c

  ZVAL_ZVAL(zv_dest, zv_src, 1, 1);
  /* 上の方法は下の方法と等しい */
  ZVAL_COPY_VALUE(zv_dest, zv_src);
  zval_copy_ctor(zv_dest);
  zval_ptr_dtor(&zv_src);

最も興味深いのはcopy=0, dtor=1の場合です。

.. code-block:: c 

  ZVAL_ZVAL(zv_dest, zv_src, 0, 1);
  /* 上の方法は下の方法と等しい */
  ZVAL_COPY_VALUE(zv_dest, zv_src);
  ZVAL_NULL(zv_src);
  zval_ptr_dtor(&zv_src);

これは ``zv_src`` から値がコピーコンストラクタを呼び出すことがなくても ``zv_dest`` へと"移動" します。これは  ``zv_src`` のrefcountが1で ``zval_ptr_dtor()`` によって破棄される場合でのみおこなわれるべきです。もしrefcountが1よりも大きい場合はzvalはNULL型で残り続けます。

さらにもう２つ、 ``COPY_PZVAL_TO_ZVAL()`` と ``REPLACE_ZVAL_VALUE()`` というzvalのコピー用マクロがあります。両方とも滅多に使われないのでここではふれません。


zvalの分離
-----------

上で説明したマクロは主にzvalを別の保存場所にコピーしたい時に使われます。典型的な例としては ``return_value`` のzvalに値をコピーするということです。コピーオンライトの文脈で使われる別のマクロのセットで、"zvalの分離"のためのマクロがあります。この機能を理解するためには次のソースコードを読むことが一番でしょう。

.. code-block:: c 

  #define SEPARATE_ZVAL(ppzv)                     \
      do {                                        \
          if (Z_REFCOUNT_PP((ppzv)) > 1) {        \
              zval *new_zv;                       \
              Z_DELREF_PP(ppzv);                  \
              ALLOC_ZVAL(new_zv);                 \
              INIT_PZVAL_COPY(new_zv, *(ppzv));   \
              *(ppzv) = new_zv;                   \
              zval_copy_ctor(new_zv);             \
          }                                       \
      } while (0)

もしrefcountが1であれば ``SEPARATE_ZVAL`` は何もしません。refcountがそれよりも大きい場合、古いzvalの参照をひとつ減らしてから新しいzvalにコピーした後、その新しいzvalを ``*ppzv`` に割り当てます。このマクロは ``zval**`` を受け取り、それが指し示す ``zval*`` を変更することに注意してください。

実際のところ、このマクロはどのように使われるのでしょう。例えば、 ``$array[42]`` のように配列のオフセットを変更したい場合を想像してください。このためには、まず ``zval*`` の値が格納されている ``zval**`` のポインタを取得します。参照カウントのために、直接その値を変更することはできませんので(他で共有している可能性があるからです)、まずそのzvalを分離する必要があります。分離はもしrefcountが1であれば古いzvalをそのまま残し、1よりも大きければコピーとして動作します。後者では、この場合配列内に格納されている新しいzvalは ``*ppzv`` に割り当てられます。

この場合、 ``MAKE_COPY_ZVAL()`` で単純にコピーしたのでは十分ではないでしょう。なぜならコピーされたzvalは配列内に格納されないからです。

zvalの変更の前に ``SEPARATE_ZVAL()`` を直接使用することについて、zvalがis_ref=1でzvalの分離が行われるべきでないケースについてはまだ説明されていません。このケースを処理するために、まずPHPが用意しているis_refをフラグを操作するマクロをまずみていきましょう。

.. code-block:: c 

  Z_ISREF_P(zv_ptr)           /* zvalが参照かどうかを取得 */  

  Z_SET_ISREF_P(zv_ptr)       /* is_ref=1に設定 */
  Z_UNSET_ISREF_P(zv_ptr)     /* is_ref=0に設定 */  

  Z_SET_ISREF_TO_P(zv_ptr, 1) /* Z_SET_ISREF_P(zv_ptr)と同じ */
  Z_SET_ISREF_TO_P(zv_ptr, 0) /* Z_UNSET_ISREF_P(zv_ptr)と同じ */

これらのマクロも、これまでのマクロのように、接尾辞なし、 ``_P`` 、 ``_PP``  の接尾辞がついて、それぞれ ``zval`` 、 ``zval*`` 、 ``zval**`` を受け取る種類のものが使えます。さらに ``PZVAL_IS_REF()`` という古いマクロがあり、これは ``Z_ISREF_P()`` と同義です。

PHPが用意している ``SEPARATE_ZVAL()`` の２種類のマクロを使ってみましょう。

.. code-block:: c 

  #define SEPARATE_ZVAL_IF_NOT_REF(ppzv)      \
      if (!PZVAL_IS_REF(*ppzv)) {             \
          SEPARATE_ZVAL(ppzv);                \
      }  

  #define SEPARATE_ZVAL_TO_MAKE_IS_REF(ppzv)  \
      if (!PZVAL_IS_REF(*ppzv)) {             \
          SEPARATE_ZVAL(ppzv);                \
          Z_SET_ISREF_PP((ppzv));             \
      }

``SEPARATE_ZVAL_IF_NOT_REF()`` はコピーオンライトによってzvalを変更する際に普通使うことはないでしょう。 ``SEPARATE_ZVAL_TO_MAKE_IS_REF()`` はzvalを参照に変更したい場合に使われるマクロです。後者は主にZendEngineによって使われ、エクステンションのコードでは滅多に使われないでしょう。

``SEPARATE`` マクロ群には他のと少し違って動作する別のマクロがあります。

.. code-block:: c

  #define SEPARATE_ARG_IF_REF(varptr) \
      if (PZVAL_IS_REF(varptr)) { \
          zval *original_var = varptr; \
          ALLOC_ZVAL(varptr); \
          INIT_PZVAL_COPY(varptr, original_var); \
          zval_copy_ctor(varptr); \
      } else { \
          Z_ADDREF_P(varptr); \
      }

最初の違いは、このマクロは ``zval**`` ではなく ``zval*`` を受け取ることです。そのためマクロが分離する ``zval*`` は変更が出来ません。さらに、このマクロは ``SEPARATE_ZVAL`` マクロとは違ってrefcountをインクリメントしてくれています。

これらとは別に、このマクロは基本的に ``SEPARATE_ZVAL_IF_NO_REF()`` の補足的なマクロです。というのも、このマクロではzvalが参照の場合に分離をします。これは関数に渡された引数が参照ではなく値であることを確認するために主に使用されます。
