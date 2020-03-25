
先日Googleから [Container Structure Tests: Unit Tests for Docker Images | Google Open Source Blog](https://opensource.googleblog.com/2018/01/container-structure-tests-unit-tests.html) というお話が出ていました。

やってみよう。

テストできるのは次のパターン。

- コマンド実行結果
- ファイルの場所とパーミッション
- ファイルの中身
- コンテナイメージのメタデータ(EXPOSEとかWORKDIRなど)
- 含まれるOSSのライセンス (GCP上で動かしてもよいかのチェックなのかな？とりあえずdebianのみ対応)

内容からして、ファイルの記述が正しければ動作も正しいだろう、という観点で使うもののようですね。 Structureっつってんだからそういうもんでしょう。

## MacOSでのセットアップ

配布しているバイナリはLinuxでのみ動くよ、とのことです。コンテナを配布しているのでそっちを使ってみましょう。

コンテナはこちらに。[gcr.io/gcp-runtimes/container-structure-test](https://console.cloud.google.com/gcr/images/gcp-runtimes/GLOBAL/container-structure-test)

```bash:
$ docker pull gcr.io/gcp-runtimes/container-structure-test:v0.1.3
v0.1.3: Pulling from gcp-runtimes/container-structure-test
cd2f5b7886e9: Pull complete 
9be56f236f9a: Pull complete 
Digest: sha256:15ae6f3996ebc05b657a042bd5dd8f4d4d66b015effea5fd024c70b8a1f9c42a
Status: Downloaded newer image for gcr.io/gcp-runtimes/container-structure-test:v0.1.3
```

ヘルプを出してみます。

```bash:
$ docker run gcr.io/gcp-runtimes/container-structure-test:v0.1.3 -h
Usage of /go_default_test:
  -driver string
    	driver to use when running tests (default "docker")
  -image string
    	path to test image
  -pull
    	force a pull of the image before running tests
  -save
    	preserve created containers after test run
  -test.bench regexp
    	run only benchmarks matching regexp
  -test.benchmem
    	print memory allocations for benchmarks
  -test.benchtime d
    	run each benchmark for duration d (default 1s)
  -test.blockprofile file
    	write a goroutine blocking profile to file
  -test.blockprofilerate rate
    	set blocking profile rate (see runtime.SetBlockProfileRate) (default 1)
  -test.count n
    	run tests and benchmarks n times (default 1)
  -test.coverprofile file
    	write a coverage profile to file
  -test.cpu list
    	comma-separated list of cpu counts to run each test with
  -test.cpuprofile file
    	write a cpu profile to file
  -test.memprofile file
    	write a memory profile to file
  -test.memprofilerate rate
    	set memory profiling rate (see runtime.MemProfileRate)
  -test.mutexprofile string
    	write a mutex contention profile to the named file after execution
  -test.mutexprofilefraction int
    	if >= 0, calls runtime.SetMutexProfileFraction() (default 1)
  -test.outputdir dir
    	write profiles to dir
  -test.parallel n
    	run at most n tests in parallel (default 3)
  -test.run regexp
    	run only tests and examples matching regexp
  -test.short
    	run smaller test suite to save time
  -test.timeout d
    	fail test binary execution after duration d (0 means unlimited)
  -test.trace file
    	write an execution trace to file
  -test.v
    	verbose: print additional output
```

よしよし。


## サンプルのテストを実行する

リポジトリにテスト用のファイルがあるので、それを使ってみましょう。[container-structure-test/tests/debian_test.yaml](https://github.com/GoogleCloudPlatform/container-structure-test/blob/79b5f1e0e7eabd7e273873de802552cf528196b9/tests/debian_test.yaml)

メタデータ以外のテストが網羅されていますね。

まずMacOSでdockerドライバ(default)をつかってみます。`docker.sock`をマウントすればOKでしょうきっと。コンフィグファイルもマウントしましょう、ルート直下でOK。

このとき`-pull`をつけておくとイメージタグが必須になり(-pullでなければローカルからlatestを探索)、テストの前にpullしてくれます。

```bash:
$ docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v `pwd`/debian_test.yaml:/debian_test.yaml \
  gcr.io/gcp-runtimes/container-structure-test:v0.1.3 \
  -pull \
  -test.v \
  --image gcr.io/google-appengine/debian8:latest \
  debian_test.yaml

latest: Pulling from google-appengine/debian8
Digest: sha256:412ef4d53215ff4a95d275ad48fe5196cb51f4f96b99c05058054b3bdf9443c1
Status: Image is up to date for gcr.io/google-appengine/debian8:latest
Using driver docker
=== RUN   TestAll
=== RUN   TestAll/Command_Test:_apt-get
2018/01/11 06:40:59 Running tests for file debian_test.yaml
=== RUN   TestAll/Command_Test:_apt-config
=== RUN   TestAll/Command_Test:_path
=== RUN   TestAll/File_Existence_Test:_Root
=== RUN   TestAll/File_Existence_Test:_Netbase
=== RUN   TestAll/File_Existence_Test:_Machine_ID
=== RUN   TestAll/File_Content_Test:_Debian_Sources
=== RUN   TestAll/File_Content_Test:_Retry_Policy
=== RUN   TestAll/Metadata_Test
=== RUN   TestAll/License_Test_#0
--- PASS: TestAll (36.99s)
    --- PASS: TestAll/Command_Test:_apt-get (1.25s)
    	docker_driver.go:74: stdout: apt 1.0.9.8.4 for amd64 compiled on Dec 11 2016 09:48:19
    		Usage: apt-get [options] command
    		       apt-get [options] install|remove pkg1 [pkg2 ...]
    		       apt-get [options] source pkg1 [pkg2 ...]
    		
    		apt-get is a simple command line interface for downloading and
    		installing packages. The most frequently used commands are update
    		and install.
    		
    		Commands:
    		   update - Retrieve new lists of packages
    		   upgrade - Perform an upgrade
    		   install - Install new packages (pkg is libc6 not libc6.deb)
    		   remove - Remove packages
    		   autoremove - Remove automatically all unused packages
    		   purge - Remove packages and config files
    		   source - Download source archives
    		   build-dep - Configure build-dependencies for source packages
    		   dist-upgrade - Distribution upgrade, see apt-get(8)
    		   dselect-upgrade - Follow dselect selections
    		   clean - Erase downloaded archive files
    		   autoclean - Erase old downloaded archive files
    		   check - Verify that there are no broken dependencies
    		   changelog - Download and display the changelog for the given package
    		   download - Download the binary package into the current directory
    		
    		Options:
    		  -h  This help text.
    		  -q  Loggable output - no progress indicator
    		  -qq No output except for errors
    		  -d  Download only - do NOT install or unpack archives
    		  -s  No-act. Perform ordering simulation
    		  -y  Assume Yes to all queries and do not prompt
    		  -f  Attempt to correct a system with broken dependencies in place
    		  -m  Attempt to continue if archives are unlocatable
    		  -u  Show a list of upgraded packages as well
    		  -b  Build the source package after fetching it
    		  -V  Show verbose version numbers
    		  -c=? Read this configuration file
    		  -o=? Set an arbitrary configuration option, eg -o dir::cache=/tmp
    		See the apt-get(8), sources.list(5) and apt.conf(5) manual
    		pages for more information and options.
    		                       This APT has Super Cow Powers.
    --- PASS: TestAll/Command_Test:_apt-config (1.98s)
    	docker_driver.go:74: stdout: APT "";
    		APT::Architecture "amd64";
    		APT::Build-Essential "";
    		APT::Build-Essential:: "build-essential";
    		APT::Install-Recommends "1";
    		APT::Install-Suggests "0";
    		APT::NeverAutoRemove "";
    		APT::NeverAutoRemove:: "^firmware-linux.*";
    		APT::NeverAutoRemove:: "^linux-firmware$";
    		APT::VersionedKernelPackages "";
    		APT::VersionedKernelPackages:: "linux-image";
    		APT::VersionedKernelPackages:: "linux-headers";
    		APT::VersionedKernelPackages:: "linux-image-extra";
    		APT::VersionedKernelPackages:: "linux-signed-image";
    		APT::VersionedKernelPackages:: "kfreebsd-image";
    		APT::VersionedKernelPackages:: "kfreebsd-headers";
    		APT::VersionedKernelPackages:: "gnumach-image";
    		APT::VersionedKernelPackages:: ".*-modules";
    		APT::VersionedKernelPackages:: ".*-kernel";
    		APT::VersionedKernelPackages:: "linux-backports-modules-.*";
    		APT::VersionedKernelPackages:: "linux-tools";
    		APT::Never-MarkAuto-Sections "";
    		APT::Never-MarkAuto-Sections:: "metapackages";
    		APT::Never-MarkAuto-Sections:: "restricted/metapackages";
    		APT::Never-MarkAuto-Sections:: "universe/metapackages";
    		APT::Never-MarkAuto-Sections:: "multiverse/metapackages";
    		APT::Never-MarkAuto-Sections:: "oldlibs";
    		APT::Never-MarkAuto-Sections:: "restricted/oldlibs";
    		APT::Never-MarkAuto-Sections:: "universe/oldlibs";
    		APT::Never-MarkAuto-Sections:: "multiverse/oldlibs";
    		APT::AutoRemove "";
    		APT::AutoRemove::SuggestsImportant "false";
    		APT::Update "";
    		APT::Update::Post-Invoke "";
    		APT::Update::Post-Invoke:: "rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true";
    		APT::Architectures "";
    		APT::Architectures:: "amd64";
    		APT::Compressor "";
    		APT::Compressor::. "";
    		APT::Compressor::.::Name ".";
    		APT::Compressor::.::Extension "";
    		APT::Compressor::.::Binary "";
    		APT::Compressor::.::Cost "1";
    		APT::Compressor::gzip "";
    		APT::Compressor::gzip::Name "gzip";
    		APT::Compressor::gzip::Extension ".gz";
    		APT::Compressor::gzip::Binary "gzip";
    		APT::Compressor::gzip::Cost "2";
    		APT::Compressor::gzip::CompressArg "";
    		APT::Compressor::gzip::CompressArg:: "-9n";
    		APT::Compressor::gzip::UncompressArg "";
    		APT::Compressor::gzip::UncompressArg:: "-d";
    		APT::Compressor::bzip2 "";
    		APT::Compressor::bzip2::Name "bzip2";
    		APT::Compressor::bzip2::Extension ".bz2";
    		APT::Compressor::bzip2::Binary "false";
    		APT::Compressor::bzip2::Cost "3";
    		APT::Compressor::xz "";
    		APT::Compressor::xz::Name "xz";
    		APT::Compressor::xz::Extension ".xz";
    		APT::Compressor::xz::Binary "false";
    		APT::Compressor::xz::Cost "4";
    		APT::Compressor::lzma "";
    		APT::Compressor::lzma::Name "lzma";
    		APT::Compressor::lzma::Extension ".lzma";
    		APT::Compressor::lzma::Binary "false";
    		APT::Compressor::lzma::Cost "5";
    		APT::Compressor::lzma::CompressArg "";
    		APT::Compressor::lzma::CompressArg:: "--suffix=";
    		APT::Compressor::lzma::CompressArg:: "-9";
    		APT::Compressor::lzma::UncompressArg "";
    		APT::Compressor::lzma::UncompressArg:: "--suffix=";
    		APT::Compressor::lzma::UncompressArg:: "-d";
    		Dir "/";
    		Dir::State "var/lib/apt/";
    		Dir::State::lists "lists/";
    		Dir::State::cdroms "cdroms.list";
    		Dir::State::mirrors "mirrors/";
    		Dir::State::extended_states "extended_states";
    		Dir::State::status "/var/lib/dpkg/status";
    		Dir::Cache "var/cache/apt/";
    		Dir::Cache::archives "archives/";
    		Dir::Cache::srcpkgcache "";
    		Dir::Cache::pkgcache "";
    		Dir::Etc "etc/apt/";
    		Dir::Etc::sourcelist "sources.list";
    		Dir::Etc::sourceparts "sources.list.d";
    		Dir::Etc::vendorlist "vendors.list";
    		Dir::Etc::vendorparts "vendors.list.d";
    		Dir::Etc::main "apt.conf";
    		Dir::Etc::netrc "auth.conf";
    		Dir::Etc::parts "apt.conf.d";
    		Dir::Etc::preferences "preferences";
    		Dir::Etc::preferencesparts "preferences.d";
    		Dir::Etc::trusted "trusted.gpg";
    		Dir::Etc::trustedparts "trusted.gpg.d";
    		Dir::Bin "";
    		Dir::Bin::methods "/usr/lib/apt/methods";
    		Dir::Bin::solvers "";
    		Dir::Bin::solvers:: "/usr/lib/apt/solvers";
    		Dir::Bin::dpkg "/usr/bin/dpkg";
    		Dir::Bin::bzip2 "/bin/bzip2";
    		Dir::Bin::xz "/usr/bin/xz";
    		Dir::Bin::lzma "/usr/bin/lzma";
    		Dir::Media "";
    		Dir::Media::MountPath "/media/apt";
    		Dir::Log "var/log/apt";
    		Dir::Log::Terminal "term.log";
    		Dir::Log::History "history.log";
    		Dir::Ignore-Files-Silently "";
    		Dir::Ignore-Files-Silently:: "~$";
    		Dir::Ignore-Files-Silently:: "\.disabled$";
    		Dir::Ignore-Files-Silently:: "\.bak$";
    		Dir::Ignore-Files-Silently:: "\.dpkg-[a-z]+$";
    		Dir::Ignore-Files-Silently:: "\.save$";
    		Dir::Ignore-Files-Silently:: "\.orig$";
    		Dir::Ignore-Files-Silently:: "\.distUpgrade$";
    		Acquire "";
    		Acquire::cdrom "";
    		Acquire::cdrom::mount "/media/cdrom/";
    		Acquire::Retries "3";
    		Acquire::GzipIndexes "true";
    		Acquire::CompressionTypes "";
    		Acquire::CompressionTypes::Order "";
    		Acquire::CompressionTypes::Order:: "gz";
    		Acquire::Languages "";
    		Acquire::Languages:: "none";
    		DPkg "";
    		DPkg::Pre-Install-Pkgs "";
    		DPkg::Pre-Install-Pkgs:: "/usr/sbin/dpkg-preconfigure --apt || true";
    		DPkg::Post-Invoke "";
    		DPkg::Post-Invoke:: "rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true";
    		CommandLine "";
    		CommandLine::AsString "apt-config dump";
    --- PASS: TestAll/Command_Test:_path (1.54s)
    	docker_driver.go:74: stdout: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    --- PASS: TestAll/File_Existence_Test:_Root (0.67s)
    --- PASS: TestAll/File_Existence_Test:_Netbase (0.16s)
    --- PASS: TestAll/File_Existence_Test:_Machine_ID (0.16s)
    --- PASS: TestAll/File_Content_Test:_Debian_Sources (0.14s)
    --- PASS: TestAll/File_Content_Test:_Retry_Policy (0.18s)
    --- PASS: TestAll/Metadata_Test (0.00s)
    --- PASS: TestAll/License_Test_#0 (30.90s)
    	licenses.go:70: acl
    	licenses.go:70: adduser
    	licenses.go:70: apt
    	licenses.go:70: base-files
    	licenses.go:70: base-passwd
    	licenses.go:70: bash
    	licenses.go:70: bsdutils
    	licenses.go:70: ca-certificates
    	licenses.go:70: coreutils
    	licenses.go:70: dash
    	licenses.go:70: debconf
    	licenses.go:70: debian-archive-keyring
    	licenses.go:70: debianutils
    	licenses.go:70: diffutils
    	licenses.go:70: dmsetup
    	licenses.go:70: dpkg
    	licenses.go:70: e2fslibs
    	licenses.go:70: e2fsprogs
    	licenses.go:70: findutils
    	licenses.go:70: gcc-4.8-base
    	licenses.go:70: gcc-4.9-base
    	licenses.go:70: gnupg
    	licenses.go:70: gpgv
    	licenses.go:70: grep
    	licenses.go:70: gzip
    	licenses.go:70: hostname
    	licenses.go:70: init
    	licenses.go:70: initscripts
    	licenses.go:70: insserv
    	licenses.go:70: libacl1
    	licenses.go:70: libapt-pkg4.12
    	licenses.go:70: libattr1
    	licenses.go:70: libaudit-common
    	licenses.go:70: libaudit1
    	licenses.go:70: libblkid1
    	licenses.go:70: libbz2-1.0
    	licenses.go:70: libc-bin
    	licenses.go:70: libc6
    	licenses.go:70: libcap2
    	licenses.go:70: libcap2-bin
    	licenses.go:70: libcomerr2
    	licenses.go:70: libcryptsetup4
    	licenses.go:70: libdb5.3
    	licenses.go:70: libdebconfclient0
    	licenses.go:70: libdevmapper1.02.1
    	licenses.go:70: libgcrypt20
    	licenses.go:70: libgpg-error0
    	licenses.go:70: libkmod2
    	licenses.go:70: liblocale-gettext-perl
    	licenses.go:70: liblzma5
    	licenses.go:70: libmount1
    	licenses.go:70: libpam-modules
    	licenses.go:70: libpam-modules-bin
    	licenses.go:70: libpam-runtime
    	licenses.go:70: libpam0g
    	licenses.go:70: libpcre3
    	licenses.go:70: libprocps3
    	licenses.go:70: libreadline6
    	licenses.go:70: libselinux1
    	licenses.go:70: libsemanage-common
    	licenses.go:70: libsemanage1
    	licenses.go:70: libsepol1
    	licenses.go:70: libslang2
    	licenses.go:70: libsmartcols1
    	licenses.go:70: libss2
    	licenses.go:70: libssl1.0.0
    	licenses.go:70: libsystemd0
    	licenses.go:70: libtext-charwidth-perl
    	licenses.go:70: libtext-iconv-perl
    	licenses.go:70: libtext-wrapi18n-perl
    	licenses.go:70: libtinfo5
    	licenses.go:70: libudev1
    	licenses.go:70: libusb-0.1-4
    	licenses.go:70: libustr-1.0-1
    	licenses.go:70: libuuid1
    	licenses.go:70: login
    	licenses.go:70: lsb-base
    	licenses.go:70: mawk
    	licenses.go:70: mount
    	licenses.go:70: multiarch-support
    	licenses.go:70: ncurses-base
    	licenses.go:70: ncurses-bin
    	licenses.go:70: netbase
    	licenses.go:70: openssl
    	licenses.go:70: passwd
    	licenses.go:70: perl
    	licenses.go:70: procps
    	licenses.go:70: readline-common
    	licenses.go:70: sed
    	licenses.go:70: sensible-utils
    	licenses.go:70: startpar
    	licenses.go:70: systemd
    	licenses.go:70: systemd-sysv
    	licenses.go:70: sysv-rc
    	licenses.go:70: sysvinit-utils
    	licenses.go:70: tar
    	licenses.go:70: tzdata
    	licenses.go:70: util-linux
    	licenses.go:70: zlib1g
	structure_test.go:47: Total tests run: 10
PASS
```

最後に`PASS`でおわればOKです。`-test.v`をつけない場合は`PASS`だけでます。FAIL時はexit_codeもちゃんと0以外で終わるので安心。

### metadataTestもやってみよう

サンプルにはmetadataTestが入ってませんでした。imageをinspectして、定義をちょろっと書いてみます。

```yaml:metatest.yaml
schemaVersion: '2.0.0'
metadataTest:
  env:
    - key: "PORT"
      value: "8080"
  entrypoint: []
  cmd: ["/bin/sh", "-c", "/bin/bash"]
```

ファイル指定を変更し、実行します。

```bash:
$ docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v `pwd`/metatest.yaml:/metatest.yaml \
  gcr.io/gcp-runtimes/container-structure-test:v0.1.3 \
  -pull \
  -test.v \
  --image gcr.io/google-appengine/debian8:latest \
  metatest.yaml

latest: Pulling from google-appengine/debian8
Digest: sha256:412ef4d53215ff4a95d275ad48fe5196cb51f4f96b99c05058054b3bdf9443c1
Status: Image is up to date for gcr.io/google-appengine/debian8:latest
Using driver docker
=== RUN   TestAll
=== RUN   TestAll/Metadata_Test
2018/01/11 06:54:58 Running tests for file metatest.yaml
--- PASS: TestAll (0.01s)
    --- PASS: TestAll/Metadata_Test (0.00s)
	structure_test.go:47: Total tests run: 1
PASS
```

無事にテストできたようですね。


## driverにtarを指定、 dockerdなしでのチェック

さて、冒頭にもリンクをおいた[Googleの記事](https://opensource.googleblog.com/2018/01/container-structure-tests-unit-tests.html)では、dockerdがいなくてもテストできるぞ！と書いてあったので`-driver tar`を指定、dockerのソケットを外してみます。

```bash:
$ docker run --rm \
  -v `pwd`/debian_test.yaml:/debian_test.yaml \
  gcr.io/gcp-runtimes/container-structure-test:v0.1.3 \
  -driver tar \
  -test.v \
  --image gcr.io/google-appengine/debian8:latest \
  debian_test.yaml


Using driver tar
=== RUN   TestAll
2018/01/11 06:44:51 Running tests for file debian_test.yaml
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry

-- 省略
--- FAIL: TestAll (19.89s)
    --- FAIL: TestAll/Command_Test:_apt-get (10.03s)
    	tar_driver.go:66: Tar driver is unable to process commands, please use a different driver
    --- PASS: TestAll/Metadata_Test (9.86s)
	structure_test.go:47: Total tests run: 1
```

commandTestsは流石にダメっぽいです。commandTestsのセクションをコメントアウトして、もう一度実行。

```bash:
Using driver tar
=== RUN   TestAll
2018/01/11 07:18:55 Running tests for file debian_test.yaml
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry
=== RUN   TestAll/File_Existence_Test:_Root
time="2018-01-11T07:19:04Z" level=info msg="Finished prepping image gcr.io/google-appengine/debian8:latest"
time="2018-01-11T07:19:04Z" level=info msg="Removing image filesystem directory /tmp/gcr.iogoogle-appenginedebian8latest743647843 from system"
=== RUN   TestAll/File_Existence_Test:_Netbase
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry
time="2018-01-11T07:19:14Z" level=info msg="Finished prepping image gcr.io/google-appengine/debian8:latest"
time="2018-01-11T07:19:14Z" level=info msg="Removing image filesystem directory /tmp/gcr.iogoogle-appenginedebian8latest188699238 from system"
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry
=== RUN   TestAll/File_Existence_Test:_Machine_ID
time="2018-01-11T07:19:24Z" level=info msg="Finished prepping image gcr.io/google-appengine/debian8:latest"
time="2018-01-11T07:19:24Z" level=info msg="Removing image filesystem directory /tmp/gcr.iogoogle-appenginedebian8latest359712397 from system"
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry
=== RUN   TestAll/File_Content_Test:_Debian_Sources
time="2018-01-11T07:19:34Z" level=info msg="Finished prepping image gcr.io/google-appengine/debian8:latest"
time="2018-01-11T07:19:34Z" level=info msg="Removing image filesystem directory /tmp/gcr.iogoogle-appenginedebian8latest785687176 from system"
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry
=== RUN   TestAll/File_Content_Test:_Retry_Policy
time="2018-01-11T07:19:44Z" level=info msg="Finished prepping image gcr.io/google-appengine/debian8:latest"
time="2018-01-11T07:19:44Z" level=info msg="Removing image filesystem directory /tmp/gcr.iogoogle-appenginedebian8latest267774023 from system"
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry

... 省略

    	licenses.go:70: startpar
    	licenses.go:70: systemd
    	licenses.go:70: systemd-sysv
    	licenses.go:70: sysv-rc
    	licenses.go:70: sysvinit-utils
    	licenses.go:70: tar
    	licenses.go:70: tzdata
    	licenses.go:70: util-linux
    	licenses.go:70: zlib1g
	structure_test.go:47: Total tests run: 7
PASS
```

PASSしました。テストごとにファイルを展開削除しているようで、これはとても時間がかかる。

metadataTestはどうなるんだろう。

```bash:
$ docker run --rm \
  -v `pwd`/metatest.yaml:/metatest.yaml \
  gcr.io/gcp-runtimes/container-structure-test:v0.1.3 \
  -driver tar \
  -test.v \
  --image gcr.io/google-appengine/debian8:latest \
  metatest.yaml


Using driver tar
=== RUN   TestAll
=== RUN   TestAll/Metadata_Test
2018/01/11 06:56:09 Running tests for file metatest.yaml
Retrieving image gcr.io/google-appengine/debian8:latest from source Cloud Registry
time="2018-01-11T06:56:19Z" level=info msg="Finished prepping image gcr.io/google-appengine/debian8:latest"
time="2018-01-11T06:56:19Z" level=info msg="Removing image filesystem directory /tmp/gcr.iogoogle-appenginedebian8latest068032231 from system"
--- PASS: TestAll (10.13s)
    --- PASS: TestAll/Metadata_Test (10.13s)
	structure_test.go:47: Total tests run: 1
PASS
```

お、ちゃんとできるね。commandTestsは邪道感があるので、これでよいとするのが理想ではありますね。普通にCIを動かすLinux環境ならファイルの展開削除も早かろうし。


## おわりに

コンテナ特化で、いつものコマンド+簡潔な結果期待の形式で読みやすいYamlでかけてチェックできるので良いとおもいます。
