PHPをビルドする
===============

この章ではPHPエクステンションの開発やPHPコアに変更を加える時に適した方法でのPHPのコンパイル方法を説明します。ここではUnixoid環境下でのビルドしか扱いません。Windowsでビルドしたい場合は、PHP wikiの `ビルド方法 <https://wiki.php.net/internals/windows/stepbystepbuild>`_ を参照してください。[1]_

.. [1] 免責条項: WindowsでのPHPコンパイルによって発生するいかなる健康上の悪影響に関して、私達は一切の責任を負いません。

またこの章ではPHPビルドシステムがどのように動作するかや、どんなツールを使用しているかについての概略の説明をしますが、詳細な部分については扱いません。

パッケージを使用しない理由
--------------------------

もし現在PHPを使用しているなら、パッケージマネージャーを通して ``sudo apt-get install php`` 等のコマンドでPHPをインストールしているかもしれません。
コンパイルの説明に入る前に、まずなぜ自分自身によるコンパイルが必要で、事前にパッケージとしてビルド済みのものを使うだけでは駄目なのかを理解する必要があります。これにはいくつかの理由があります。

第一に、ビルド済みのパッケージは結果としてのバイナリしか含まれず、エクステンションのコンパイルに必要なヘッダーファイル等のファイルが含まれていません。
これは ``php-dev`` としばしば呼ばれている開発用パッケージをインストールすることで解決できます。
valgrindやgdbなどを使用してデバッグを容易にするために、 ``php-dbg`` と呼ばれる別のパッケージを利用するとデバッグシンボル付きでインストールできます。

しかし、たとえヘッダーファイルやデバッグシンボルを含めてインストールしたとしても、PHPのリリースビルドを使用していることになります。
リリースビルドは高度に最適化されてビルドされておりますので、デバッグが非常に難しいことを意味しています。さらに、リリースビルドではメモリーリークやデータ構造の不整合が発生しても警告が出ないようになっています。またビルド済みパッケージでは、開発中にとても有効なスレッドセーフが有効となっていません。

別の問題として、ほとんどすべてのパッケージはPHPに追加のパッチが当てられています。これらのパッチは設定に関する小さな変更のこともあれば、Suhosinのようなかなり影響の大きいパッチもあります。
いくつかのパッチはopcacheのような低レベルのエクステンションと上手く適合しないものがあります。

PHPは `php.net <http://www.php.net/>`_ 上で提供されているバージョンのみをサポートし、変更が加わったディストリビューションのバージョンはサポートしません。もしバグレポートをおこなう場合や、パッチを送ったり、エクステンション開発のためにヘルプチャンネルを利用する場合等は、いつでもPHPの公式のバージョンに対しておこなうべきです。このドキュメントで"PHP"についてふれるときは、常に公式にサポートされているバージョンのものをさしています。


ソースコードの取得
------------------

PHPをビルドする前に、まずPHPのソースコードを手に入れる必要があります。ソースコードを手に入れるには2つの方法があります。 `PHP's download page <http://www.php.net/downloads.php>`_ からダウンロードするか、Gitリポジトリの `git.php.net <http://git.php.net/>`_ (あるいは `Github <https://github.com/php/php-src>`_ 上のミラー) からcloneできます。

両者の方法によって、ビルド方法がわずかに異なります。Gitリポジトリには ``configure`` スクリプトが含まれていませんので autoconfを利用している ``buildconf`` スクリプトを通して生成する必要があります。さらに、Gitリポジトリには生成済みのパーサーが含まれていないのでbisonをインストールする必要があります。

推奨の方法はGitリポジトリからソースコードを取得する方法です。なぜなら、バージョンを最新に維持したり、異なるバージョンで自身のコードを試すことも容易だからです。またパッチや、PullRequestをPHPに送る時にはGitからのCheckoutが必要になります。

リポジトリからcloneするためには下記のコマンドをshellで実行してください。 

.. code-block:: sh
  
  ~> git clone http://git.php.net/repository/php-src.git
  ~> cd php-src
  # デフォルトでは、現在開発がすすめられているmasterブランチになっています。
  # かわりに、安定したブランチをチェックアウトすることも出来ます。
  ~/php-src> git checkout PHP-5.5

GitからのCheckoutで問題がある場合は、PHP wikiにある `Git FAQ <https://wiki.php.net/vcs/gitfaq>`_ を参照してください。もしPHP開発に貢献したい場合、Git FAQはGit上でのセットアップ方法も記載してますので参考にしてください。さらに、異なるPHPバージョンでの複数の作業ディレクトリをセットアップするための方法も記載しています。これはエクステンションや修正などのテストを複数の異なるPHPのバージョンや設定上でテストするためにはとても有効です。


ビルドをおこなうにあたってパッケージマネージャーから下記の依存パッケージをインストールする必要があります(最初の3つはデフォルトでインストール済みかもしれません)。

 - ``gcc`` (あるいは他のコンパイラースイート)
 - ``libc-dev`` ヘッダーを含むCの標準ライブラリー
 - ``make`` PHPが使用しているビルド管理ツール
 - ``autoconf`` (2.59以上) ``configure`` スクリプトを生成するために使用
 - ``automake`` (1.4以上) ``Makefile.in`` を生成する
 - ``libtool`` 共有ライブラリー管理用
 - ``bison`` (2.4以上) PHPパーサーを生成するために使用
 - (オプション) ``re2c`` PHP lexerを生成するために使用。 Gitリポジトリにはすでにlexerが含まれているので、lexerの変更をしたい場合のみre2cをインストールする必要があります。

Debian/Ubuntuでは次のコマンドですべてインストールできます。

.. code-block:: sh

  ~/php-src> sudo apt-get install build-essential autoconf automake libtool bison re2c

有効にしているエクステンションに依存して、 ``./configure`` の段階でPHPはいくつかの追加のライブラリーを必要とします。追加でのライブラリーをインストール際は、そのパッケージに ``-dev`` あるいは ``-devel`` のものがあるかチェックして、それをインストールするようにしてください。 ``dev`` のついていないパッケージは大抵、必要なヘッダーファイルを含んでいません。例えば、デフォルトのPHPビルドではlibxmlが必要ですが、 ``libxml2-dev`` のパッケージからインストールできます。

DebianやUbuntuをお使いなら、 ``sudo apt-get build-dep php5`` で一度に沢山の依存パッケージをインストールできます。ただし、デフォルトでのビルドをおこなうだけであれば、実際必要でないものが多いです。

ビルド概略
-----------

ビルドの各ステップの詳細をみていく前に、下記のデフォルトでのPHPビルドを実行するためのコマンドを確認しましょう。

.. code-block:: sh

  ~/php-src> ./buildconf     # Gitから取得してビルドする場合のみ必要
  ~/php-src> ./configure
  ~/php-src> make -jN


ビルドを早くするために、``N`` を利用できるCPUのコア数に置き換えてください。(CPUコア数は ``grep "cpu cores" /proc/cpuinfo`` で確認できます)
デフォルトのPHPでは ``sapi/cli/php`` や ``sapi/cgi/php-cgi`` に配置されるCLIやCGI SAPIのバイナリをビルドします。ビルドが上手くいっているかを確認するために ``sapi/cli/php -v`` を実行してみてください。

また、``sudo make install`` と実行すれば、PHPを ``/usr/local`` にインストールすることができます。インストール先はconfigureの段階で ``--prefix`` のオプションを指定することで変更できます。 

.. code-block:: sh

  ~/php-src> ./configure --prefix=$HOME/myphp
  ~/php-src> make -jN
  ~/php-src> make install

上記の場合、``make install`` でのインストール先は ``$HOME/myphp`` となります。PHPのインストールは必須ではありませんが、インストールしておくと
、エクステンションの開発を除いて、ビルドしたPHPを使いたい場合に便利です。

ではビルドの各ステップの詳細をみていきましょう！

./buildconf スクリプト
----------------------

Gitリポジトリからビルドする場合、まず最初にする必要があるのは ``./buildconf`` スクリプトの実行です。このスクリプトがやることは ``build/build.mk`` ファイルを呼び出し、またそこから ``build/build2.mk`` が呼び出すという事にすぎません。

これらmakefilesの主な役割は ``./configure`` スクリプトを生成するための ``autoconf`` と、 ``main/php_config.h.in`` のテンプレートファイルを生成するための ``autoheader`` を実行することです。``main/php_config.h.in`` のファイルは最終的な設定のヘッダーファイルである ``main/php_config.h`` を生成するために使用されます。

これらのユーティリティはPHPのビルドプロセスの大半が記述されている ``configure.in`` ファイルから、 ``acinclude.m4`` ファイル（PHP特有のM4マクロが多く記述されたファイル）と 、個々のエクステンションやSAPIの ``config.m4`` ファイル(とその他の ``m4`` ファイル郡)を生成します。

朗報なのは、エクステンションを開発する時やPHPコアの部分を修正する時であっても、ビルドシステムに直接関係することは少ないということです。
後ほどちょっとした ``config.m4`` ファイルを書く必要がありますが、大抵 ``acinclude.m4`` が提供する高水準のマクロを２，３使うだけです。ここではこれ以上詳細にふれません。

``./buildconf`` スクリプトは２つだけオプションがあります。 ``--debug`` オプションはautoconfやautoheaderを呼び出す際に発生した警告を表示するようにします。ビルドシステム自身について開発する以外ではこのオプションは特に必要ないでしょう。
２つめのオプションは ``--force`` で、これはリリースパッケージ(例えばパッケージのソースコードをダウンロードして新しい ``./configure`` を生成したい場合など)で ``./buildconf`` の実行を許可し、``config.cache`` や ``autom4te.cache/`` の設定のキャッシュを削除します。

``git pull`` などでリポジトリを最新にして ``make`` の段階で奇妙なエラーが発生した場合、大抵はビルドの設定の何かが変更し ``./buildconf --force`` を実行する必要があることを意味します。


./configure スクリプト
----------------------
``./configure`` スクリプトが作成されると、PHPビルドをカスタマイズすることが可能になります。 ``--help`` オプションを使ってサポートしているオプションの全てを確認できます。

.. code-block:: sh
  ~/php-src> ./configure --help | less

helpで表示されるオプションの最初の部分はautoconfの設定ファイルによってサポートされている一般的なオプションが表示されるでしょう。前述した ``--prefix=DIR`` はこのオプションのうちのひとつで、 ``make install`` でのインストール先の変更をするためのものです。別の役に立つオプションで ``-C`` というものがあり、これは ``config.cache`` ファイルに結果をキャッシュし ``./configure`` の続けての実行スピードを速くします。このオプションは既にビルド済みの場合で、異なったビルド設定に素早く変更したい場合には有効です。

一般的なautoconfのオプションとは別にPHP特有の設定も多くあります。例えば、エクステンションやSAPIをコンパイルするかどうかを選択するためには ``--enable-NAME`` あるいは ``--disable-NAME`` を使用します。もしエクステンションやSAPIに依存ライブラリがある場合、 ``--with-NAME`` あるいは ``--without-NAME`` を使います。もし ``NAME`` で指定されたライブラリーがデフォルトの場所に存在しない場合(例えば自身でコンパイルした場合など)、 ``--with-NAME=DIR`` でライブラリーのファイルパスを指定することができます。

デフォルトのPHPでは、多くのエクステンションとCLIやCGI SAPIをビルドします。 ``-m`` を使用すればPHPバイナリに含まれているエクステンションが確認できます。デフォルトのPHP 5.5のビルドでは下記のような結果になります。 ::

  ~/php-src> sapi/cli/php -m
  [PHP Modules]
  Core
  ctype
  date
  dom
  ereg
  fileinfo
  filter
  hash
  iconv
  json
  libxml
  pcre
  PDO
  pdo_sqlite
  Phar
  posix
  Reflection
  session
  SimpleXML
  SPL
  sqlite3
  standard
  tokenizer
  xml
  xmlreader
  xmlwriter

CGI SAPI、tokenizerやsqlite3のエクステンションを無効とし、変わりにopcacheやgmpを有効にしたいという場合、configureコマンドは下記のようになります。

.. code-block:: sh

  ~/php-src> ./configure --disable-cgi --disable-tokenizer --without-sqlite3 \
                       --enable-opcache --with-gmp

デフォルトでは、ほとんどのエクステンションが静的にコンパイルされるでしょう。つまり結果としてのバイナリの一部となります。opcacheのエクステンションだけはデフォルトでは ``modules/`` ディレクトリの中に共有オブジェクト ``opcache.so`` が生成されます。他のエクステンションも ``--enable-NAME=shared`` あるいは ``--with-NAME=shared `` とすることで、同様に共有オブジェクトを生成することができます(しかし全てのエクステンションがこれをサポートしているわけではありません)。次のセクションでは共有オブジェクトをどのように利用するかについて見ていきましょう。

どちらのオプションスイッチを使うべきか、またデフォルトでエクステンションが有効となっているかどうかを確認するためには ``./configure --help`` を実行してください。オプションスイッチが ``--enable-NAME`` あるいは ``--with-NAME`` と表示されている場合は、デフォルトではそのエクステンションはコンパイルされず、明示的に有効にしなければいけないことを意味します。一方、 ``--disable-NAME`` あるいは ``--without-NAME`` と表示されている場合はデフォルトでコンパイルされ、明示的に無効とすることができることを意味します。

エクステンションの中には無効にすることができないものもあります。最低限のエクステンションでのみビルドしたい場合は ``--disable-all`` オプションを使用してください。

.. code-block:: sh

  ~/php-src> ./configure --disable-all && make -jN
  ~/php-src> sapi/cli/php -m
  [PHP Modules]
  Core
  date
  ereg
  pcre
  Reflection
  SPL
  standard

``--disable-all`` オプションはビルドを速くしたい場合や多くの機能(例えばPHP言語の改修をおこなう場合など)が必要ない場合は非常に有効です。最小のビルドにするためには ``--disable-cgi`` オプションを追加で指定し、CLIのバイナリのみ生成されるようにします。

エクステンションやPHPの開発をする上では、常に指定すべきもう2つのオプションがあります。

``--enable-debug`` オプションを指定してデバッグモードを有効になります。デバッグモードではコンパイルが ``-g`` オプション付きで実行され、デバッグシンボルが生成され、最適化レベルが最低である ``-O0`` が使用されるようになります。これによってPHPはかなり遅くなりますが、 ``gdb`` のようなツールを使用してのデバッグがより容易になります。さらに、デバッグモードでは ``ZEND_DEBUG`` マクロが定義され、これによりZendEngine上で様々なデバッグのためのヘルパーが有効になります。メモリーリークや不正なデータ構造の使用等が検出されるようになります。

``--enable-maintainer-zts`` オプションを有効にするとスレッドセーフになります。このオプションによって ``ZTS`` マクロが定義され、TSRM(thread-safe resource manage)の機能がPHPによって使用されるようになります。スレッドセーフなエクステンションを開発することはとてもシンプルですが、このオプションが有効になっているかだけ確認するようにしてください。さもなければ、きっと ``TSRMLS_*`` マクロを使う事を忘れてしまい、あなたのコードがスレッドセーフ環境下でビルドされなくなってしまうでしょう。

一方、これらのオプションは重大な速度低下を引き起こすので、ベンチマークでパフォーマンスの測定をする場合には使うべきではありません。

``--enable-debug`` や ``--enable-maintainer-zts`` はPHPバイナリのABIを変更するので、多くの関数に対して引数が追加されることに留意してください。デバッグモードでコンパイルされたエクステンションはリリースモードでビルドされたPHPバイナリには適合しません。同様に、スレッドセーフなエクステンションはスレッドセーフでないPHPビルドには適合しないです。

ABIが合わないため、 ``make install`` (そしてPECLのインストール)では共有エクステンションはオプションによってそれぞれ異なったディレクトリに格納されます。

 - ``$PREFIX/lib/php/extensions/no-debug-non-zts-API_NO`` ZTS無効でリリースビルド
 - ``$PREFIX/lib/php/extensions/debug-non-zts-API_NO`` ZTS無効でデバッグビルド
 - ``$PREFIX/lib/php/extensions/no-debug-zts-API_NO`` ZTS有効でリリースビルド
 - ``$PREFIX/lib/php/extensions/debug-zts-API_NO`` ZTS有効でデバッグビルド

上記の ``API_NO`` のプレースホルダは ``ZEND_MODULE_API_NO`` が参照され、これは単純な ``20100525`` のような日付となり内部的にAPIのバージョン番号のために使用されています。

多くの用途において上で述べた設定オプションで十分ですが、勿論 ``./configure`` にはhelpで確認できる、多くのオプションが存在します。

configure実行のためにオプションを指定するのとは別に、多くの環境変数によっても指定することもできます。configureのhelpの出力の最後の方に重要なものがいくつか記述されています( ``./configure --help | tail -25`` )。

例えば ``CC`` によって別のコンパイラーを使用したり、 ``CFLAGS`` でコンパイルで使用されるフラグを変更することが出来ます。

.. code-block:: sh

  ~/php-src> ./configure --disable-all CC=clang CFLAGS="-O3 -march=native"

上記の設定では、gccの代わりにclangを使用し、非常に高い最適化レベル( ``-O3 -march=native`` )を使用しています。

make と make install
---------------------

全ての設定が完了すれば ``make`` でコンパイルを実行できます。

.. code-block:: sh

 ~/php-src> make -jN    # N はコア数

この作業によって、有効になっているSAPI用のPHPバイナリ(デフォルトでは ``sapi/cli/php`` と ``sapi/cgi/php-cgi`` )と、 ``modules/`` ディレクトリにエクステンションの共有オブジェクトが生成されます。

さて、 ``make install`` を実行すれば ``/usr/local`` (デフォルト)あるいは ``--prefix`` オプションで指定した場所にPHPがインストールされます。

``make install`` がおこなう事は多くのファイルを新しい場所にコピーするにすぎません。configureの段階で ``--without-pear`` を指定しない限りはPEARをダウンロードされ、インストールされます。デフォルトでのPHPビルドでの結果、treeは下記になります。 ::


  > tree -L 3 -F ~/myphp  

  /home/myuser/myphp
  |-- bin
  |   |-- pear*
  |   |-- peardev*
  |   |-- pecl*
  |   |-- phar -> /home/myuser/myphp/bin/phar.phar*
  |   |-- phar.phar*
  |   |-- php*
  |   |-- php-cgi*
  |   |-- php-config*
  |   `-- phpize*
  |-- etc
  |   `-- pear.conf
  |-- include
  |   `-- php
  |       |-- ext/
  |       |-- include/
  |       |-- main/
  |       |-- sapi/
  |       |-- TSRM/
  |       `-- Zend/
  |-- lib
  |   `-- php
  |       |-- Archive/
  |       |-- build/
  |       |-- Console/
  |       |-- data/
  |       |-- doc/
  |       |-- OS/
  |       |-- PEAR/
  |       |-- PEAR5.php
  |       |-- pearcmd.php
  |       |-- PEAR.php
  |       |-- peclcmd.php
  |       |-- Structures/
  |       |-- System.php
  |       |-- test/
  |       `-- XML/
  `-- php
      `-- man
          `-- man1/

このディレクトリ構造の簡単な要点は次の通りです。

 - bin/ にはPHPバイナリ( ``php`` と ``php-cgi`` )と ``phpize`` と ``php-config`` のスクリプトが格納されています。またPEAR/PECLスクリプトのホームディレクトリでもあります。
 - etc/ には設定ファイルが格納されています。デフォルトでは *php.ini* ファイルはここに格納され **ない** ことに注意してください。
 - include/php にはヘッダーファイルがあり、これはエクステンションをビルドする際や何かしらのソフトウェアに組み込む際に必要となってきます。
 - lib/php　にはPEARファイルが格納されています。lib/php/build ディレクトリにはエクステンションをビルドするのに必要なファイルである、PHPのM4マクロを含む ``acinclude.m4`` といったファイルがあります。エクステンションをコンパイルすると、lib/php/extensionsに格納されます。
 - php/man お分かりのように、 ``php`` コマンドのためのmanページです。

前述したように、 *php.ini* ファイルはデフォルトでは etc/にはありません。PHPバイナリの ``--ini`` オプションを使用してphp.iniの場所を表示することができます。 ::

  ~/myphp/bin> ./php --ini
  Configuration File (php.ini) Path: /home/myuser/myphp/lib
  Loaded Configuration File:         (none)
  Scan for additional .ini files in: (none)
  Additional .ini files parsed:      (none)

ご覧のように、 *php.ini* ファイルのデフォルトのディレクトリは ``$PREFIX/etc`` (sysconfdir)よりもむしろ ``$PREFIX/lib`` (libdir) となっています。configureの ``--with-config-file-path=PATH`` オプションでデフォルトの *php.ini* の場所を変更することができます。

また ``make install`` は iniファイルを生成しないことにも注意してください。 *php.ini* ファイルを使用する場合は、自身で作成する必要があります。例えば、デフォルトでのdevelopment用の設定をコピーして作成してもよいでしょう。 ::

  ~/myphp/bin> cp ~/php-src/php.ini-development ~/myphp/lib/php.ini
  ~/myphp/bin> ./php --ini
  Configuration File (php.ini) Path: /home/myuser/myphp/lib
  Loaded Configuration File:         /home/myuser/myphp/lib/php.ini
  Scan for additional .ini files in: (none)
  Additional .ini files parsed:      (none)

PHPのバイナリとは別に bin/ ディレクトリには ``phpize`` と ``php-config`` という重要なスクリプトが格納されています。

``phpize`` はエクステンションにとっての ``./buildconf`` のようなものです。lib/php/build　から様々なファイルをコピーしてautoconf/autoheaderを呼び出します。次のセクションでより詳しくこのツールについて見ていきます。

``php-config`` はPHPビルドの設定についての情報を提供します。試しに実行してみましょう。

.. code-block:: sh

  ~/myphp/bin> ./php-config
  Usage: ./php-config [OPTION]
  Options:
    --prefix            [/home/myuser/myphp]
    --includes          [-I/home/myuser/myphp/include/php -I/home/myuser/myphp/include/php/main -I/home/myuser/myphp/include/php/TSRM -I/home/myuser/myphp/include/php/Zend -I/home/myuser/myphp/include/php/ext -I/home/myuser/myphp/include/php/ext/date/lib]
    --ldflags           [ -L/usr/lib/i386-linux-gnu]
    --libs              [-lcrypt   -lresolv -lcrypt -lrt -lrt -lm -ldl -lnsl  -lxml2 -lxml2 -lxml2 -lcrypt -lxml2 -lxml2 -lxml2 -lcrypt ]
    --extension-dir     [/home/myuser/myphp/lib/php/extensions/debug-zts-20100525]
    --include-dir       [/home/myuser/myphp/include/php]
    --man-dir           [/home/myuser/myphp/php/man]
    --php-binary        [/home/myuser/myphp/bin/php]
    --php-sapis         [ cli cgi]
    --configure-options [--prefix=/home/myuser/myphp --enable-debug --enable-maintainer-zts]
    --version           [5.4.16-dev]
    --vernum            [50416]

このスクリプトはlinuxのディストリビューションで使われる ``pkg-config`` スクリプトと似ています。エクステンションのビルドの際に呼び出され、コンパイラーのオプションやパスについての情報を取得するために使用されます。このスクリプトは、configureのオプションやデフォルトのエクステンションのディレクトリといった、ビルドについての情報を素早く得るためにも使用できます。これらの情報は ``./php -i`` (phpinfo)でも得られますが、 ``php-config`` はAutotoolsでより使いやすいように、簡潔な形式で出力してくれます。


テストスイートの実行
----------------------

``make`` コマンドの実行が成功して終了すると、次にように ``make test`` を実行するよう促すメッセージが表示されるでしょう。

.. code-block:: sh

  Build complete.
  Don't forget to run 'make test'


``make test`` を実行するとPHPのCLIバイナリに対してPHPのソースコードの tests/ディレクトリにあるテストスイートが実行されます。デフォルトのビルドでは約9000のテスト(この数は最小のビルドであれば少なくなりますし、追加でエクステンションを有効にすれば多くなるでしょう)が実行され、これには数分かかるでしょう。
``make test`` コマンドは並列実行でないので ``-jN`` オプションを指定したところで速くなるわけではありません。

もしPHPのコンパイルがその環境で初めてであれば、テストスイートの実行を推奨します。OSやビルド環境によっては、テスト実行によってPHPのバグが見つかる可能性があります。テストで失敗したものがあれば、開発者に失敗したテストを調査するのを可能にするよう、QAにバグレポートを送信するかどうかを尋ねられます。通常、少しのテスト失敗が出ることはよくあるので、かなり多くのテスト失敗が出ない限りはそのバイナリは問題なく動作するでしょう。

``make test`` コマンドは内部的にCLIバイナリを使って ``run-tests.php`` を実行しています。 ``sapi/cli/php run-tests.php --help`` と実行することで、このスクリプトのオプションが表示されます。

``run-tests.php`` を手動で実行する場合は、 ``-p`` あるいは ``-P`` オプション(あるいは環境変数)のどちらかを指定する必要があります。

.. code-block:: sh

  ~/php-src> sapi/cli/php run-tests.php -p `pwd`/sapi/cli/php
  ~/php-src> sapi/cli/php run-tests.php -P

``-p`` オプションはどのバイナリでテストを実行するかを明示的に指定するためのものです。全てのテストが正確に実行できるよう、絶対パスで指定する必要があるので注意してください。 ``-P`` オプションは ``run-test.php`` を実行しているそのバイナリをテストで使用するためのショートカットです。上記の例は両方とも同じ結果となります。

全てのテストスイートを実行するかわりに、``run-tests.php`` で引数を指定することで、特定のディレクトリのみに限定することができます。例えば、ZendEngine、リフレクションのエクステンション、arrayの関数のみに限定する場合は次の通りです。

.. code-block:: sh

  ~/php-src> sapi/cli/php run-tests.php -P Zend/ ext/reflection/ ext/standard/tests/array/


これは修正を加えた部分に関連するテストスイートのみを素早く実行できるため、とても有効な方法です。例えばPHPの言語部分に変更を加えた場合は、エクステンションのテストを気にする必要はなく、ZendEngineが正常に動いているかさえを検証すればよいからです。

このテストを限定するためのディレクトリの引数やオプションを渡すために ``run-tests.php`` を明示的に実行する必要はありません。代わりに、``make test`` を通して ``TESTS`` という変数を使うことができます。例えば、さきほどの例は次のようになります。

.. code-block:: sh
  
  ~/php-src> make test TESTS="Zend/ ext/reflection/ ext/standard/tests/array/"

``run-tests.php`` に関しては、自分でテストを書く方法やテスト失敗のデバッグ方法についてふれる部分で後ほど詳しく見ていくことにします。

コンパイルでの問題の解決方法とmake clean
----------------------------------------

ご存知のように、 ``make`` コマンドは都度全てのファイルをリコンパイルするわけではなく、前回のビルドから ``.c`` ファイルの中で変更があったファイルだけをコンパイルします。この仕組みはビルド時間の縮小のためには素晴らしい方法ですが、いつもこの方法で上手くいくとは限りません。例えば、ヘッダーファイルのある構造体を変更した場合、 ``make`` コマンドはそのヘッダーファイルを使用している ``.c`` ファイルを自動的にリコンパイルしてくれるわけではないので、結果ビルドが壊れてしまいます。

``make`` コマンドを実行中に奇妙なエラーが発生した場合や、出来上がったビルドは壊れている場合(例えば、 ``make test`` が最初のテストを実行するよりも前にエラーになる時など)は、 ``make clean`` を実行すべきです。これによってコンパイル済みのものが削除され、次回の ``make`` コマンドで全体がビルドされるようになります。

``./configure`` オプションを変更した場合には ``make clean`` を実行する事が必要な時があります。追加でエクステンションを有効にしただけであれば、変更のあったファイルだけのビルドで問題ありませんが、他のオプションを変更した場合等は全体ビルドが必要になるかもしれません。

より強力なものとして ``make distclean`` というコマンドがあります。これは通常のファイル削除に加え、 ``./configure`` コマンドの実行によって作成されたファイルの全てが元通りになります。また設定ファイル、Makefiles、設定のヘッダーファイルなど様々なファイルが削除されます。コマンド名は"cleans for distribution"を意味している通り、リリースマネージャーによって使われることがほとんどです。

``config.m4`` ファイルや他のPHPビルドシステムのファイルの変更によってコンパイルで問題がおこることがあります。こういったファイルを変更した場合は、 ``./buildconf`` スクリプトからやり直す必要があります。自らで変更を加えた場合にはそのことを覚えているでしょうが、 ``git pull`` (あるいはその他のリポジトリの更新コマンド)によって起こった場合は気づきにくいでしょう。

もし ``make clean`` でも解決しないコンパイルの問題が発生した場合は、 ``/buildconf --force`` を実行することで解決することがあります。前回の ``./configure`` オプションを後ほどタイプしなくとも、 ``./config.nice`` スクリプトを実行すれば前回と同じ ``./configure`` を実行できます。

.. code-block:: sh

  ~/php-src> make clean
  ~/php-src> ./buildconf --force
  ~/php-src> ./config.nice
  ~/php-src> make -jN

もうひとつファイル削除のスクリプトで ``./vcsclean`` というものがあります。これはGitからソースコードを取得した場合のみしか使えません。これはGit管理されていないファイルをまとめて削除するもので ``git clean -X -f -d`` をまとめたものです。このスクリプトは注意して使うようにしてください。
