独自のオブジェクトストレージ
============================

前回のセクションではシンプルな内部のクラスの作成のための準備を行いました。そこで説明した機能のほとんどは、PHPのユーザーランドと同じように動作するものを単に回りくどく表現されただけなので、かなり簡単なものだったはずです。一方、このセクションではユーザーランドのクラスでは利用できない領域へと進みます。つまり独自のオブジェクトストレージの作成とアクセスです。

どのようにしてオブジェクトは作られるか
--------------------------------------

初めのステップとして、PHPでどのようにオブジェクトが生成されるかを見ていきましょう。このためには、 ``object_and_properties_init`` マクロかそれに近いマクロでよりシンプルなものを使います。

.. code-block:: c

  // SomeClassというオブジェクト型を生成し、 properties_hashtableでプロパティを設定する
  zval *obj;
  MAKE_STD_ZVAL(obj);
  object_and_properties_init(obj, class_entry_of_SomeClass, properties_hashtable);  

  // SomeClassというオブジェクト型を生成する (プロパティはデフォルト)
  zval *obj;
  MAKE_STD_ZVAL(obj);
  object_init_ex(obj, class_entry_of_SomeClass);
  // = object_and_properties_init(obj, class_entry_of_SomeClass, NULL)  

  // デフォルトのオブジェクト(stdClass)を生成する
  zval *obj;
  MAKE_STD_ZVAL(obj);
  object_init(obj);
  // = object_init_ex(obj, NULL) = object_and_properties_init(obj, NULL, NULL)

最後のケース、つまり ``stdClass`` オブジェクトを生成する際には、おそらくその後にプロパティを加えたくなるでしょう。これは大抵、前の章での ``zend_update_property`` でするのではなく、代わりに ``add_property`` マクロを使用します。

.. code-block:: c

  add_property_long(obj, "id", id);
  add_property_string(obj, "name", name, 1); // 1 means the string should be copied
  add_property_bool(obj, "isAdmin", is_admin);
  // 同様に、_null(), _double(), _stringl(), _resource(), _zval()のマクロもある

オブジェクトが生成される際に実際何が起きているのでしょうか。それを探るために、 ``_object_and_properties_init`` 関数を見てみましょう。

.. code-block:: c

  ZEND_API int _object_and_properties_init(
      zval *arg, zend_class_entry *class_type, HashTable *properties ZEND_FILE_LINE_DC TSRMLS_DC
  ) {
      zend_object *object;  

      if (class_type->ce_flags
          & (ZEND_ACC_INTERFACE|ZEND_ACC_IMPLICIT_ABSTRACT_CLASS|ZEND_ACC_EXPLICIT_ABSTRACT_CLASS)
      ) {
          char *what = (class_type->ce_flags & ZEND_ACC_INTERFACE)                 ? "interface"
                     : ((class_type->ce_flags & ZEND_ACC_TRAIT) == ZEND_ACC_TRAIT) ? "trait"
                     : "abstract class";
          zend_error(E_ERROR, "Cannot instantiate %s %s", what, class_type->name);
      }  

      zend_update_class_constants(class_type TSRMLS_CC);  

      Z_TYPE_P(arg) = IS_OBJECT;
      if (class_type->create_object == NULL) {
          Z_OBJVAL_P(arg) = zend_objects_new(&object, class_type TSRMLS_CC);
          if (properties) {
              object->properties = properties;
              object->properties_table = NULL;
          } else {
              object_properties_init(object, class_type);
          }
      } else {
          Z_OBJVAL_P(arg) = class_type->create_object(class_type TSRMLS_CC);
      }
      return SUCCESS;
  }

この関数は基本的に3つのことをしています。初めに、インスタンスが実際生成可能かを検証し、それからクラス定数を解決します(これは最初のインスタンス生成時のみ実行されますが、その詳細はここでは重要ではありません)。その後、重要な部分がやってきます。つまり、クラスが ``create_object`` ハンドラーをもっているかチェックします。持っていればそのハンドラーが呼ばれ、持っていなければデフォルトの ``zend_objects_new`` の実装が使用されます(そして追加でプロパティが初期化されます)。

下記が ``zend_objects_new`` がその後行う処理です。

.. code-block:: c

  ZEND_API zend_object_value zend_objects_new(
      zend_object **object, zend_class_entry *class_type TSRMLS_DC
  ) {
      zend_object_value retval;  

      *object = emalloc(sizeof(zend_object));
      (*object)->ce = class_type;
      (*object)->properties = NULL;
      (*object)->properties_table = NULL;
      (*object)->guards = NULL;
      retval.handle = zend_objects_store_put(*object,
          (zend_objects_store_dtor_t) zend_objects_destroy_object,
          (zend_objects_free_object_storage_t) zend_objects_free_object_storage,
          NULL TSRMLS_CC
      );
      retval.handlers = &std_object_handlers;
      return retval;
  }

上記のコードには3つの興味深い点があります。1つは ``zend_object`` 構造体で、それは次のように定義されています。

.. code-block:: c

  typedef struct _zend_object {
      zend_class_entry *ce;
      HashTable *properties;
      zval **properties_table;
      HashTable *guards; /* protects from __get/__set ... recursion */
  } zend_object;

これは"標準的"なオブジェクトの構造体です。構造体は、オブジェクト生成で使用されるクラスエントリ、プロパティのハッシュテーブル、プロパティ"テーブル"、そして再帰保護のためのハッシュテーブルを持っています。 ``properties`` と ``properties_table`` の厳密な違いは、この章の後のセクションで扱いますが、現時点では、後者がクラスのプロパティの宣言に使用され、前者は宣言されなかったプロパティのためのものということだけ知っておいておいて下さい。 ``guards`` の仕組みがどのように動作するかも後のセクションで扱います。

``zend_objects_new`` 関数は前述の標準的なオブジェクトのメモリを割り当て、初期化します。その後、オブジェクトストアにオブジェクトのデータを格納するために ``zend_objects_store_put`` を呼び出します。オブジェクトストアとは動的にリサイズされる ``zend_object_store_bucket`` の配列に過ぎません。

.. code-block:: c

  typedef struct _zend_object_store_bucket {
      zend_bool destructor_called;
      zend_bool valid;
      union _store_bucket {
          struct _store_object {
              void *object;
              zend_objects_store_dtor_t dtor;
              zend_objects_free_object_storage_t free_storage;
              zend_objects_store_clone_t clone;
              const zend_object_handlers *handlers;
              zend_uint refcount;
              gc_root_buffer *buffered;
          } obj;
          struct {
              int next;
          } free_list;
      } bucket;
  } zend_object_store_bucket;

ここでメインとなる部分は ``_store_object`` 構造体で、 ``void *object`` メンバーに格納されているオブジェクトを持っており、その後に、デストラクタ、メモリ解放、クローン用の3つのハンドラーが続ています。この構造体には追加的なものも幾つか含まれており、例えば、自身の ``refcount`` プロパティを保持しています。なぜなら、オブジェクトストアの中の1つのオブジェクトは複数のzvalから同時に参照される可能性があり、PHPは後で解放出来るように、いくつ参照があるか把握しておく必要があるからです。加えて、オブジェクトの ``handlers`` (オブジェクトの破棄で必要になります)やガーベッジコレクションのルートバッファ(PHPのサイクルコレクターがどのように動作するかは後の章で扱います)も格納されています。

``zend_objects_new`` 関数に話を戻すと、最後にこの関数が行っているのは、オブジェクトの ``handlers`` にデフォルトの ``std_object_handlers`` を設定するということです。

create_objectのオーバーライド
-------------------------------

独自のオブジェクトストレージを使用したい場合、基本的に上述の3つのステップを繰り返すことになります。つまり、まず基礎となる標準的なオブジェクトを含んでいるオブジェクトのメモリ割り当てと初期化をします。その後、オブジェクトストアにいくつかのハンドラーと共に格納します。そして最後にオブジェクトハンドラーの構造体を設定します。

そのためには、 ``create_object`` のクラスハンドラーをオーバーライドしなければなりません。下記はどのようになるかの見本となる例です。

.. code-block:: c

  zend_class_entry *test_ce;  

  /*  独自のオブジェクトで使用されるオブジェクトハンドラーを保持するために
   * (真のグローバルな)変数が必要になる。 このオブジェクトハンドラーはMINITで初期化される。 */
  static zend_object_handlers test_object_handlers;  

  /* 独自のクラス構造体。 この構造体は、まず最初のプロパティに `zend_object` の値 (ポインターではない！) をもっており、
   * その後に、追加で必要なプロパティの宣言が続く */
  typedef struct _test_object {
      zend_object std;
      long additional_property;
  } test_object;  

  /* オブジェクトが解放される際に呼び出されるハンドラー。このハンドラーでは
   * std オブジェクトの破棄(プロパティのハッシュテーブル等もこれにより解放される)とオブジェクトの構造体自体の解放もおこなわなければならない。
   * (また、他のリソースにメモリ割り当てがされている場合は、ここでそれらも明示的に解放しなければならない) */
  static void test_free_object_storage_handler(test_object *intern TSRMLS_DC)
  {
      zend_object_std_dtor(&intern->std TSRMLS_CC);
      efree(intern);
  }  

  /* このハンドラーはオブジェクトの生成の際に呼び出されるハンドラー。 このハンドラーはクラスエントリを引数で受け取り、
   * (このクラスを継承しているクラスの場合でも、このクラスのクラスエントリが使われるので、このハンドラーが使用されることになる) 
   * オブジェクトの値(オブジェクトとオブジェクトハンドラーの構造体を保持している)を返す。 */
  zend_object_value test_create_object_handler(zend_class_entry *class_type TSRMLS_DC)
  {
      zend_object_value retval;  

      /* 内部オブジェクト構造体にメモリ割り当てを行う。 
       * 慣習として、内部構造体を保持している変数は `intern` とすることが多い。 */
      test_object *intern = emalloc(sizeof(test_object));
      memset(intern, 0, sizeof(test_object));  

      /* 元となる std zend_objectを初期化する  */
      zend_object_std_init(&intern->std, class_type TSRMLS_CC);  

      /* プロパティを使用しない場合でも、 object_properties_init()を呼び出す必要がある。
       * これは継承先のクラスでプロパティを使用するかもしれないからである。
       * (ここで行う処理の多くは大抵、継承先のクラスを破壊しないためのもの)
       */
      object_properties_init(&intern->std, class_type);  

      /* オブジェクトを、デフォルトの破棄ハンドラーと独自の解放ハンドラーを持たせて、オブジェクトストアに保持する。 
       * 最後の引数のNULLはクローンハンドラーで、ここでは空にしておく。
       */
      retval.handle = zend_objects_store_put(
          intern,
          (zend_objects_store_dtor_t) zend_objects_destroy_object,
          (zend_objects_free_object_storage_t) test_free_object_storage_handler,
          NULL TSRMLS_CC
      );  

      /* 独自のオブジェクトハンドラーを割り当てる */
      retval.handlers = &test_object_handlers;  

      return retval;
  }  

  /* ここではメソッドはなし */
  const zend_function_entry test_functions[] = {
      PHP_FE_END
  };  

  PHP_MINIT_FUNCTION(test2)
  {
      /* いつものクラス登録処理... */
      zend_class_entry tmp_ce;
      INIT_CLASS_ENTRY(tmp_ce, "Test", test_functions);
      test_ce = zend_register_internal_class(&tmp_ce TSRMLS_CC);  

      /* クラスエントリにクラス生成のハンドラーを設定する */
      test_ce->create_object = test_create_object_handler;  

      /* デフォルトのオブジェクトハンドラーに独自のオブジェクトハンドラーを初期化する。 
       * この後、普通は個々のハンドラーをオーバーライドすることになるが、ここではデフォルトのハンドラーのままにしておく。 */
      memcpy(&test_object_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));  

      return SUCCESS;
  }

上記のコードは実際にはまだ使えるものとはなっていませんが、PHPの内部での全てのクラスの基本構造を示しています。

オブジェクトストアのハンドラー
------------------------------

既に述べたように、3つのオブジェクトストレージハンドラーがあります。1つは破棄、1つは解放、1つはクローンのためのものです。

破棄と解放の両方のハンドラーは同じような事をするように思えて、最初は少し混乱するかもしれません。両方のハンドラーがあるのは、PHPがオブジェクトの破棄のシステムにおいて、最初にデストラクターが呼ばれて、その後にオブジェクトが解放されるという、2つのフェーズが用意されているためです。両者のフェーズはお互いに分離して実行されます。

実際のところ、これはスクリプトが終了した時点でまだ存在している全てのオブジェクトに対して実行されます。それらのオブジェクトに対して、PHPはまず全てのデストラクターを呼び出します(これはシャットダウン時に実行するように登録されている関数が全て呼び出だした直後です)が、解放処理はその後のエグゼキューターのシャットダウンのいち部分として行われるのです。デストラクターと解放の分離をするのは、エグゼキューターのシャットダウンの間にデストラクターが実行されないことを保証するためです。さもないと、ユーザーランドのコードがシャットダウン途中の状態で実行されるという状況になってしまうかもしれないからです。この分離がないと、シャットダウン中の ``zval_ptr_dtor`` が失敗してしまうでしょう。　

デストラクターのハンドラーの別の特徴としては、必ずしも呼び出されるとは限らないということです。例えば、デストラクターで ``die`` を実行していたとすると、残りのオブジェクトのデストラクターはスキップされてしまいます。

つまり、基本的にその2つのハンドラーの違いは、デストラクターはユーザーランドで実行されるが呼び出されるとは限らず、一方で、解放処理は常に呼び出されるが、どんなPHPコードも実行されてはいけないということです。そのため、多くの場合では、解放ハンドラーのみ独自のものを指定して、デストラクターのハンドラーとしては、 ``__destruct`` が存在していればそれを呼び出すというデフォルトの挙動を提供している ``zend_objects_destroy_object`` を使います。たとえ、 ``__destruct`` を使用していなくてもデストラクターのハンドラーは指定しなければなりません。さもないと、継承先のクラスでデストラクターが使用できなくなってしまうでしょう。

さて、残すはクローンのハンドラーだけです。このハンドラーの意味するところは単純ですが、少し扱いにくくなっています。下記にクローンハンドラーがどのようになるかの例を示します。

.. code-block:: c

  static void test_clone_object_storage_handler(
      test_object *object, test_object **object_clone_target TSRMLS_DC
  ) {
      /* 新しいオブジェクトを生成 */
      test_object *object_clone = emalloc(sizeof(test_object));
      zend_object_std_init(&object_clone->std, object->std.ce TSRMLS_CC);
      object_properties_init(&object_clone->std, object->std.ce);  

      /* ここで他にクローンすべきものがあればクローンする */
      object_clone->additional_property = object->additional_property;  

      /* クローンされたオブジェクトを返す */
      *object_clone_target = object_clone;
  }

``zend_objects_store_put`` の最後の引数にクローンハンドラーが渡されています。

.. code-block:: c

  retval.handle = zend_objects_store_put(
      intern,
      (zend_objects_store_dtor_t) zend_objects_destroy_object,
      (zend_objects_free_object_storage_t) test_free_object_storage_handler,
      (zend_objects_store_clone_t) test_clone_object_storage_handler
      TSRMLS_CC
  );

しかしこれではまだクローンハンドラーが動作するのに十分でありません。というのも、デフォルトではオブジェクトストレージのクローンハンドラーは単に無視されてしまうからです。動作させるためには、オブジェクトハンドラーの構造体のデフォルトのクローンハンドラーを ``zend_objects_store_clone_obj`` で置き換えます。

.. code-block:: c

  memcpy(&test_object_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers));
  test_object_handler.clone_obj = zend_objects_store_clone_obj;

しかし、標準のクローンハンドラーを上書きすると、それによっていくつか問題が出てきます。つまり、現在の(独自のオブジェクトストレージ上のものではなく実際の)プロパティがコピーされず、また ``__clone`` メソッドも呼ばれないということです。そのため、内部のクラスのほとんどが、オブジェクトストレージのクローンハンドラーを通して設定するよりもむしろ、代わりに直接専用のクローンハンドラーを指定しています。このやり方にはちょっとしたイディオムがあります。例として、デフォルトのクローンハンドラーは以下のようになっています。

.. code-block:: c

  ZEND_API zend_object_value zend_objects_clone_obj(zval *zobject TSRMLS_DC)
  {
      zend_object_value new_obj_val;
      zend_object *old_object;
      zend_object *new_object;
      zend_object_handle handle = Z_OBJ_HANDLE_P(zobject);  

      /* 生成が上書きでないと仮定すると、クローン処理は上書きされたものに依存してしまうので、
       * 上書きされたもの自体も、ここで上書きしなければならない */
      old_object = zend_objects_get_address(zobject TSRMLS_CC);
      new_obj_val = zend_objects_new(&new_object, old_object->ce TSRMLS_CC);  

      zend_objects_clone_members(new_object, new_obj_val, old_object, handle TSRMLS_CC);  

      return new_obj_val;
  }


この関数はまずオブジェクトから ``zend_objects_get_address`` を使用して ``zend_object*`` の構造体を取得し、同じクラスエントリから新しいオブジェクトを生成( ``zend_objects_new`` を使用)し、それから(名前の通り)プロパティをクローンする ``zend_objects_clone_members`` を呼び出しますが、もし存在すれば ``__clone`` も呼び出します。

独自のクローンするためのハンドラーも似たかたちとなりますが、主な違いは ``zend_objects_new`` を使うのではなく、むしろ ``create_object`` ハンドラーを呼び出すようにします。

.. code-block:: c

  static zend_object_value test_clone_handler(zval *object TSRMLS_DC)
  {
      /* 古いオブジェクトの内部構造体を取得 */
      test_object *old_object = zend_object_store_get_object(object TSRMLS_CC);  

      /* 同じクラスエントリで新しいオブジェクトを生成する。 この結果得られるのは、新しいオブジェクトの実際の内部構造体ではなく、
       * zend_object_valueが戻ってくるだけである。  */
      zend_object_value new_object_val = test_create_object_handler(Z_OBJCE_P(object) TSRMLS_CC);  

      /* 内部構造体を取得するためには、 create_objectハンドラーで得られたhandleを使用して
       * オブジェクトストアから取得しなければならない
       */
      test_object *new_object = zend_object_store_get_object_by_handle(
          new_object_val.handle TSRMLS_CC
      );  

      /* プロパティをクローンして __clone を呼び出す */
      zend_objects_clone_members(
          &new_object->std, new_object_val,
          &old_object->std, Z_OBJ_HANDLE_P(object) TSRMLS_CC
      );  

      /* ここに独自のクローン処理を記述する */
      new_object->additional_property = old_object->additional_property;  

      return new_object_val;
  }  

  /* ... */
  test_object_handler.clone_obj = test_clone_handler;



オブジェクトストアとのやりとり
------------------------------

これまで示してきたサンプルコードで、オブジェクトストアとのやりとりを行うための幾つかの関数を見てきました。まずは ``zend_objects_store_put`` で、これはオブジェクトストアにオブジェクトを格納するために使用されます。また、オブジェクトストアからオブジェクトを取得するための次の3つの関数にも言及しました。1つが ``zend_object_store_get_object_by_handle()`` で、これは名前の通り、与えられたハンドルによってオブジェクトストアからオブジェクトを取得します。この関数はオブジェクトハンドルは持っているが、それに関連するzvalを持っていない場合に(例えばクローンハンドラーの中で)使用されます。一方で、他の多くのケースでは、zvalを受け取り、そこからハンドルを取り出す ``zend_object_store_get_object()`` を使うことが多いでしょう。

3つ目のゲッター関数は ``zend_objects_get_address()`` で、これは ``zend_object_store_get_object()`` とちょうど同じことをしますが、 ``void*`` ではなく ``zend_object*`` として結果を返します。C言語では ``void*`` だと他のポインター型に暗黙的に型変換できるので、この関数はかなり使いづらいでしょう。

これらの関数の中で最も重要な関数は ``zend_object_store_get_object()`` です。この関数を使うことが多くなるでしょう。ほとんど全てのメソッドは次のようになるでしょう。

.. code-block:: c

  PHP_METHOD(Test, foo)
  {
      zval *object;
      test_object *intern;  

      if (zend_parse_parameters_none() == FAILURE) {
          return;
      }  

      object = getThis();
      intern = zend_object_store_get_object(object TSRMLS_CC);  

      /* 内部のプロパティを返すといったような処理をここで行う */
      RETURN_LONG(intern->additional_property);
  }

オブジェクトストアで提供されている関数は、オブジェクトの参照カウントを管理するものなど、まだいくつかありますが、それらは直接には滅多に使用されないのでここでは扱いません。


