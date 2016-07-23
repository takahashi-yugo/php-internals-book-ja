オブジェクトハンドラー
======================

これまでのセクションでは既にいくつかのオブジェクトハンドラーを使ってきました。特に、ハンドラーを指定するために使用される構造体を生成する方法や ``clone_obj`` を使ってクローンの動作を実装する方法はご存知でしょう。しかし、これは単なる始まりにすぎません。というのも、PHPのオブジェクトの働きのほとんど全てはオブジェクトハンドラーを通して行われ、全てのマジックメソッドやマジックインターフェイスは内部的にオブジェクトやクラスのハンドラーによって実装されています。さらに、かなり多くのハンドラーはPHPのユーザーランドでは使えません。例えば、内部のクラスは独自の比較や型変換の振る舞いをおこなうことが出来ます。

非常に多くのオブジェクトハンドラーがあるので、その内のいくつかのハンドラーのための(前回のセクションでの型付き配列を使用した)例を取り上げることしかできません。他の全てのハンドラーについては簡単な説明にとどめます。

概略
----

この記事を書いている時点で、オブジェクトハンドラーには26種類あります。下記にそれぞれのシグネチャと簡単な説明と共に一覧で記載します。

.. c:member:: 
  zval *read_property(zval *object, zval *member, int type, const struct _zend_literal *key TSRMLS_DC)
  void write_property(zval *object, zval *member, zval *value, const struct _zend_literal *key TSRMLS_DC)
  int has_property(zval *object, zval *member, int has_set_exists, const struct _zend_literal *key TSRMLS_DC)
  void unset_property(zval *object, zval *member, const struct _zend_literal *key TSRMLS_DC)
  zval **get_property_ptr_ptr(zval *object, zval *member, const struct _zend_literal *key TSRMLS_DC)

  これらのハンドラーは ``__get`` 、 ``__set`` 、 ``__isset`` 、 ``__unset`` メソッドに対応しています。 ``get_property_ptr_ptr`` は内部的に参照で返す ``__get`` と同等のものです。これらに渡される ``zend_literal *key`` は最適化のためのもので、例えばプロパティ名の計算済みのハッシュ値を保持しています。

.. c:member::
  zval *read_dimension(zval *object, zval *offset, int type TSRMLS_DC)
  void write_dimension(zval *object, zval *offset, zval *value TSRMLS_DC)
  int has_dimension(zval *object, zval *member, int check_empty TSRMLS_DC)
  void unset_dimension(zval *object, zval *offset TSRMLS_DC)

  これらのハンドラーは ``ArrayAccess`` インターフェイスの内部的な表現です。

.. c:member::
  void set(zval **object, zval *value TSRMLS_DC)
  zval *get(zval *object TSRMLS_DC)

  これらのハンドラーは"object value"の取得と設定をおこないます。これらは複合代入演算子( ``+=`` や ``++`` など)を(ある程度までの)オーバーライドのために使うことが出来たり、主にプロキシーオブジェクトのために存在します。

.. c:member::
  HashTable *get_properties(zval *object TSRMLS_DC)
  HashTable *get_debug_info(zval *object, int *is_temp TSRMLS_DC)

  ハッシュテーブルのオブジェクトのプロパティを取得するために使用されます。前者はより一般的な目的のため、例えば ``get_object_vars`` 関数も使われるような場合のためのものです。一方、後者は ``var_dump`` のようなデバッグ関数において専らプロパティを表示するために使用されます。たとえオブジェクトが何らプロパティを持っていない場合でも、役に立つデバッグ情報を出力します。

.. c:member::
  union _zend_function *get_method(zval **object_ptr, char *method, int method_len, const struct _zend_literal *key TSRMLS_DC)
  int call_method(const char *method, INTERNAL_FUNCTION_PARAMETERS)

  ``get_method`` ハンドラーはあるメソッドを呼び出すために使用される ``zend_function`` を取得します。呼び出したい特定の ``zend_function`` がないが、 ``__call`` のような包括的な振る舞いをおこないたい場合は、 ``get_method`` は ``call_method`` が代わりに使用されるように ``ZEND_OVERLOADED_FUNCTION`` の合図を送る事が出来ます。

.. c:member::
  union _zend_function *get_constructor(zval *object TSRMLS_DC)

  ``get_method`` のようですが、コンストラクターの関数を取得します。このハンドラーをオーバーライドする最も一般的な理由は、ハンドラーの中でエラーを投げることで手動でのオブジェクトの生成を禁止するためです。

.. c:member::
  int count_elements(zval *object, long *count TSRMLS_DC)

  これは ``Countable::count`` メソッドの実装で内部的におこなっていることです。

.. c:member::
  int compare_objects(zval *object1, zval *object2 TSRMLS_DC)
  int cast_object(zval *readobj, zval *retval, int type TSRMLS_DC)

  内部のクラスは独自の比較の振る舞いを実装したり、全ての型との型変換においての振る舞いをオーバーライドする機能があります。一方で、ユーザーランドでのクラスは ``__toString`` を通して文字列への型変換のみオーバーライドすることが出来ます。

.. c:member::
  int get_closure(zval *obj, zend_class_entry **ce_ptr, union _zend_function **fptr_ptr, zval **zobj_ptr TSRMLS_DC)

  このハンドラーはオブジェクトが関数として使われる場合に呼び出されます。つまりこれは ``__invoke`` の内部版です。この名前は、主にクロージャの実装に使用される事から由来してます。

.. c:member::
  zend_class_entry *get_class_entry(const zval *object TSRMLS_DC)
  int get_class_name(const zval *object, const char **class_name, zend_uint *class_name_len, int parent TSRMLS_DC)

  これらの2つのハンドラーはオブジェクトからクラスエントリとクラス名を取得するために使われます。これらのハンドラーをオーバーライドする理由はあまりありません。それが必要になってくるような唯一のケースは、標準の ``zend_object`` を基底にしないで独自のオブジェクトの構造体をつくる場合です。
  
.. c:member::
  void add_ref(zval *object TSRMLS_DC)
  void del_ref(zval *object TSRMLS_DC)
  zend_object_value clone_obj(zval *object TSRMLS_DC)
  HashTable *get_gc(zval *object, zval ***table, int *n TSRMLS_DC)

  これらのハンドラーは様々なオブジェクトの管理タスクのために使われます。 ``add_ref`` は新たなzvalがオブジェクトの参照を始めた時に呼び出され、 ``del_ref`` は参照が除かれた時に呼び出されます。デフォルトではこれらのハンドラーはオブジェクトストアの参照カウントを変更します。これも実質的にはオーバーライドする理由はないでしょう。考え得る唯一のケースは、Zendのオブジェクトストアではなく、独自のストレージ機能を使う事を選択した場合です。

  既に ``clone_obj`` はご存知でしょうから、飛ばして ``get_gc`` にいきます。このハンドラーはオブジェクトによって保持されている全ての変数を返しますので、循環参照を適切に収集することが出来ます。

オブジェクトハンドラーを使用した配列アクセスの実装
--------------------------------------------------

前回のセクションでは、 ``ArrayAccess`` インターフェイスはバッファビューが配列のような振る舞いをおこなうために使用されています。 さて、``*_dimension`` のそれぞれのオブジェクトハンドラーを使うことで、この実装を改善していきたいと思います。これらのハンドラーは ``ArrayAccess`` の実装のためにも使用されますが、独自の実装をすることでメソッド呼び出しのオーバーヘッドが避けられるため、より高速となります。

配列の次元に関するオブジェクトハンドラーは ``read_dimension`` 、 ``write_dimension`` 、 ``has_dimension`` 、 ``unset_dimension`` です。これらのハンドラーは全て最初の引数にオブジェクトのzvalを、2番目にオフセットのzvalを受け取ります。ここでの目的では、オフセットが整数とならないといけないので、zvalから整数値を取得するためのヘルパー関数をまず紹介しましょう。

.. code-block:: c

  static long get_long_from_zval(zval *zv)
  {
      if (Z_TYPE_P(zv) == IS_LONG) {
          return Z_LVAL_P(zv);
      } else {
          zval tmp = *zv;
          zval_copy_ctor(&tmp);
          convert_to_long(&tmp);
          return Z_LVAL(tmp);
      }
  }

これでそれぞれのハンドラーを書くのはかなり簡単になりました。例えば、 ``read_dimension`` ハンドラーは下記のようになります。

.. code-block:: c

  static zval *array_buffer_view_read_dimension(zval *object, zval *zv_offset, int type TSRMLS_DC)
  {
      buffer_view_object *intern = zend_object_store_get_object(object TSRMLS_CC);
      zval *retval;
      long offset;  

      if (!zv_offset) {
          zend_throw_exception(NULL, "Cannot append to a typed array", 0 TSRMLS_CC);
          return NULL;
      }  

      offset = get_long_from_zval(zv_offset);
      if (offset < 0 || offset >= intern->length) {
          zend_throw_exception(NULL, "Offset is outside the buffer range", 0 TSRMLS_CC);
          return NULL;
      }  

      retval = buffer_view_offset_get(intern, offset);
      Z_DELREF_P(retval); /* どこからも参照されていない場合はRefcountを 0 にしなければならない */
      return retval;
  }

最後に ``Z_DELREF_P(retval)`` としている点に少々違和感を感じるかもしれません。というのも、 ``read_dimension`` はzvalがどこからも参照されていない場合には、refcountが0のzvalを返すことになっているからです(ここでも同様です)。ZendEnineはrefcountをインクリメントします。refcountが0であれば、返り値に対しての参照に関わる操作は意味がないとZendEngineが判断するようになります(実際には何も変更されていないため)

別の奇妙に思える点は、配列の値の読み取りのハンドラーの中で配列への値の追加ができるかどうか(これは ``zv_offset = NULL`` かどうかで判断できます)のチェックをしなければならないところです。これは上記のコードで使用されていない ``type`` のパラメーターに関連しています。このパラメーターによって、どのような文脈で読み取りが発生したかを特定できます。通常の ``$foo[0]`` だと、 ``type`` は ``BP_VAR_R`` となりますが、これは他にも ``BP_VAR_W`` 、 ``BP_VAR_RW`` 、 ``BP_VAR_IS`` 、 ``BP_VAR_UNSET`` の種類があります。このような"読み取りでない" タイプがどのような場合に発生するかを理解するために、下記の例を考えてみましょう。

.. code-block:: php

  <?php  

  $foo[0][1];        // [0] は read_dimension(..., BP_VAR_R),
                     // [1] は read_dimension(..., BP_VAR_R)
  $foo[0][1] = $bar; // [0] は read_dimension(..., BP_VAR_W),     [1] は write_dimension
  $foo[][1] = $bar;  // []  は read_dimension(..., BP_VAR_W),     [1] は write_dimension
  isset($foo[0][1]); // [0] は read_dimension(..., BP_VAR_IS),    [1] は has_dimension
  unset($foo[0][1]); // [0] は read_dimension(..., BP_VAR_UNSET), [1] は unset_dimension

ご覧のように、他の ``BP_VAR`` のタイプは次元をネストしてアクセスした際に発生します。この場合、最も外側の次元へのアクセスがこの操作における実際のハンドラーが呼ばれ、内側の次元へのアクセスはそれぞれのタイプでの読み取りハンドラーを通して行われます。そのため、次元がネストされたアクセスにおいて ``[]`` の配列への挿入の演算子が使われている場合、 ``read_dimension`` はオフセットが ``NULL`` で呼び出されます。

``type`` パラメーターは文脈に応じて、振る舞いを変更するために使用されます。例えば、 ``isset`` は何の警告や、エラー、例外も投げないことになっています。 ``BP_VAR_IS`` タイプかどうかを明示的にチェックすることで、この振る舞いを守ることが出来ます。

.. code-block:: c

  if (type == BP_VAR_IS)
      return &EG(uninitialized_zval_ptr);
  }

しかし今回の場合では、ネストされた次元でのアクセスは意味をなさないので、このような振る舞いについては特に気にする必要はありません。

残りのハンドラーは ``read_dimension`` と似ています(が、それほど扱い辛くはありません)。

.. code-block:: c
  
  static void array_buffer_view_write_dimension(
      zval *object, zval *zv_offset, zval *value TSRMLS_DC
  ) {
      buffer_view_object *intern = zend_object_store_get_object(object TSRMLS_CC);
      long offset;  

      if (!zv_offset) {
          zend_throw_exception(NULL, "Cannot append to a typed array", 0 TSRMLS_CC);
          return;
      }  

      offset = get_long_from_zval(zv_offset);
      if (offset < 0 || offset >= intern->length) {
          zend_throw_exception(NULL, "Offset is outside the buffer range", 0 TSRMLS_CC);
          return;
      }  

      buffer_view_offset_set(intern, offset, value);
  }  

  static int array_buffer_view_has_dimension(
      zval *object, zval *zv_offset, int check_empty TSRMLS_DC
  ) {
      buffer_view_object *intern = zend_object_store_get_object(object TSRMLS_CC);
      long offset = get_long_from_zval(zv_offset);  

      if (offset < 0 || offset >= intern->length) {
          return 0;
      }  

      if (check_empty) {
          int retval;
          zval *value = buffer_view_offset_get(intern, offset);
          retval = zend_is_true(value);
          zval_ptr_dtor(&value);
          return retval;
      }  

      return 1;
  }  

  static void array_buffer_view_unset_dimension(zval *object, zval *zv_offset TSRMLS_DC)
  {
      zend_throw_exception(NULL, "Cannot unset offsets in a typed array", 0 TSRMLS_CC);
  }


これらのハンドラーについて言うことはほとんどありません。注意を要するに値する唯一の点は ``has_dimension`` の ``check_empty`` パラメーターです。このパラメーターが ``0`` の場合は、 ``isset`` の呼び出しで、 ``1`` であれば ``empty`` での呼び出しであることを意味します。 ``isset`` では、単に存在チェックだけおこなわれ、 ``empty`` ではtrueかどうかのチェックが行われます。

最後に ``MINIT`` に新しいハンドラーを割り当てます。

.. code-block:: c

  memcpy(&array_buffer_view_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
  array_buffer_view_handlers.clone_obj       = array_buffer_view_clone; /* 前のセクションから */
  array_buffer_view_handlers.read_dimension  = array_buffer_view_read_dimension;
  array_buffer_view_handlers.write_dimension = array_buffer_view_write_dimension;
  array_buffer_view_handlers.has_dimension   = array_buffer_view_has_dimension;
  array_buffer_view_handlers.unset_dimension = array_buffer_view_unset_dimension;

これで全ての配列のアクセスがこれまで通り動作しつつ、高速となっているでしょう。

Honoring inheritance
--------------------

オブジェクトハンドラーを実装する際に常に考えなければならないある主要な問題は、ハンドラーが継承の連鎖の端から端までに適用されるということです。もしユーザーがビュークラスのひとつを拡張した場合、そのクラスもまた同じハンドラーを使用します。そのため、次元アクセスのハンドラーがオーバーライドされていると、ユーザーは継承先のクラスで ``ArrayAccess`` をもはや使用できなくなります。

この問題を解決するための簡単な方法は、次元アクセスのハンドラーの中でそのクラスが拡張されているのかを確認し、この場合では標準のハンドラーにさかのぼるようにするということです。

.. code-block:: c

  if (intern->std.ce->parent) {
      return zend_get_std_object_handlers()->read_dimension(object, zv_offset, type TSRMLS_CC);
  }


ビューオブジェクトの比較
------------------------

現状では、ビューオブジェクトは同じ型(そしてプロパティが何もない場合)であれば、常に等しいと判断されます。この動作は全く期待するものではありません。代わりに、自身で比較の振る舞いを実装すべきです。つまり、2つのバッファビューが同じバッファを、同じオフセット、同じ長さ、同じ型で使用している場合に等しいと判断されるという振る舞いです。さらに、2つのクラスエントリも同じであるべきでしょう(つまり継承したクラスとは等しいと判断されないということです)。追加として、プロパティも同じであるべきで、。言い換えれば、それぞれの内部のオブジェクトがバイト単位で同じであれば等しい2つのバッファビューは等しいということです。これは ``memcmp`` で簡単に確認できます。

.. code-block:: c

  static int array_buffer_view_compare_objects(zval *obj1, zval *obj2 TSRMLS_DC)
  {
      buffer_view_object *intern1 = zend_object_store_get_object(obj1 TSRMLS_CC);
      buffer_view_object *intern2 = zend_object_store_get_object(obj2 TSRMLS_CC);  

      if (memcmp(intern1, intern2, sizeof(buffer_view_object)) == 0) {
          return 0; /* 同じ */
      } else {
          return 1; /* 等しくない */
      }
  }

ご覧のように、 ``compare_objects`` ハンドラーは2つのオブジェクトを受け取って、両者の関係性を返します。返り値は -1(より小さい)、0(等しい)、1(より大きい)のいづれかです。

このケースでは大小の関係には意味はないので、 ``$view1 < $view2`` や ``$view1 > $view2`` が常にfalseとなるようにしたいです。これはオブジェクトが等しくない場合に、このハンドラーが1を返すようにすることで可能となります。1は"より大きい"を意味するので、 ``$view1 > $view2`` がtrueを返すと考えて、これで上手く動作するのか疑問に思うかもしれません。この策略が上手く動作する理由は、PHPが ``$a > $b`` を ``$b < $a`` (また ``$a >= $b`` を ``b <= $a`` )に自動的に変換するからです。そのため、常に"より小さい"の関係性で比較されるので1を返す(比較した結果の順序に関係なく)ことでどんな比較もfalseとなります。

同じような比較のハンドラーを ``ArrayBuffer`` クラスでも書くことができるでしょう。

デバッグ情報
------------

バッファビューのオブジェクトを ``var_dump`` や ``print_r`` でダンプしても、役に立つ情報は何も得られないでしょう。::

 object(Int8Array)#2 (0) {
 }

代わりに配列の中身が出力されるとずっと助かるでしょう。そのように振る舞いは ``get_debug_info`` ハンドラーを使用して実装できます。

.. code-block::c

  static HashTable *array_buffer_view_get_debug_info(zval *obj, int *is_temp TSRMLS_DC)
  {
      buffer_view_object *intern = zend_object_store_get_object(obj TSRMLS_CC);
      HashTable *props = Z_OBJPROP_P(obj);
      HashTable *ht;
      int i;  

      ALLOC_HASHTABLE(ht);
      ZEND_INIT_SYMTABLE_EX(ht, intern->length + zend_hash_num_elements(props), 0);
      zend_hash_copy(ht, props, (copy_ctor_func_t) zval_add_ref, NULL, sizeof(zval *));  

      *is_temp = 1;  

      for (i = 0; i < intern->length; ++i) {
          zval *value = buffer_view_offset_get(intern, i);
          zend_hash_index_update(ht, i, (void *) &value, sizeof(zval *), NULL);
      }  

      return ht;
  }

このハンドラーでは、サイズヒントを提供するために ``ZEND_INIT_SYMTABLE_EX`` を使用してハッシュテーブルを作成し、プロパティをコピーし(ユーザーが独自でプロパティを追加した場合のため)、それからビューをループしていき、ハッシュテーブルにその全ての要素を挿入していきます。

``is_temp`` のパラメーターに ``1`` を書き込むと、後で解放される一時的なハッシュテーブルを使用するということを意味します。代わりに、このポインターに ``0`` を書き込めば、ハッシュテーブルをどこかに保存しておいて、それを手動で解放しなければなりません(多くのオブジェクトはこのために、内部の構造体に何らかの ``debug_info`` フィールドを持っています)。


生成される出力の簡単な例を示します。

.. code-block:: php

  <?php
  $buffer = new ArrayBuffer(4);  

  $view = new Int8Array($buffer);
  $view->foo = 'bar';
  $view[0] = 10; $view[1] = 20; $view[2] = -10; $view[3] = -20;  

  var_dump($view);  

  // 出力  

  object(Int8Array)#2 (5) {
    ["foo"]=>
    string(3) "bar"
    [0]=>
    int(10)
    [1]=>
    int(20)
    [2]=>
    int(-10)
    [3]=>
    int(-20)
  }

型付き配列でもう1つ実装することができるハンドラーが ``count_elements`` で、これは内部での ``Countable::count()`` と同等のものです。このハンドラーについては何も特別なことはないので、読者の方への課題とします(継承のチェックを忘れないでくださいね!)。