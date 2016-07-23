内部構造と実装
===============

PHPのオブジェクト指向に関するこの(最後の)セクションでは、これまで軽く触れるだけで留めていた、内部構造について見てきたいと思います。特に、デフォルトのオブジェクトの構造やオブジェクトストアについてより徹底的に説明していきます。

オブジェクトのプロパティ
------------------------

PHPのオブジェクト指向の仕組みで最も複雑な部分は、おそらくオブジェクトのプロパティの操作でしょう。以下では、その一端をより詳細に見ていきます。

プロパティストレージ
~~~~~~~~~~~~~~~~~~~~~

PHPではオブジェクトのプロパティは宣言できますが、宣言しないといけないというわけではありません。このような状況をどのように処理するのが効率的でしょうか。それを探るため、今一度、標準の ``zend_object`` 構造体を思い出してみましょう。

.. code-block:: c

  typedef struct _zend_object {
      zend_class_entry *ce;
      HashTable *properties;
      zval **properties_table;
      HashTable *guards;
  } zend_object;

この構造体はプロパティを保持するために2つのフィールドを含んでいます。つまり ``properties`` のハッシュテーブルと ``properties_table`` のzvalの配列です。分離したその2つのフィールドは、宣言されたプロパティと動的に追加されたプロパティの両方を最も良い方法で操作するために使用されます。動的に追加される、つまりクラスで宣言されていないプロパティに関しては、 ``properties`` (単なるプロパティ名 => 値のマッピング)を使うしか仕方ありません。

一方、宣言されたプロパティに関しては、それらをハッシュテーブルで保持するのは非常に無駄となるでしょう。というのも、PHPのハッシュテーブルは要素毎のオーバーヘッドが非常に大きいですが(100バイト近い)、ここで必要なのは値の ``zval`` へのポインターを保持するだけだからです。このため、PHPはちょっとしたトリックを採用しています。その方法とは、プロパティは通常のCの配列に格納し、オフセットを使ってアクセスするようにするというものです。それぞれのプロパティ名のためのオフセットはクラスエントリ内の(グローバルの)ハッシュテーブルに格納されています。そのため、プロパティの参照は一段階余分に間接参照されます。つまり、直接にプロパティの値を取得するのではなく、まずプロパティのオフセットを取得してから、そのオフセットを使用して実際のプロパティの値を取り出します。

プロパティ情報(ストレージのオフセットも含めて)は ``class_entry->properties_info`` に格納されています。このハッシュテーブルはプロパティ名を ``zend_property_info`` 構造体に対応付けしています。

.. code-block:: c

  typedef struct _zend_property_info {
      zend_uint flags;
      const char *name;
      int name_length;
      ulong h;                 /* ハッシュ名 */
      int offset;              /* ストレージのオフセット */
      const char *doc_comment;
      int doc_comment_len;
      zend_class_entry *ce;    /* 宣言するクラスのクラスエントリー */
  } zend_property_info;

ここで１つ残る疑問は、プロパティの両方のタイプが存在する際にどうなるかということです。この場合、両方のプロパティの構造体が同時に使用されます。全てのハッシュテーブルは ``properties`` ハッシュテーブルに格納されますが、同時に ``properties_table`` にもプロパティへのポインターが含まれます。しかし両方が使用されている場合には、プロパティテーブルは ``zval*`` ではなく ``zval**`` で値を保持することに注意してください。

全てのプロパティが宣言されていたとしても、 ``get_properties`` ハンドラーが使われる場合などでは、ハッシュテーブルとしてのプロパティが必要になることがあります。この場合、PHPは ``properties`` を使用する方に切り替えます(あるいは前述したように、両方を使用するというハイブリッドなアプローチをとることもあります)。このハンドリングは ``rebuild_object_properties`` 関数によって行われます。

.. code-block:: c

  ZEND_API HashTable *zend_std_get_properties(zval *object TSRMLS_DC)
  {
      zend_object *zobj;
      zobj = Z_OBJ_P(object);
      if (!zobj->properties) {
          rebuild_object_properties(zobj);
      }
      return zobj->properties;
  }

プロパティの名前修飾
~~~~~~~~~~~~~~~~~~~~

下記のコードスニペットについて考えてみましょう。

.. code-block:: php

  <?php  

  class A {
      private $prop = 'A';
  }  

  class B extends A {
      private $prop = 'B';
  }  

  class C extends B {
      protected $prop = 'C';
  }  

  var_dump(new C);  

  // Output:
  object(C)#1 (3) {
    ["prop":protected]=>
    string(1) "C"
    ["prop":"B":private]=>
    string(1) "B"
    ["prop":"A":private]=>
    string(1) "A"
  }

上記の例では、同じ名前の ``$prop`` というプロパティが3回定義されています。それぞれ、 ``A`` のprivateなプロパティ、 ``B`` のprivateなプロパティ、 ``C`` のprotectedなプロパティとなっています。これら3つのプロパティが同じ名前であっても、それぞれ別のプロパティとして区別されていますし、別個のストレージを必要とします。

このような状況に対応するために、PHPはプロパティ名をプロパティのアクセス権のタイプと定義しているクラスで修飾しています。::

  class Foo { private $prop;   } => "\0Foo\0prop"
  class Bar { private $prop;   } => "\0Bar\0prop"
  class Rab { protected $prop; } => "\0*\0prop"
  class Oof { public $prop;    } => "prop"

ご覧のように、publicなプロパティはそのままの名前で、protectedなプロパティには ``\0*\0`` の接頭辞( ``\0`` はヌルバイトです)が付き、privateなプロパティには ``\0ClassName\0`` で始まるようになります。

大抵は、この修飾された名前はユーザーランド上には出てこないようになっています。この名前が見受けられるのはいくつかの特殊なケースのみで、例えばオブジェクトを配列に型変換した場合や、シリアライズされた結果でなどです。内部的にも、この名前修飾(マングリング)については意識しなくてもよいようになっていて、例えば ``zend_declare_property`` APIは自動的に名前修飾してくれます。

唯一、名前修飾について気をつけないとならないケースは、 ``property_info->name`` フィールドを使う場合や、 ``zobj->properties`` のハッシュテーブルを直接使う場合になります。この場合には、 ``zend_(un)mangle_property_name`` APIを使えばよいでしょう。

.. code-block:: c

  // アンマングリング
  const char *class_name, *property_name;
  int property_name_len;  

  if (zend_unmangle_property_name_ex(
          mangled_property_name, mangled_property_name_len,
          &class_name, &property_name, &property_name_len
      ) == SUCCESS) {
      // ...
  }  

  // マングリング
  char *mangled_property_name;
  int mangled_property_name_len;  

  zend_mangle_property_name(
      &mangled_property_name, &mangled_property_name_len,
      class_name, class_name_len, property_name, property_name_len,
      should_do_persistent_alloc ? 1 : 0
  );

プロパティの再帰保護機能
~~~~~~~~~~~~~~~~~~~~~~~~

``zend_object`` の最後のメンバーは ``HashTable *guards`` フィールドです。これが何のために使用されるのかを探るため、``__set`` のマジックメソッドを使っている下記のコードがどのような結果になるのかを考えてみてください。

.. code-block:: php

  <?php  

  class Foo {
      public function __set($name, $value) {
          $this->$name = $value;
      }
  }  

  $foo = new Foo;
  $foo->bar = 'baz';
  var_dump($foo->bar);

このスクリプトでの ``$foo->bar = 'baz'`` の代入は、 ``$bar`` のプロパティが定義されていないので、 ``$foo->__set('bar', 'baz')`` が実行されるでしょう。この場合、メソッドの中身の ``$this->$name = $value`` の行は ``$foo->bar = 'baz'`` となるでしょう。ここでも ``$bar`` が定義されていないのは同じです。ではここでも ``__set`` が(再帰的に)呼び出されることを意味するのでしょうか。


実際にはそのようなことは起こりません。むしろ、PHPは ``__set`` 内かどうかを確認して、再帰呼び出しをしないようにします。このようにすることで、新しい ``$bar`` というプロパティがつくられます。この振る舞いを実装するために、PHPは既に ``__set`` が呼び出しているかどうかを記憶しておくという再帰保護機能を使っています。このような保護機能は、プロパティ名と ``zend_guard`` 構造体を対応付けした ``guards`` というハッシュテーブルに保持されています。

.. code-block:: c

  typedef struct _zend_guard {
      zend_bool in_get;
      zend_bool in_set;
      zend_bool in_unset;
      zend_bool in_isset;
      zend_bool dummy; /* sizeof(zend_guard) must not be equal to sizeof(void*) */
  } zend_guard;

オブジェクトストア
------------------

これまで多くのオブジェクトストアを使ってきましたが、ここではより詳細に見ていきましょう。

.. code-block:: c

  typedef struct _zend_objects_store {
      zend_object_store_bucket *object_buckets;
      zend_uint top;
      zend_uint size;
      int free_list_head;
  } zend_objects_store;

基本的に、オブジェクトストアとは動的にリサイズされる ``object_buckets`` という配列です。 ``size`` はメモリ割り当てのサイズを指定する一方で、 ``top`` は使用されている次のオブジェクトハンドルを表しています。オブジェクトハンドルは1から始まって、全てのハンドルは真と評価できるものであることが保証されています。そのため、 ``top == 1`` であれば、次のオブジェクトでは ``handle = 1`` となりますが、 ``object_buckets[0]`` の位置に格納されています。

``free_list_head`` は未使用のバケットの連結リストの先頭要素です。オブジェクトが破棄される時は常に、未使用のバケットは破棄されずに残り、この配列に格納されます。もし新しいオブジェクトが生成された時に未使用のバケットが存在していれば(つまり ``free_list_head`` が ``-1`` でない場合)、 ``top`` のバケットではなく、代わりにこのバケットが使用されます。

この連結リストがどのように管理されているのかを確認するため、 ``zend_object_store_bucket`` 構造体をみてみましょう。

.. code-block:: c

  typedef struct _zend_object_store_bucket {
      zend_bool destructor_called;
      zend_bool valid;
      zend_uchar apply_count;
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

バケットが使用されていれば(つまりオブジェクトを格納していれば)、 ``valid`` メンバーは1となります。この場合、共有体の ``struct _store_object`` の部分が使用されることになります。バケットが使用されていなければ、 ``valid`` が0となり、 ``free_list.next`` が使用されます。

未使用のオブジェクトハンドルを回収する様子を下記の簡単なスクリプトで確認することが出来ます。

.. code-block:: php

  <?php
  var_dump($a = new stdClass); // object(stdClass)#1 (0) {}
  var_dump($b = new stdClass); // object(stdClass)#2 (0) {}
  var_dump($c = new stdClass); // object(stdClass)#3 (0) {}

  unset($b); // handle 2を解放
  unset($a); // handle 1を解放

  var_dump($e = new stdClass); // object(stdClass)#1 (0) {}
  var_dump($f = new stdClass); // object(stdClass)#2 (0) {}

ご覧のように、 ``$b`` と ``$a`` のハンドルが破棄された順番とは逆順に再利用されていることが分かります。

``valid`` とは別で、バケットの構造体は ``destructor_called`` というフラグも持っています。このフラグはPHPのオブジェクトの2段階の破棄プロセスで必要なものです。2段階の破棄プロセスは既に説明した通り、PHPはデストラクタの段階(ユーザーランドから実行できるが、常に実行する必要はないもの)と解放の段階(ユーザーランドからは実行出来ず、常に実行する必要があるもの)のことです。デストラクタのハンドラーが呼び出された後に、 ``destructor_called`` フラグが1にセットされ、オブジェクトが解放される際にもう一度デストラクタが実行されないようにします。

``apply_count`` メンバーは ``HashTable`` の ``nApplyCount`` メンバーと同じ役割を持っています。つまり、 無限の再帰を保護するためのものです。このメンバーは ``Z_OBJ_UNPROTECT_RECURSION(zval_ptr)`` (再帰から抜ける)や ``Z_OBJ_PROTECT_RECURSION(zval_ptr)`` (再帰に入る)のマクロを通して使用されます。後者は、オブジェクトのネストが3階層以上になるとエラーが投げられます。現在のところ、この保護の仕組みはオブジェクトの比較ハンドラーの中でのみしか使用されていません。

``_store_object`` 構造体の ``handlers`` メンバーはオブジェクトの破棄でも必要なものです。というのも、 ``dtor`` ハンドラーは格納されているオブジェクトとそのハンドルが渡されるだけだからです。

.. code-block:: c

  typedef void (*zend_objects_store_dtor_t)(void *object, zend_object_handle handle TSRMLS_DC);

しかし、 ``__destruct`` を呼び出すためには、PHPはzvalが必要になります。そのため、渡されたオブジェクトハンドルや ``bucket.obj.handlers`` に格納されているオブジェクトハンドラーを使用して一時的なzvalを生成します。問題は、このメンバーが ``zval_ptr_dtor`` を通してか、そのzvalが分かっているような他のメソッドを通してオブジェクトが破棄された場合にのみ、セットされるということです。

一方、オブジェクトが( ``zend_objects_store_call_destructors`` を使用して)スクリプトのシャットダウンの段階で破棄されると、zvalが分かりません。この場合、 ``bucket.obj.handlers`` は ``NULL`` となり、PHPはデフォルトのオブジェクトハンドラーを使用します。そのため、オーバーロードされたオブジェクトの振る舞いは ``__destruct`` の中では利用できないことがあります。下記がその例です。::

  class DLL extends SplDoublyLinkedList {
      public function __destruct() {
          var_dump($this);
      }
  }  

  $dll = new DLL;
  $dll->push(1);
  $dll->push(2);
  $dll->push(3);  

  var_dump($dll);  

  set_error_handler(function() use ($dll) {});

このコードスニペットは ``SplDoublyLinkedList`` に ``__destruct`` メソッドを追加しており、エラーハンドラー(エラーハンドラーはシャットダウン中に解放される最後のもののうちの1つです)に組み込むことでシャットダウン中にデストラクターが実行されることを強制します。このコードは下記のような結果を出力します。::

  object(DLL)#1 (2) {
    ["flags":"SplDoublyLinkedList":private]=>
    int(0)
    ["dllist":"SplDoublyLinkedList":private]=>
    array(3) {
      [0]=>
      int(1)
      [1]=>
      int(2)
      [2]=>
      int(3)
    }
  }
  object(DLL)#1 (0) {
  }

デストラクタの外側の ``var_dump`` では、 ``get_debug_info`` が呼び出され、役に立つデバッグ情報が得られます。デストラクタの内側では、PHPはデフォルトのオブジェクトハンドラーを使用しますのでクラス名以外なにも情報が得られていません。同じことが、クローンや比較などのような他のハンドラーにも当てはまり、正常に動作しなくなるでしょう。

これで、オブジェクト指向についてのこの章を終わります。今やあなたはPHPのオブジェクト指向のシステムの動作の仕組みや、エクステンションの利用法について十分よく理解されていることでしょう。
