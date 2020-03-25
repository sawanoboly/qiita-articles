

[Habitat - Automation That Travels with the App](https://www.habitat.sh/)というのがあります。

詳細は上の公式とか、Podcastでも話題にしたことがあるのでそれを。

[Track 2 GCP.AWSといわゆるSeverless、ほかHabitatの話](https://cloudinfra.audio/track-2-gcp-awsといわゆるseverless-ほかhabitatの話-のメモ-3966dc86ad43#.xt4645ict)

さて今回はHabitatの機能のうち、アプリケーションの起動・管理をせず、特定用途の実行環境のみアーカイブにまとめて使えるようにしてみるチャレンジのメモです。

出てくるコードはここに。 => https://github.com/sawanoboly/habitat-ruby-archive-example

## 背景と目的

- (もらいものかつ秘伝的な)Rubyのスクリプトを動かしたい
- 新規サーバ(インスタンス)起動時にgemとかをオンデマンドなインストールをしている

と、これだけでも結構ヒヤヒヤする状態のものを移植する必要がありました。

### なぜHabitatを試したのか

スクリプトが要求するRubygemsごとRubyを固めてしまいたいと考えた。

- 64bit Linuxならなんでもよいようにしたい
    - Habitat 0.8くらいからtarエクスポートがついた
- Omnibusは依存周りがわりと中途半端だし時間もかかる
- Habitatはビルド再現性(≒依存解決)へのこだわりがなんかスゴイ

Habitatはそれっぽいことにも向いているかもと思いました。

> もちろん多少極端な感じとなり、汎用性はイマイチです。
> たいていの場合ははBundlerでいいし、固めるにしても`bundle package --all`でRubyGemsを固めてしまえば十分な環境再現ができると思います。
> そもそも外部コマンドとか叩くケースもあると思うので。

そのあたりを踏まえた上でHabitatで[公式(?)パッケージとして用意されているRuby](https://app.habitat.sh/#/pkgs/core/ruby/2.3.1)を見てみます。

```
$ ldd `/hab/bin/hab pkg path core/ruby`/bin/ruby 
	linux-vdso.so.1 =>  (0x00007ffdc8dff000)
	libruby.so.2.3 => /hab/pkgs/core/ruby/2.3.1/20161102184006/lib/libruby.so.2.3 (0x00007f8699c8e000)
	libpthread.so.0 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libpthread.so.0 (0x00007f8699a70000)
	libdl.so.2 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libdl.so.2 (0x00007f869986c000)
	libcrypt.so.1 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libcrypt.so.1 (0x00007f8699634000)
	libm.so.6 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libm.so.6 (0x00007f8699335000)
	libc.so.6 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libc.so.6 (0x00007f8698f91000)
	/hab/pkgs/core/glibc/2.22/20160612063629/lib/ld-linux-x86-64.so.2 => /lib64/ld-linux-x86-64.so.2 (0x00007f869a178000)
```

なんというか、、とことんぼっちを追求しています。

一応、パッケージに含まれる実行可能ファイルは`hab pkg binlink`でリンクを好きなパスに書き出しておくことができます。

```
$ sudo /hab/bin/hab pkg binlink --dest /opt/local/bin core/ruby/2.3.1 ruby
» Symlinking ruby from core/ruby/2.3.1 into /opt/local/bin
★ Binary ruby from core/ruby/2.3.1/20161102184006 symlinked to /opt/local/bin/ruby

$ /opt/local/bin/ruby --copyright
ruby - Copyright (C) 1993-2016 Yukihiro Matsumoto
```

ついでにgemもリンクして、`GEM_PATHS`なんかも確認。

```
$ /opt/local/bin/gem env
RubyGems Environment:
  - RUBYGEMS VERSION: 2.6.8
  - RUBY VERSION: 2.3.1 (2016-04-26 patchlevel 112) [x86_64-linux]
  - INSTALLATION DIRECTORY: /hab/pkgs/core/ruby/2.3.1/20161102184006/lib/ruby/gems/2.3.0
  - USER INSTALLATION DIRECTORY: /home/vagrant/.gem/ruby/2.3.0
  - RUBY EXECUTABLE: /hab/pkgs/core/ruby/2.3.1/20161102184006/bin/ruby
  - EXECUTABLE DIRECTORY: /hab/pkgs/core/ruby/2.3.1/20161102184006/bin
  - SPEC CACHE DIRECTORY: /home/vagrant/.gem/specs
  - SYSTEM CONFIGURATION DIRECTORY: /hab/pkgs/core/ruby/2.3.1/20161102184006/etc
  - RUBYGEMS PLATFORMS:
    - ruby
    - x86_64-linux
  - GEM PATHS:
     - /hab/pkgs/core/ruby/2.3.1/20161102184006/lib/ruby/gems/2.3.0
     - /home/vagrant/.gem/ruby/2.3.0
  - GEM CONFIGURATION:
     - :update_sources => true
     - :verbose => true
     - :backtrace => false
     - :bulk_threshold => 1000
  - REMOTE SOURCES:
     - https://rubygems.org/
  - SHELL PATH:
     - /usr/local/bin
     - /bin
     - /usr/bin
     - /usr/local/sbin
     - /usr/sbin
     - /sbin
     - /home/vagrant/bin
```

### そのまま使うのは(ほぼ)駄目です

Pure Rubyのスクリプトを回すだけならこれでもよいです。しかしRubyGems、特に`native extension`を含むが必要となると。。。

GEM_HOME/GEM_PATH環境変数とbundlerを駆使すればなんとかできなくもないですが、Habitatのぼっち理論からして、あとからbundleとかは甘えのようです。あれだ、イミュータブルだ。

必要なライブラリを含んだパッケージを作ってみることにします。とれる方式は次の２つ。

- 公式のRubyプランをもとに改造パッケージをつくる
- RubyGemsのみのパッケージを作る

両方いってみよう。


## 公式のRubyプランをもとに改造パッケージをつくる

さて、まず手っ取り早いのは公開Rubyパッケージをを改変して新しいパッケージを作ることです。

- [core-plans/ruby](https://github.com/habitat-sh/core-plans/tree/master/ruby)


ファイルはこれだけ。公式のチュートリアルではアプリケーションを起動させたがるので色々と必要に思えますが、単純にビルド＆パッケージにするだけならこんなもんです。`plan.sh`だけしかないのも沢山あります。

```shell:
with_ruby/
├── GlobalSignRootCA.pem
├── patches
│   └── ruby-2_1_3-no-mkmf.patch
└── plan.sh
```

planもまあ、知れた感じで。

変数の役割などはドキュメントで => [Habitat plan syntax reference](https://www.habitat.sh/docs/reference/plan-syntax/)

```shell:plan.sh
pkg_name=ruby
pkg_origin=core
pkg_version=2.3.1
pkg_description="A dynamic, open source programming language with a focus on \
  simplicity and productivity. It has an elegant syntax that is natural to \
  read and easy to write."
pkg_license=("Ruby")
pkg_maintainer="The Habitat Maintainers <humans@habitat.sh>"
pkg_source=https://cache.ruby-lang.org/pub/${pkg_name}/${pkg_name}-${pkg_version}.tar.gz
pkg_upstream_url=https://www.ruby-lang.org/en/
pkg_filename=${pkg_name}-${pkg_version}.tar.gz
pkg_shasum=b87c738cb2032bf4920fef8e3864dc5cf8eae9d89d8d523ce0236945c5797dcd
pkg_deps=(core/glibc core/ncurses core/zlib core/openssl core/libyaml core/libffi)
pkg_build_deps=(core/coreutils core/diffutils core/patch core/make core/gcc core/sed)
pkg_lib_dirs=(lib)
pkg_include_dirs=(include)
pkg_bin_dirs=(bin)
pkg_interpreters=(bin/ruby)

do_build() {
    CFLAGS="${CFLAGS} -O3 -g -pipe"
    patch -p1 -i "$PLAN_CONTEXT/patches/ruby-2_1_3-no-mkmf.patch"

    # Resolves issue for older versions of RubyGems which require new trust
    # authority SSL certificate required as of 2016-10-06.
    #
    # Most likely the next Ruby release will resolve this issue as the vendored
    # version of RubyGems should be newer.
    #
    # For more details see:
    # http://guides.rubygems.org/ssl-certificate-update/#manual-solution-to-ssl-issue
    cp -v "$PLAN_CONTEXT/GlobalSignRootCA.pem" lib/rubygems/ssl_certs/

    ./configure "--prefix=$pkg_prefix" \
                --enable-shared \
                --disable-install-doc \
                "--with-openssl-dir=$(_resolve_dependency core/openssl)" \
                "--with-libyaml-dir=$(_resolve_dependency core/libyaml)"
    make
}

do_install() {
  do_default_install
  gem update --system --no-document
  gem install rb-readline --no-document
}

do_check() {
  make test
}
```

だいたい何をやっているかはわかりますね。

### プランを改造する

環境はmacOS `10.12.1`, habitatは`0.13.1`を使ってます。

RubyGemsを管理する場合、`gem install`で全てバージョン指定... とは流石にやりたくないので、`Gemfile.lock`を使いました。

手元のRubyで適当につくりましょう。これは某環境で必要なRubyGemsを再現のため列挙したGemfileです。

```ruby:Gemfile
# frozen_string_literal: true
# A sample Gemfile
source "https://rubygems.org"

gem 'docker-api'
gem 'filewatcher'
gem 'aws-sdk'
gem 'sys-filesystem'
```

いくつかnative extentionを作るやつが入ってますね。lockを作ってプランのディレクトリに混ぜます。

```shell:
with_ruby/
├── Gemfile          # 追加
├── Gemfile.lock     # 追加(どこかで作る)
├── GlobalSignRootCA.pem
├── patches
│   └── ruby-2_1_3-no-mkmf.patch
└── plan.sh
```


まず`pkg_origin`, `pkg_maintainer`を自分の情報に書き換えます。(※`core/ruby`の流用planではpkg_nameを変更すると幾つか不都合がでてくるのでrubyのままにしてます。

そして、do_installにちょいと追加します。

```
do_install() {
  do_default_install
  gem update --system --no-document
  gem install rb-readline --no-document
  gem install bundler --no-document
  BUNDLE_GEMFILE="$PLAN_CONTEXT/Gemfile" bundle install
}
```

ほかにnative extentionで必要なライブラリの追加があれば、`pkg_build_deps`(ビルド時のみ必要)または`pkg_deps`(実行時にそのSharedObjectが必要)を追加します。たいていcoreにあるかと思います。
この例ではlibffiが必要ですが、既に入っているので変更なしです。


### パッケージをビルドする

オリジン/パッケージは`sawanoboly/ruby`となります。
で、hab studioに入る。`studio`はmacならDocker、Linuxならcgroupやnamespaceを駆使したコンテナ環境だ。

```
$ hab studio enter
0.13.1: Pulling from studio
Digest: sha256:7584882a621ff81b3faa99a4ada54915298b39613a4a7e6b0f6bc1b7a793536a
Status: Image is up to date for habitat-docker-registry.bintray.io/studio:0.13.1
   hab-studio: Creating Studio at /hab/studios/src (default)
   hab-studio: Importing sawanoboly secret origin key
» Importing origin key from standard input
★ Imported secret origin key sawanoboly-20161114083512.
   hab-studio: Entering Studio at /hab/studios/src (default)
   hab-studio: Exported: HAB_ORIGIN=sawanoboly

[1][default:/src:0]# pwd
/src

## カレントディレクトリをマウントしている
[2][default:/src:0]# Gemfile  Gemfile.lock  GlobalSignRootCA.pem  patches  plan.sh
```

ちなみに、プロンプトでパスのすぐ右にある数字は直前のコマンドのexit_statusのようです。

プランからパッケージを作るには`build`(studioの外から`hab pkg build`でもOK。)コマンドです。

> ビルド時、ホストの名前解決でコケたりすることがあるようで、そんなときはstudio環境内の`/etc/resolv.conf`で他の有効なリゾルバを向ければよいです。

```shell:hab_studio_build
# build
   : Loading /src/plan.sh
   ruby4myscript: Plan loaded
   ruby4myscript: Validating plan metadata
   ruby4myscript: hab-plan-build setup
   ruby4myscript: Using HAB_BIN=/hab/pkgs/core/hab/0.13.1/20161114235527/bin/hab for installs, signing, and hashing
   ruby4myscript: Resolving dependencies


... ## 依存関係にあるパッケージたちをダウンロードしていく


   ruby: hab-plan-build cleanup
   ruby: 
   ruby: Source Cache: /hab/cache/src/ruby-2.3.1
   ruby: Installed Path: /hab/pkgs/sawanoboly/ruby/2.3.1/20161115062140
   ruby: Artifact: /src/results/sawanoboly-ruby-2.3.1-20161115062140-x86_64-linux.hart
   ruby: Build Report: /src/results/last_build.env
   ruby: SHA256 Checksum: 76c313df25c12cd05e160072c89c2291102a8c2ba0250a5a072e7754a314663c
   ruby: Blake2b Checksum: e63676e4ac6452f1eded5c2932e06b5aa0702c766fc0a9aad8f6e72329e05431
   ruby: 
   ruby: I love it when a plan.sh comes together.
   ruby: 
   ruby: Build time: 5m59s
```

ビルドが済んだら、`hab pkg path sawanoboly/ruby`とするとパスがわかります。

```shell:
[13][default:/src:0]# hab pkg path sawanoboly/ruby
/hab/pkgs/sawanoboly/ruby/2.3.1/20161115062140
```

パッケージに含まれるメタ情報として、例えばFILESを見てみます。

```shell:
[12][default:/src:0]# cat `hab pkg path sawanoboly/ruby`/FILES | grep aws-sdk | head -n 3
88e39dfae824756eddfd745d4955beb18b23e1caa3e0a28964f00103409ea79e  /hab/pkgs/sawanoboly/ruby/2.3.1/20161115062140/lib/ruby/gems/2.3.0/cache/aws-sdk-2.6.19.gem
a5974ca042b7bd636f48d7ef64534b452e2ea5c54806088b4428cb34e32e76c8  /hab/pkgs/sawanoboly/ruby/2.3.1/20161115062140/lib/ruby/gems/2.3.0/cache/aws-sdk-core-2.6.19.gem
1f5aa802301b5b1602148353089f20a706f8d711e871e8d71ea39a43b29fa25d  /hab/pkgs/sawanoboly/ruby/2.3.1/20161115062140/lib/ruby/gems/2.3.0/cache/aws-sdk-resources-2.6.19.gem
```

全てのファイルにハッシュがついています。また、この時使った依存のライブラリたちのパッケージについているタイムスタンプも保存されています。
手順と成果物の再現性にこだわりを見せている雰囲気です。


### パッケージをtarアーカイブとして保存する

`hab pkg export tar {ORIGIN/PKG}`とすることで、環境一式のアーカイブを作成できます。

```
[26][default:/src:0]# hab pkg export tar sawanoboly/ruby
∵ Missing package for core/hab-pkg-tarize
» Installing core/hab-pkg-tarize
↓ Downloading core/hab-pkg-tarize/0.13.1/20161115003912
    3.79 KB / 3.79 KB / [================================================================================================================================================================================================================] 100.00 % 33.83 MB/s  
→ Using core/acl/2.2.52/20161031042300
→ Using core/attr/2.4.47/20161031042251
→ Using core/bash/4.3.42/20161102154320

... ## 専用の処理が色々走り、habコマンド(およびアプリケーションスタック環境用)などのファイルを含めてアーカイブができあがる。

✓ Installed core/libsodium/1.0.8/20161102180731
✓ Installed core/xz/5.2.2/20161031043427
✓ Installed core/hab-sup/0.13.1/20161115003952
★ Install of core/hab-sup/0.13.1/20161115003952 complete with 6 new packages installed.
» Symlinking hab from core/hab into /tmp/hab-pkg-tarize-KoMG/hab/bin
★ Binary hab from core/hab/0.13.1/20161114235527 symlinked to /tmp/hab-pkg-tarize-KoMG/hab/bin/hab
» Symlinking bash from core/busybox-static into /tmp/hab-pkg-tarize-KoMG/bin
★ Binary bash from core/busybox-static/1.24.2/20161102170221 symlinked to /tmp/hab-pkg-tarize-KoMG/bin/bash
» Symlinking sh from core/busybox-static into /tmp/hab-pkg-tarize-KoMG/bin
★ Binary sh from core/busybox-static/1.24.2/20161102170221 symlinked to /tmp/hab-pkg-tarize-KoMG/bin/sh
```

出来上がったアーカイブはタイムスタンプが付いて、一点ものっぽさがあります。

```
[28][default:/src:0]# ls sawanoboly-ruby-2.3.1-20161115062140.tar.gz 
sawanoboly-ruby-2.3.1-20161115062140.tar.gz


[29][default:/src:0]# du -sh sawanoboly-ruby-2.3.1-20161115062140.tar.gz 
72M	sawanoboly-ruby-2.3.1-20161115062140.tar.gz
```

サイズもまあまあ常識の範囲内です。


### できたパッケージをLinuxサーバにインストールしてみる

とりあえずTest-KitchenでCentOS6でも作ってみます。`synced_folders`を設定しておけばできた先から放り込めるので楽かな。

```yaml:.kitchen.yml
...
platforms:
  - name: centos-6.8
    driver:
      synced_folders:
        - ["ext_tools", "/opt/ext_tools"]
...
```

ということで、この段落のシェル出力はVMのCentOS6です。

tarを`-C /`で展開します。このパスでないと動きません。
すらハブに独自のディストリビューションを突っ込む感じですね。

```shell:CentOS6
(VM)$ tree -L 3 -d /hab/
/hab/
├── bin
└── pkgs
    ├── core
    │   ├── acl
    │   ├── attr
    │   ├── busybox-static
    │   ├── bzip2
    │   ├── cacerts
    │   ├── coreutils
    │   ├── gcc-libs
    │   ├── glibc
    │   ├── gmp
    │   ├── grep
    │   ├── hab
    │   ├── hab-sup
    │   ├── libarchive
    │   ├── libcap
    │   ├── libffi
    │   ├── libsodium
    │   ├── libtool
    │   ├── libyaml
    │   ├── linux-headers
    │   ├── ncurses
    │   ├── openssl
    │   ├── pcre
    │   ├── sed
    │   ├── xz
    │   └── zlib
    └── sawanoboly
        └── ruby

30 directories
```

Habitatまわりのオペレーションは`/hab/bin/hab`で行います。PATHを通す程のものでは無いかな。。？

```
(VM)$ ldd /hab/bin/hab
	not a dynamic executable


(VM)$ /hab/bin/hab --version
hab 0.13.1/20161114235527
```

このhabバイナリを使って、LinuxでもStudioを作ることもできますが、それはまた今度。

#### 実行ファイルのリンクを作成し、実行してみる

ざっくりbinのしたを全てリンクにしてみます。`hab pkg binlink`をつかいましょう。もちろんPATHをそこに向けても構いませんが。

```shell:hab_pkg_binlink
(VM)$ ls `/hab/bin/hab pkg path sawanoboly/ruby`/bin
aws.rb  bundle  bundler  erb  filewatcher  gem  irb  rake  rdoc  ri  ruby  update_rubygems

(VM)$ for x in `ls $(/hab/bin/hab pkg path sawanoboly/ruby)/bin` ; do sudo /hab/bin/hab pkg binlink --dest /opt/local/bin sawanoboly/ruby $x ; done

» Symlinking aws.rb from sawanoboly/ruby into /opt/local/bin
Ω Creating parent directory /opt/local/bin
★ Binary aws.rb from sawanoboly/ruby/2.3.1/20161115062140 symlinked to /opt/local/bin/aws.rb
» Symlinking bundle from sawanoboly/ruby into /opt/local/bin

... (省略)

★ Binary ruby from sawanoboly/ruby/2.3.1/20161115062140 symlinked to /opt/local/bin/ruby
» Symlinking update_rubygems from sawanoboly/ruby into /opt/local/bin
★ Binary update_rubygems from sawanoboly/ruby/2.3.1/20161115062140 symlinked to /opt/local/bin/update_rubygems
```

外部のRubyGemとか、なんか気難しい感じのRubyGemもrequireできてるね。

```
$ /opt/local/bin/ruby -raws-sdk -e 'puts Aws::VERSION'
2.6.19


$ /opt/local/bin/ruby -rnet/https -e 'puts OpenSSL::VERSION'
1.1.0
```


さて、binの下をまるごとリンクにして使おうと思ったら、実行可能ファイルを含むRubyGemではshbangがイマイチになってます。

```shell:
## これいいやつ
$ head -n1 /opt/local/bin/erb 
#!/hab/pkgs/sawanoboly/ruby/2.3.1/20161115062140/bin/ruby

## 動かないやつ
$ head -n1 /opt/local/bin/filewatcher 
#! ruby

### shbangがヘンである

$ /opt/local/bin/filewatcher --help
-bash: /opt/local/bin/filewatcher: ruby: bad interpreter: No such file or directory
```

少し調整がいるようだ。


### shbangを調整する

Railsのサンプルを参考にしてみると、shbangに色々工夫をして乗り切るような事をしている模様だった。

[core-plans/ruby-rails-sample](https://github.com/habitat-sh/core-plans/tree/master/ruby-rails-sample)

へたにbinstubしてBundlerコンテキストになってもそれはそれで困る。`/usr/bin/env`の取扱いだけ真似してみる。
`do_prepare`をそのまま使って、さらに`do_install()`に少し処理を追加。

```
do_install() {
  do_default_install
  gem update --system --no-document
  gem install rb-readline --no-document
  gem install bundler --no-document
  if [ -d ./.bundle ] ; then rm -rf ./bundle ; fi
  BUNDLE_GEMFILE="$PLAN_CONTEXT/Gemfile" bundle install

  for binexec in ${pkg_prefix}/bin/*; do
    if [ "$binexec" == "${pkg_prefix}/bin/ruby" ] ; then continue ; fi
    build_line "Setting shebang for ${binexec} to 'ruby'"
    [[ -f $binexec ]] && sed -e "s#/usr/bin/env ruby#${pkg_prefix}/bin/ruby#" -i $binexec
  done

  if [[ `readlink /usr/bin/env` = "$(pkg_path_for coreutils)/bin/env" ]]; then
    build_line "Removing the symlink we created for '/usr/bin/env'"
    rm /usr/bin/env
  fi
}
```


この辺をちょっと試してみるために、プランの中で`attach`を使った。 [Debugging Plans](https://www.habitat.sh/docs/create-packages-debugging/)

> 余談:
> ちなみに、ここで出て来る`sed`でさえ専用にビルドしたバイナリを使います。
> Studio内ではPATHがえらいことになってる。
> 
> ```
> # echo $PATH | tr ':' '\n'
> /hab/bin
> 
> /hab/pkgs/core/hab-plan-build/0.13.1/20161115003556/bin
> /hab/pkgs/core/diffutils/3.3/20161031043343/bin
> /hab/pkgs/core/less/481/20161102154500/bin
> /hab/pkgs/core/make/4.2.1/20161102154828/bin
> /hab/pkgs/core/mg/20160118/20161102210444/bin
> /hab/pkgs/core/util-linux/2.27.1/20161102155008/bin
> /hab/pkgs/core/vim/8.0.0004/20161102192849/bin
> /hab/pkgs/core/ncurses/6.0/20161102154037/bin
> /hab/pkgs/core/acl/2.2.52/20161031042300/bin
> /hab/pkgs/core/attr/2.4.47/20161031042251/bin
> /hab/pkgs/core/bash/4.3.42/20161102154320/bin
> /hab/pkgs/core/binutils/2.25.1/20161031031252/bin
> /hab/pkgs/core/bzip2/1.0.6/20161031042910/bin
> ...
> ```


で、これで作られた実行可能ファイルはこのようになります。

```
$ head -n 2 /hab/pkgs/sawanoboly/ruby/2.3.1/20161115112245/bin/filewatcher
#!/hab/pkgs/sawanoboly/ruby/2.3.1/20161115112245/bin/ruby
#
```

これなら単品で動くかな？ 再度`pkg export`を行い、CentOSに展開してみる。
`hab pkg path`は特に指定がなければ最新のパッケージを指すので、binlinkももう一度実行でよい。

OK, 動いたわー。

```
$ /opt/local/bin/filewatcher -v
filewatcher, version 0.5.3 by Thomas Flemming 2015
```

作成時に少々ややこしい処理が必要ですが、なんとか作れたと思います。


## RubyGemsのみのパッケージを作る場合(ダイジェスト)

前述の`core/ruby`を改造する場合に気になる点として。

- Rubyのビルドに時間がかかる
- 折角`core/ruby`があるので、それの依存で別パッケージにしたい

というようなことも思うじゃないですか。

これも一応可能でした。だいたい次の点に注意です。

- 実行時、`GEM_PATH`には`core/ruby`と作成したrubygemsのパッケージパスを両方指定する
- native extentionから必要なShared Objectがある場合、`pkg_deps`に入れておかないとちゃんとリンクしてくれない。

実行時にも取り回し注意、となります。ただGEM_PATH次第でGemsetを切り替える感じになるので、神経質なBundle Packagerとして使えなくもないかなと思います。


### パッケージを作る

オリジン/パッケージを`sawanoboly/rubygems4myscript`として、buildします。こちらの方式はRubyをビルドしなくて済むのでattachして色々確認とかがやりやすくはなります。

他パッケージへの依存はこのくらい。

```
pkg_deps=(
  core/glibc
  core/ruby
  core/libffi
  core/bundler
)

pkg_build_deps=(
  core/gcc
  core/make
  core/coreutils
  )
```

これも [core-plans/ruby-rails-sample](https://github.com/habitat-sh/core-plans/tree/master/ruby-rails-sample) をだいたい参考にしつつ、先程のRuby改造版を踏襲して作成。

shbangが`pkg_path_for`ヘルパーを使っていたり、RubyGemsが`core/ruby`側に入っていかないように`GEM_HOME`を自分のパッケージを指定しています。

```shell:plan.sh
...

do_install() {
  export CPPFLAGS="${CPPFLAGS} ${CFLAGS}"
  local _bundler_dir=$(pkg_path_for bundler)

  export GEM_HOME=${pkg_prefix}
  export GEM_PATH=${_bundler_dir}:${GEM_HOME}

  BUNDLE_GEMFILE="$PLAN_CONTEXT/Gemfile" bundle install

  for binexec in ${pkg_prefix}/bin/*; do
    build_line "Setting shebang for ${binexec} to 'core/ruby'"
    [[ -f $binexec ]] && sed -e "s#/usr/bin/env ruby#$(pkg_path_for core/ruby)/bin/ruby#" -i $binexec
  done

  if [[ `readlink /usr/bin/env` = "$(pkg_path_for coreutils)/bin/env" ]]; then
    build_line "Removing the symlink we created for '/usr/bin/env'"
    rm /usr/bin/env
  fi
}

...
```

こんなツリーができます。とてもBundlerです。

```
$ tree -L 5 -d /hab/pkgs/sawanoboly/
/hab/pkgs/sawanoboly/
└── rubygems4myscript
    └── 2.3.1
        └── 20161115120237
            ├── bin
            ├── build_info
            ├── cache
            ├── doc
            ├── extensions
            │   └── x86_64-linux
            ├── gems
            │   ├── aws-sdk-2.6.19
            │   ├── aws-sdk-core-2.6.19
            │   ├── aws-sdk-resources-2.6.19
            │   ├── docker-api-1.32.1
            │   ├── excon-0.54.0
            │   ├── ffi-1.9.14
            │   ├── filewatcher-0.5.3
            │   ├── jmespath-1.3.1
            │   ├── json-2.0.2
            │   ├── sys-filesystem-1.1.7
            │   └── trollop-2.1.2
            └── specifications

22 directories
```

Linuxサーバにインストールして、binlinkを作りました。

```
$ for x in `ls $(/hab/bin/hab pkg path sawanoboly/rubygems4myscript)/bin` ; do sudo /hab/bin/hab pkg binlink --dest /opt/local/bin sawanoboly/rubygems4myscript $x ; done

» Symlinking aws.rb from sawanoboly/rubygems4myscript into /opt/local/bin
★ Binary aws.rb from sawanoboly/rubygems4myscript/2.3.1/20161115120237 symlinked to /opt/local/bin/aws.rb
» Symlinking filewatcher from sawanoboly/rubygems4myscript into /opt/local/bin
★ Binary filewatcher from sawanoboly/rubygems4myscript/2.3.1/20161115120237 symlinked to /opt/local/bin/filewatcher
```

実行してみると、まずはコケます。`core/ruby`本体が持つデフォルトのGEM_PATHと合わないからですね。

```
$ /opt/local/bin/filewatcher 
/hab/pkgs/core/ruby/2.3.1/20161102184006/lib/ruby/site_ruby/2.3.0/rubygems.rb:270:in `find_spec_for_exe': can't find gem filewatcher (>= 0.a) (Gem::GemNotFoundException)
	from /hab/pkgs/core/ruby/2.3.1/20161102184006/lib/ruby/site_ruby/2.3.0/rubygems.rb:298:in `activate_bin_path'
	from /opt/local/bin/filewatcher:22:in `<main>'
```

このパッケージ`sawanoboly/rubygems4myscript`と`core/ruby`に`GEM_PATH`を通したらようやく実行できます。

```
$ GEM_PATH=`/hab/bin/hab pkg path sawanoboly/rubygems4myscript`:`/hab/bin/hab pkg path core/ruby` /opt/local/bin/filewatcher -v
filewatcher, version 0.5.3 by Thomas Flemming 2015
```

`pkg_deps`の指定が足りないと、ここで`Not Found`がずらっと並んでしまいます。

```
$ find `/hab/bin/hab pkg path sawanoboly/rubygems4myscript` -name '*.so' | xargs ldd
/hab/pkgs/sawanoboly/rubygems4myscript/2.3.1/20161115121308/gems/ffi-1.9.14/lib/ffi_c.so:
	linux-vdso.so.1 =>  (0x00007ffd8a58e000)
	libruby.so.2.3 => /hab/pkgs/core/ruby/2.3.1/20161102184006/lib/libruby.so.2.3 (0x00007fbc33a8f000)
	libffi.so.6 => /hab/pkgs/core/libffi/3.2.1/20161102160809/lib/libffi.so.6 (0x00007fbc33881000)
	libpthread.so.0 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libpthread.so.0 (0x00007fbc33664000)
	libdl.so.2 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libdl.so.2 (0x00007fbc33460000)
	libcrypt.so.1 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libcrypt.so.1 (0x00007fbc33227000)
	libm.so.6 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libm.so.6 (0x00007fbc32f29000)
	libc.so.6 => /hab/pkgs/core/glibc/2.22/20160612063629/lib/libc.so.6 (0x00007fbc32b85000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fbc3419c000)
...
```

## 非対話でやりたい場合

habitatはLinuxシステム上で動かすのが一番シームレスです。ほかではそのまま実行する場合に一部の機能に制限があります。

一応、`hab studio run`を経由すれば、例えばmacOSからでもフル機能を使うことができました。

こんな感じ。`hab studio run hab pkg export tar core/ruby`


ということでこんなスクリプトを用意して、

```shell:build.sh
build
hab pkg export tar sawanoboly/rubygems4myscript
```

`hab studio run`で実行すればtar.gzがカレントディレクトリに出来上がります。

```shell:
$ hab studio run ./build.sh
```


## おわりに

habitatはアプリケーションスタックのコントロールまで行うプロダクトのようなんですが、こんなのもアリかなと。
(多分)パッケージをビルドするときにできる`*.hart`を使えば全く同じ環境がとことん再現できるのはスゴイを通り越して少々あきれるくらいです。
大々的に使う予定は無いけども、たまに出せるツールとして覚えておこうと思います。

手堅く行こうと思えばどこまでも手堅くできる、そんなhabitatでした。
