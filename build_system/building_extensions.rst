
PHPエクステンションをビルドする
================================

さて、PHP自体のコンパイルはみてきましたので、次はエクステンションのコンパイルへとすすみましょう。エクステンションのビルドがどのように行われ、指定可能なオプションによってどのように変わるかをみていきます。

エクステンションのロード
-------------------------

前回のセクションでふれたように、PHPエクステンションはPHPバイナリへと静的にビルドされるか、共有オブジェクト( ``.so`` ) へとコンパイルするかのどちらかになります。組み込みのエクステンションの多くにとって静的な関連付けがデフォルトの動作ですが、 ``./configure`` に ``--enable-EXTNAME=shared`` あるいは ``--with-EXTNAME=shared`` と明示的に指定することで共有オブジェクトを生成することも可能です。

静的なエクステンションはそのままで有効ですが、共有オブジェクトはiniファイルで ``extension`` あるいは ``zend_extension`` を使ってロードする必要があります。[1]_ どちらのオプションも ``.so`` ファイルの絶対パスか ``extension_dir`` の設定からの相対パスを受け取ります( ``zend_extension`` での相対パスはPHP 5.5からのみ有効で、それ以前は絶対パスでなければなりません)。


例として、次のようなconfigureのコマンドをみてみましょう。

.. code-block:: sh

  ~/php-src> ./configure --prefix=$HOME/myphp \
                       --enable-debug --enable-maintainer-zts \
                       --enable-opcache --with-gmp=shared

この場合、opcacheのエクステンションGMPのエクステンションがコンパイルされて共有オブジェクトが ``modules/`` に格納されます。``extension_dir`` を変更するか、絶対パスを渡すことでロードすることができます。

.. code-block:: sh

  ~/php-src> sapi/cli/php -dzend_extension=`pwd`/modules/opcache.so \
                        -dextension=`pwd`/modules/gmp.so
  # あるいは
  ~/php-src> sapi/cli/php -dextension_dir=`pwd`/modules \
                        -dzend_extension=opcache.so -dextension=gmp.so


``make install`` のステップで、両方の ``.so`` ファイルはPHPのインストール場所のエクステンションのディレクトリに移動されます。エクステンションのディレクトリは ``php-config --extension-dir`` で確認でき、上記のビルドオプションでは、 ``/home/myuser/myphp/lib/php/extensions/no-debug-non-zts-MODULE_API`` となるでしょう。この値はiniファイルでの ``extension_dir`` のデフォルトでもあるので、明示的に指定する必要はなく直接エクステンションをロードできます。

.. code-block:: sh

  ~/myphp> bin/php -dzend_extension=opcache.so -dextension=gmp.so

ここでひとつの疑問が残ります。静的ロードと共有オブジェクトのロードのどちらの仕組みを使うべきでしょうか？共有オブジェクトはベースとなるPHPバイナリをもつことができ、php.iniで追加的にロードすることが可能です。ディストリビューションでは基本のPHPパッケージを提供し、エクステンションは別のパッケージとして配布することでこの仕組みを利用しています。一方、PHPバイナリを自身でコンパイルする場合は、必要なエクステンションがわかっているでしょうからこの必要はありません。

大雑把に言えば、組み込みのエクステンションは静的にロードし、その他のものは共有オブジェクトをロードするようにします。理由としては単に、エクステンションを外部からビルドする方が一見して確認できて容易だからです。別の利点としては、PHPをビルドしなおさなくともエクステンションを更新できます。

.. [1]  通常のエクステンションとZendエクステンションの違いについては後ほど説明します。今のところはZendエクステンションはより低レベル(例えばopcacheやxdebug)なもので、ZendEngine自体の動作をフックしているという理解で十分です。

PECLからエクステンションをインストールする
-------------------------------------------

`PECL <https://pecl.php.net/>`_ (PHP Extension Community Library) では数多くのPHPエクステンションが提供されています。主なPHPのディストリビューションから除かれたエクステンションは大抵PECL上には残り続けます。同様に、現在はPHPに組み込まれているエクステンションも以前はPECLで配布されていたエクステンションだったのです。

PHPのビルドのconfigureの段階で、 ``--without-pear`` と指定しない限り、 ``make install`` でPEARの一部としてPECLがダウンロードされてインストールされます。``pecl`` スクリプトは ``$PREFIX/bin`` ディレクトリの中にあります。エクステンションのインストールは単に ``pecl install EXTNAME`` とするだけです。

.. code-block:: sh
  
  ~/myphp> bin/pecl install apcu-4.0.2

このコマンドでは `APCu <https://pecl.php.net/package/APCu>`_ がダウンロード、コンパイル、そしてインストールされます。結果、 ``apcu.so`` ファイルがエクステンションのディレクトリに格納され、iniオプションで ``extension=apcu.so`` とすればロードできます。

``pecl install`` はエンドユーザーにとって大変手軽ですが、エクステンションの開発者にとっては興味がありません。以下では、手動でエクステンションをビルドする方法である、PHPのソースツリーにインポートするやり方(静的ロードが可能になります)と外的にビルドするやり方(共有オブジェクトをロードする必要がある)の2つの方法を説明していきます。

PHPのソースツリーにエクステンションを追加する
----------------------------------------------

サードパーティ製のPHPエクステンションと組み込みのエクステンションには基本的に違いはありません。PHPのソースツリーの中にエクステンションをコピーするだけで、通常のビルド手順を実行することでエクステンションをビルドすることができます。APCuを例に試してみましょう。

まず初めに、PHPソースツリーの ``ext/EXTNAME`` のディレクトリにエクステンションのソースコードを配置する必要があります。エクステンションがGitから取得できるなら、 ``ext/`` の中でリポジトリからcloneするだけです。

.. code-block:: sh

  ~/php-src/ext> git clone https://github.com/krakjoe/apcu.git


あるいはソースコードのtarballをダウンロードして解凍するのでもよいです。

.. code-block:: sh

  /tmp> wget http://pecl.php.net/get/apcu-4.0.2.tgz
  /tmp> tar xzf apcu-4.0.2.tgz
  /tmp> mkdir ~/php-src/ext/apcu
  /tmp> cp -r apcu-4.0.2/. ~/php-src/ext/apcu

エクステンションは ``config.m4`` ファイルを含んでおり、autoconfでのエクステンション固有のビルド方法が記述されています。 ``./configure`` スクリプトに組み込むため、もう一度 ``./buildconf`` を実行しなければなりません。設定ファイルが確実に再生成されるのを保証するために、事前に削除しておくことをお勧めします。

.. code-block:: sh

  ~/php-src> rm configure && ./buildconf --force


``./config.nice`` スクリプトでAPCuを既存の設定ファイルに追加することもできますし、全体の設定をコマンドラインで指定しなおすのでもいいです。

.. code-block:: sh

  ~/php-src> ./config.nice --enable-apcu
  # または
  ~/php-src> ./configure --enable-apcu # --other-options

最後に　``make -jN`` を実行してビルドします。 ``--enable-apcu=shared`` を使ってないので、エクステンションはPHPバイナリに静的に関連付けられます。つまり、エクステンションを使うのに追加の作業はありません。当たり前ですがインストールするには ``make install`` を実行してください。

phpizeを使ってエクステンションをビルドする
-------------------------------------------

:doc:`/build_system/building_php` でも既に述べたように ``phpize`` スクリプトを使ってPHPとは分離してエクステンションをビルドすることも可能です。

``phpize`` はPHPのビルドにとっての ``./buildconf`` と同じような役割を果たします。つまり、まず ``$PREFIX/lib/php/build`` のファイルをコピーすることでPHPビルドシステムをエクステンションの中にインポートします。これらのファイルの中には ``acinclude.m4`` (PHP’s M4 macros)や、 ``phpize.m4`` (これは ``configure.in`` にリネームされメインのビルド手順を含んでいます)や ``run-tests.php`` などがあります。

``phpize`` はautoconfを呼び出してエクステンションのビルドをカスタマイズするのに使用される ``./configure`` ファイルを生成します。``--enable-apcu`` と指定する必要はありません。代わりに、 ``php-config`` スクリプトのパスを指定するため ``--with-php-config`` を使用する必要があります。

.. code-block:: sh

  /tmp/apcu-4.0.2> ~/myphp/bin/phpize
  Configuring for:
  PHP Api Version:         20121113
  Zend Module Api No:      20121113
  Zend Extension Api No:   220121113

  /tmp/apcu-4.0.2> ./configure --with-php-config=$HOME/myphp/bin/php-config
  /tmp/apcu-4.0.2> make -jN && make install

エクステンションをビルドする際は常に ``--with-php-config`` を指定すべきです(ただしPHPが1つだけインストールされている場合は除く)。さもないと、 ``./configure`` で正確にどのPHPに対してのバージョンやフラグを使うかを決めることができません。また ``php-config`` を指定することで、 ``make install`` で ``.so`` ファイル( ``modules/`` ディレクトリに格納されています)が正しいエクステンションのディレクトリに移動することを保証されます。

``run-tests.php`` ファイルもまた ``phpize`` の段階でコピーされますので、 ``make test`` (あるいはrun-tests.phpを明示的に実行)でエクステンションのテストを実行することができます。

コンパイル済みのオブジェクトを削除してくれる ``make clean`` もまた利用可能で、これによってエクステンションの全体をリビルドできるようになります。加えて、phpizeには ``phpize --clean`` という削除用のオプションがあります。これは ``/configure`` で生成されたファイル同様、 ``phpize`` によってインポートされたファイルを全て削除します。

エクステンションについての情報を表示する
------------------------------------------

PHPのCLIバイナリにはエクステンションについての情報を表示するためのいくつかのオプションがあります。既にふれた ``-m`` はロードされているエクステンションの全てを一覧表示します。これによってエクステンションが正常にロードされているかを検証することができます。

.. code-block:: sh

  ~/myphp/bin> ./php -dextension=apcu.so -m | grep apcu
  apcu

ある機能についての情報を表示するための ``--r`` からはじまるオプションがいくつかあります。例えば、 ``--ri`` を使えば、エクステンションの設定値が表示されます。

.. code-block:: sh

  ~/myphp/bin> ./php -dextension=apcu.so --ri apcu
  apcu  

  APCu Support => disabled
  Version => 4.0.2
  APCu Debugging => Disabled
  MMAP Support => Enabled
  MMAP File Mask =>
  Serialization Support => broken
  Revision => $Revision: 328290 $
  Build Date => Jan  1 2014 16:40:00  

  Directive => Local Value => Master Value
  apc.enabled => On => On
  apc.shm_segments => 1 => 1
  apc.shm_size => 32M => 32M
  apc.entries_hint => 4096 => 4096
  apc.gc_ttl => 3600 => 3600
  apc.ttl => 0 => 0
  # ...


``--re`` はエクステンションによって追加されたiniの設定値、定数、関数、クラスの全てが表示されます。

.. code-block:: sh

  ~/myphp/bin> ./php -dextension=apcu.so --re apcu
  Extension [ <persistent> extension #27 apcu version 4.0.2 ] {
    - INI {
      Entry [ apc.enabled <SYSTEM> ]
        Current = '1'
      }
      Entry [ apc.shm_segments <SYSTEM> ]
        Current = '1'
      }
      # ...
    }  

    - Constants [1] {
      Constant [ boolean APCU_APC_FULL_BC ] { 1 }
    }  

    - Functions {
      Function [ <internal:apcu> function apcu_cache_info ] {  

        - Parameters [2] {
          Parameter #0 [ <optional> $type ]
          Parameter #1 [ <optional> $limited ]
        }
      }
      # ...
    }
  }

``--re`` は通常のエクステンションのみで動作し、Zendエクステンションには代わりに ``--rz`` を使用します。opcacheで試してみましょう。

.. code-block:: sh

  ~/myphp/bin> ./php -dzend_extension=opcache.so --rz "Zend OPcache"
  Zend Extension [ Zend OPcache 7.0.3-dev Copyright (c) 1999-2013 by Zend Technologies <http://www.zend.com/> ]

ご覧のように、役に立つような情報が表示されていません。理由としてはopcacheは通常のエクステンションとZendエクステンションの両方を登録していますが、通常のエクステンションの方にiniの設定値や定数や関数などの全てが含まれているからです。そのため、このケースでは ``--re`` を使うべきでしょう。他のZendエクステンションでは ``--rz`` を通して情報を取得できます。
