<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

ChefのOpscodeが公開している[omnibus-ruby](https://github.com/opscode/omnibus-ruby)というのがあります。`omnibus-chef`(/opt/chef`にRubyごとChefをインストール)のなんでも版ですね。

libやincludeもまるごと、lddで拾える依存も全部パッケージにするので、実際Ruby関連に限らず使えると思います。

> ビルド環境については刷新したのでこちらをどうぞ。
> [Omnibus[-ruby]プロジェクトのビルド環境をDockerで。Dockerhub公開中。](http://qiita.com/sawanoboly/items/0902a97ef71d99ad60a6)


## omnibusの準備

まずツール群を揃えます。

```ruby:Gemfile
source "https://rubygems.org"

gem 'omnibus'
```

一旦これで`bundle`します。

Vagrantのプラグインを入れたらVM内でパッケージ作成が出来て便利、今回は`Vagrant-1.3.3`を使いました。

```shell:vagrant_plugins
vagrant plugin install vagrant-berkshelf
vagrant plugin install vagrant-omnibus
```

## サンプル、omnibus-godを作る

なんか作ってみましょう、Godがちょうどいいかな。

- [God - A Process Monitoring Framework in Ruby](http://godrb.com)


## プロジェクトを作成

omnibusは、`Project`というまとまりに、任意の`Software`をビルドして一つのパッケージにするという考え方で組まれています。

まずプロジェクト。

```shell:create_project
$ omnibus project god
No configuration file `/Users/sawanoboriyu/worktemp/omnibus/omnibus.rb', using defaults
      create  omnibus-god/Gemfile
      create  omnibus-god/.gitignore
      create  omnibus-god/README.md
      create  omnibus-god/omnibus.rb.example
      create  omnibus-god/config/projects/god.rb
      create  omnibus-god/config/software/c-example.rb
      create  omnibus-god/config/software/erlang-example.rb
      create  omnibus-god/config/software/ruby-example.rb
      create  omnibus-god/Berksfile
      create  omnibus-god/Vagrantfile
      create  omnibus-god/package-scripts/god/makeselfinst
      create  omnibus-god/package-scripts/god/postinst
      create  omnibus-god/package-scripts/god/postrm
```

そしてもう一度bundleです。

```shell:bundle_for_project
$ cd omnibus-god
$ bundle install --binstubs
```

これで準備はOK。


## Vagrantのチェック

自動生成のVagrantfileに1.2のオプションが残っているので、1.3を使う場合は2つコメントアウトします。

```ruby:Vagrantfile
#  config.ssh.max_tries = 40
#  config.ssh.timeout   = 120
  config.ssh.forward_agent = true
```

VMのリストを見てみます。


```shell:vagrant_status-1
$ vagrant status
Current machine states:

ubuntu-10.04              not created (virtualbox)
ubuntu-11.04              not created (virtualbox)
ubuntu-12.04              not created (virtualbox)
centos-5                  not created (virtualbox)
centos-6                  not created (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

## Softwareの定義をする

godを入れるので、`config/projects/software/god.rb`を作成します。

取り急ぎgithubから取ってきます。


```ruby:config/projects/software/god.rb
name "god"
version "v0.13.2"          # git ref
gem_version = '0.13.2'  # local_variable

dependency "ruby"
dependency "rubygems"
dependency "bundler"

source :git => "https://github.com/mojombo/god.git"
relative_path "god"

env = {
  "PATH" => "#{install_dir}/embedded/bin:#{ENV["PATH"]}"
}

build do
  command "rake build", :env => env
  command "sudo #{install_dir}/embedded/bin/gem install --no-ri --no-rdoc -l pkg/god-#{gem_version}.gem", :env => env
  command "sudo ln -fs #{install_dir}/embedded/bin/god #{install_dir}/bin/god", :env => env
end
```

name, buildが必須なくらいで、ほかの注意点はこのくらいでしょうか。

- `version `がgitのタグ/ブランチになる
- `relative_path` はgit cloneした後にcdする先
- このdependencyの３つはプリセット、独自に使う場合は別途`software/`以下に定義も可能
- `command`2行目のgemコマンドは`dependency`にgemがあるから事前に準備される

ほかにも`c-example`, `erlang-example`が付いているので参考にすると良いでしょう。


## Projectファイルを修正

初期ファイルから、`dependency "god"`の一文を加えます。さきほどsoftwareで定義したものがリストできます。

```ruby:config/projects/god.rb
name "god"
maintainer "CHANGE ME"
homepage "CHANGEME.com"

replaces        "god"
install_path    "/opt/god"
build_version   Omnibus::BuildVersion.new.semver
build_iteration 1

# creates required build directories
dependency "preparation"

# god dependencies/components
dependency "god"    ## add this line

# version manifest file
dependency "version-manifest"

exclude "\.git*"
exclude "bundler\/git"
```

## Vagrant内でパッケージ作成

upしてひたすら待ちましょう。`omnibus::default`がChefで流れていきます。

```shell:vagrant_up_ubuntu
$ vagrant up ubuntu-12.04
-- snip --
[2013-09-23T09:05:11+00:00] INFO: Forking chef instance to converge...
[2013-09-23T09:05:11+00:00] INFO: *** Chef 11.6.0 ***
[2013-09-23T09:05:11+00:00] INFO: Setting the run_list to ["recipe[omnibus::default]"] from JSON
[2013-09-23T09:05:11+00:00] INFO: Run List is [recipe[omnibus::default]]
[2013-09-23T09:05:11+00:00] INFO: Run List expands to [omnibus::default]
[2013-09-23T09:05:11+00:00] INFO: Starting Chef Run for god-omnibus-build-lab
[2013-09-23T09:05:11+00:00] INFO: Running start handlers
-- snip --

[builder:god] building god
[builder:god] Executing: `rake build` with cwd=/var/cache/omnibus/src/god,timeout=5400,env="PATH=/opt/god/embedded/bin:/opt/ruby1.9/lib/ruby/gems/1.9.1/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games"
[builder:god] rake command succeeded, 0.537102291s

[builder:god] Executing: `sudo /opt/god/embedded/bin/gem install --no-ri --no-rdoc -l pkg/god-0.13.2.gem` with cwd=/var/cache/omnibus/src/god,timeout=5400,env="PATH=/opt/god/embedded/bin:/opt/ruby1.9/lib/ruby/gems/1.9.1/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games"
[builder:god] sudo command succeeded, 0.557560365s

[builder:god] Executing: `sudo ln -fs /opt/god/embedded/bin/god /opt/god/bin/god` with cwd=/var/cache/omnibus/src/god,timeout=5400,env="PATH=/opt/god/embedded/bin:/opt/ruby1.9/lib/ruby/gems/1.9.1/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games"
[builder:god] sudo command succeeded, 0.045231982s

[builder:god] god build succeeded, 1.141283219s
-- snip --

[health_check] Executing `find /opt/god/ -type f | xargs ldd > ldd.out 2>/dev/null`
{:timestamp=>"2013-09-23T09:19:38.647119+0000", :message=>"Created deb package", :path=>"god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb"}
```

### debパッケージを確認

vagrantのVM内で作られたdebパケージは手元に来ます。

```shell:ls_pkg-1
$ ls -1 pkg/
god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb
god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb.metadata.json
```

### rpmもつくれる

```
$ vagrant up centos-6 
-- snip --

[health_check] Executing `find /opt/god/ -type f | xargs ldd > ldd.out 2>/dev/null`
{:timestamp=>"2013-09-23T09:50:53.418506+0000", :message=>"no value for epoch is set, defaulting to nil", :level=>:warn}
{:timestamp=>"2013-09-23T09:50:53.424771+0000", :message=>"no value for epoch is set, defaulting to nil", :level=>:warn}
{:timestamp=>"2013-09-23T09:51:27.379855+0000", :message=>"Created rpm", :path=>"god-0.0.0+20130923094240-1.el6.x86_64.rpm"}
```

```shell:ls_pkg-2
$ ls -1 pkg/
god-0.0.0+20130923094240-1.el6.x86_64.rpm
god-0.0.0+20130923094240-1.el6.x86_64.rpm.metadata.json
god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb
god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb.metadata.json
```


## deb,rpmパッケージの中身

出来上がったdebの中身を見てみましょう。

```shell:dpkg
$ dpkg --info /var/cache/omnibus/pkg/god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb  
 new debian package, version 2.0.
 size 19872738 bytes: control archive= 142079 bytes.
     282 bytes,    12 lines      control              
  527865 bytes,  5746 lines      md5sums              
     236 bytes,    17 lines   *  postinst             #!/bin/bash
     128 bytes,     9 lines   *  postrm               #!/bin/bash
 Package: god
 Version: 0.0.0+20130923091221-1.ubuntu.12.04
 License: unknown
 Vendor: vagrant@god-omnibus-build-lab
 Architecture: amd64
 Maintainer: CHANGE ME
 Installed-Size: 70592
 Replaces: god
 Section: default
 Priority: extra
 Homepage: CHANGEME.com
 Description: The full stack of god

$ dpkg --contents /var/cache/omnibus/pkg/god_0.0.0+20130923091221-1.ubuntu.12.04_amd64.deb | head
drwx------ vagrant/vagrant   0 2013-09-23 09:19 ./
drwxrwxr-x vagrant/vagrant   0 2013-09-23 09:19 ./opt/
drwxr-xr-x vagrant/vagrant   0 2013-09-23 09:19 ./opt/god/
drwxrwxr-x vagrant/vagrant   0 2013-09-23 09:19 ./opt/god/bin/
lrwxrwxrwx vagrant/vagrant   0 2013-09-23 09:19 ./opt/god/bin/god -> /opt/god/embedded/bin/god
-rw-rw-r-- vagrant/vagrant 1278 2013-09-23 09:19 ./opt/god/version-manifest.txt
drwxrwxr-x vagrant/vagrant    0 2013-09-23 09:19 ./opt/god/embedded/
drwxrwxr-x vagrant/vagrant    0 2013-09-23 09:19 ./opt/god/embedded/ssl/
lrwxrwxrwx vagrant/vagrant    0 2013-09-23 09:13 ./opt/god/embedded/ssl/cert.pem -> /opt/god/embedded/ssl/certs/cacert.pem
-rw-r--r-- vagrant/vagrant 10835 2013-09-23 09:15 ./opt/god/embedded/ssl/openssl.cnf
```

ざっと8,000ファイル！ほどありました、展開するとこのように全部入りであることがわかります。

```shell:tree_of_omnibus_god
$ tree opt/ -d
opt/
└── god
    ├── bin
    └── embedded
        ├── bin
        ├── include
        │   ├── editline
        │   ├── openssl
        │   └── ruby-1.9.1
        │       ├── ruby
        │       │   └── backward
        │       └── x86_64-linux
        │           └── ruby
        ├── lib
        │   ├── engines
        │   ├── pkgconfig
        │   ├── ruby
        │   │   ├── 1.9.1
        │   │   │   ├── bigdecimal
        │   │   │   ├── cgi
        │   │   │   │   └── session
        │   │   │   ├── date
        │   │   │   ├── digest
        │   │   │   ├── dl
        │   │   │   ├── drb
        │   │   │   ├── io
        │   │   │   │   └── console
        │   │   │   ├── irb
        │   │   │   │   ├── cmd
        │   │   │   │   ├── ext
        │   │   │   │   └── lc
        │   │   │   │       └── ja
        │   │   │   ├── json
        │   │   │   │   └── add
        │   │   │   ├── matrix
        │   │   │   ├── minitest
        │   │   │   ├── net
        │   │   │   ├── openssl
        │   │   │   ├── optparse
        │   │   │   ├── psych
        │   │   │   │   ├── handlers
        │   │   │   │   ├── json
        │   │   │   │   ├── nodes
        │   │   │   │   └── visitors
        │   │   │   ├── racc
        │   │   │   ├── rake
        │   │   │   │   ├── contrib
        │   │   │   │   ├── ext
        │   │   │   │   ├── lib
        │   │   │   │   └── loaders
        │   │   │   ├── rbconfig
        │   │   │   ├── rdoc
        │   │   │   │   ├── generator
        │   │   │   │   │   └── template
        │   │   │   │   │       └── darkfish
        │   │   │   │   │           ├── images
        │   │   │   │   │           └── js
        │   │   │   │   ├── markup
        │   │   │   │   ├── parser
        │   │   │   │   ├── ri
        │   │   │   │   └── stats
        │   │   │   ├── rexml
        │   │   │   │   ├── dtd
        │   │   │   │   ├── formatters
        │   │   │   │   ├── light
        │   │   │   │   ├── parsers
        │   │   │   │   └── validation
        │   │   │   ├── rinda
        │   │   │   ├── ripper
        │   │   │   ├── rss
        │   │   │   │   ├── content
        │   │   │   │   ├── dublincore
        │   │   │   │   └── maker
        │   │   │   ├── rubygems
        │   │   │   │   ├── commands
        │   │   │   │   ├── ext
        │   │   │   │   ├── package
        │   │   │   │   │   └── tar_reader
        │   │   │   │   └── ssl_certs
        │   │   │   ├── shell
        │   │   │   ├── syck
        │   │   │   ├── test
        │   │   │   │   └── unit
        │   │   │   ├── uri
        │   │   │   ├── webrick
        │   │   │   │   ├── httpauth
        │   │   │   │   └── httpservlet
        │   │   │   ├── x86_64-linux
        │   │   │   │   ├── digest
        │   │   │   │   ├── dl
        │   │   │   │   ├── enc
        │   │   │   │   │   └── trans
        │   │   │   │   ├── io
        │   │   │   │   ├── json
        │   │   │   │   │   └── ext
        │   │   │   │   ├── mathn
        │   │   │   │   └── racc
        │   │   │   ├── xmlrpc
        │   │   │   └── yaml
        │   │   ├── gems
        │   │   │   └── 1.9.1
        │   │   │       ├── cache
        │   │   │       ├── doc
        │   │   │       │   └── rubygems-1.8.24
        │   │   │       │       ├── rdoc
        │   │   │       │       │   ├── Gem
        │   │   │       │       │   │   ├── Commands
        │   │   │       │       │   │   ├── Ext
        │   │   │       │       │   │   ├── Installer
        │   │   │       │       │   │   ├── MockGemUi
        │   │   │       │       │   │   ├── Package
        │   │   │       │       │   │   ├── RemoteFetcher
        │   │   │       │       │   │   ├── Security
        │   │   │       │       │   │   └── StreamUI
        │   │   │       │       │   ├── images
        │   │   │       │       │   ├── js
        │   │   │       │       │   ├── lib
        │   │   │       │       │   │   ├── rbconfig
        │   │   │       │       │   │   └── rubygems
        │   │   │       │       │   │       ├── commands
        │   │   │       │       │   │       ├── ext
        │   │   │       │       │   │       └── package
        │   │   │       │       │   ├── OpenSSL
        │   │   │       │       │   │   └── X509
        │   │   │       │       │   └── Psych
        │   │   │       │       └── ri
        │   │   │       │           ├── Date
        │   │   │       │           ├── DefaultKey
        │   │   │       │           ├── Gem
        │   │   │       │           │   ├── Builder
        │   │   │       │           │   ├── Command
        │   │   │       │           │   ├── CommandLineError
        │   │   │       │           │   ├── CommandManager
        │   │   │       │           │   ├── Commands
        │   │   │       │           │   │   ├── BuildCommand
        │   │   │       │           │   │   ├── CertCommand
        │   │   │       │           │   │   ├── CheckCommand
        │   │   │       │           │   │   ├── CleanupCommand
        │   │   │       │           │   │   ├── ContentsCommand
        │   │   │       │           │   │   ├── DependencyCommand
        │   │   │       │           │   │   ├── EnvironmentCommand
        │   │   │       │           │   │   ├── FetchCommand
        │   │   │       │           │   │   ├── GenerateIndexCommand
        │   │   │       │           │   │   ├── HelpCommand
        │   │   │       │           │   │   ├── InstallCommand
        │   │   │       │           │   │   ├── ListCommand
        │   │   │       │           │   │   ├── LockCommand
        │   │   │       │           │   │   ├── OutdatedCommand
        │   │   │       │           │   │   ├── OwnerCommand
        │   │   │       │           │   │   ├── PristineCommand
        │   │   │       │           │   │   ├── PushCommand
        │   │   │       │           │   │   ├── QueryCommand
        │   │   │       │           │   │   ├── RdocCommand
        │   │   │       │           │   │   ├── SearchCommand
        │   │   │       │           │   │   ├── ServerCommand
        │   │   │       │           │   │   ├── SetupCommand
        │   │   │       │           │   │   ├── SourcesCommand
        │   │   │       │           │   │   ├── SpecificationCommand
        │   │   │       │           │   │   ├── StaleCommand
        │   │   │       │           │   │   ├── UninstallCommand
        │   │   │       │           │   │   ├── UnpackCommand
        │   │   │       │           │   │   ├── UpdateCommand
        │   │   │       │           │   │   └── WhichCommand
        │   │   │       │           │   ├── ConfigFile
        │   │   │       │           │   ├── ConsoleUI
        │   │   │       │           │   ├── DefaultUserInteraction
        │   │   │       │           │   ├── Dependency
        │   │   │       │           │   ├── DependencyError
        │   │   │       │           │   ├── DependencyInstaller
        │   │   │       │           │   ├── DependencyList
        │   │   │       │           │   ├── DependencyRemovalException
        │   │   │       │           │   ├── Deprecate
        │   │   │       │           │   ├── DocManager
        │   │   │       │           │   ├── DocumentError
        │   │   │       │           │   ├── EndOfYAMLException
        │   │   │       │           │   ├── ErrorReason
        │   │   │       │           │   ├── Exception
        │   │   │       │           │   ├── Ext
        │   │   │       │           │   │   ├── Builder
        │   │   │       │           │   │   ├── ConfigureBuilder
        │   │   │       │           │   │   ├── ExtConfBuilder
        │   │   │       │           │   │   └── RakeBuilder
        │   │   │       │           │   ├── FakeFetcher
        │   │   │       │           │   ├── FilePermissionError
        │   │   │       │           │   ├── Format
        │   │   │       │           │   ├── FormatException
        │   │   │       │           │   ├── GemcutterUtilities
        │   │   │       │           │   ├── GemNotFoundException
        │   │   │       │           │   ├── GemNotInHomeException
        │   │   │       │           │   ├── GemPathSearcher
        │   │   │       │           │   ├── GemRunner
        │   │   │       │           │   ├── Indexer
        │   │   │       │           │   ├── Installer
        │   │   │       │           │   │   └── ExtensionBuildError
        │   │   │       │           │   ├── InstallError
        │   │   │       │           │   ├── InstallerTestCase
        │   │   │       │           │   ├── InstallUpdateOptions
        │   │   │       │           │   ├── InvalidSpecificationException
        │   │   │       │           │   ├── LoadError
        │   │   │       │           │   ├── LocalRemoteOptions
        │   │   │       │           │   ├── MockGemUi
        │   │   │       │           │   │   ├── SystemExitException
        │   │   │       │           │   │   ├── TermError
        │   │   │       │           │   │   └── TTY
        │   │   │       │           │   ├── NoAliasYAMLTree
        │   │   │       │           │   ├── OldFormat
        │   │   │       │           │   ├── OperationNotSupportedError
        │   │   │       │           │   ├── Package
        │   │   │       │           │   │   └── TarTestCase
        │   │   │       │           │   ├── PackageTask
        │   │   │       │           │   ├── PathSupport
        │   │   │       │           │   ├── Platform
        │   │   │       │           │   ├── PlatformMismatch
        │   │   │       │           │   ├── RemoteError
        │   │   │       │           │   ├── RemoteFetcher
        │   │   │       │           │   │   └── FetchError
        │   │   │       │           │   ├── RemoteInstallationCancelled
        │   │   │       │           │   ├── RemoteInstallationSkipped
        │   │   │       │           │   ├── RemoteSourceException
        │   │   │       │           │   ├── Requirement
        │   │   │       │           │   ├── RequirePathsBuilder
        │   │   │       │           │   ├── Security
        │   │   │       │           │   │   ├── Exception
        │   │   │       │           │   │   ├── Policy
        │   │   │       │           │   │   └── Signer
        │   │   │       │           │   ├── Server
        │   │   │       │           │   ├── SilentUI
        │   │   │       │           │   ├── SourceIndex
        │   │   │       │           │   ├── SpecFetcher
        │   │   │       │           │   ├── Specification
        │   │   │       │           │   ├── StreamUI
        │   │   │       │           │   │   ├── SilentDownloadReporter
        │   │   │       │           │   │   ├── SilentProgressReporter
        │   │   │       │           │   │   ├── SimpleProgressReporter
        │   │   │       │           │   │   ├── VerboseDownloadReporter
        │   │   │       │           │   │   └── VerboseProgressReporter
        │   │   │       │           │   ├── SystemExitException
        │   │   │       │           │   ├── TestCase
        │   │   │       │           │   ├── Text
        │   │   │       │           │   ├── Uninstaller
        │   │   │       │           │   ├── UserInteraction
        │   │   │       │           │   ├── Validator
        │   │   │       │           │   ├── VerificationError
        │   │   │       │           │   ├── Version
        │   │   │       │           │   └── VersionOption
        │   │   │       │           ├── GemGauntlet
        │   │   │       │           ├── Kernel
        │   │   │       │           ├── Object
        │   │   │       │           ├── OpenSSL
        │   │   │       │           │   └── X509
        │   │   │       │           │       └── Certificate
        │   │   │       │           ├── Psych
        │   │   │       │           │   └── PrivateType
        │   │   │       │           ├── RbConfig
        │   │   │       │           └── TempIO
        │   │   │       ├── gems
        │   │   │       │   ├── bundler-1.1.5
        │   │   │       │   │   ├── bin
        │   │   │       │   │   ├── lib
        │   │   │       │   │   │   └── bundler
        │   │   │       │   │   │       ├── man
        │   │   │       │   │   │       ├── templates
        │   │   │       │   │   │       │   └── newgem
        │   │   │       │   │   │       │       ├── bin
        │   │   │       │   │   │       │       └── lib
        │   │   │       │   │   │       │           └── newgem
        │   │   │       │   │   │       └── vendor
        │   │   │       │   │   │           ├── net
        │   │   │       │   │   │           │   └── http
        │   │   │       │   │   │           └── thor
        │   │   │       │   │   │               ├── actions
        │   │   │       │   │   │               ├── core_ext
        │   │   │       │   │   │               ├── parser
        │   │   │       │   │   │               └── shell
        │   │   │       │   │   ├── man
        │   │   │       │   │   └── spec
        │   │   │       │   │       ├── bundler
        │   │   │       │   │       ├── cache
        │   │   │       │   │       ├── install
        │   │   │       │   │       │   └── gems
        │   │   │       │   │       ├── lock
        │   │   │       │   │       ├── other
        │   │   │       │   │       ├── realworld
        │   │   │       │   │       ├── resolver
        │   │   │       │   │       ├── runtime
        │   │   │       │   │       ├── support
        │   │   │       │   │       │   ├── artifice
        │   │   │       │   │       │   ├── fakeweb
        │   │   │       │   │       │   └── rubygems_hax
        │   │   │       │   │       └── update
        │   │   │       │   ├── god-0.13.2
        │   │   │       │   │   ├── bin
        │   │   │       │   │   ├── doc
        │   │   │       │   │   ├── ext
        │   │   │       │   │   │   └── god
        │   │   │       │   │   ├── lib
        │   │   │       │   │   │   └── god
        │   │   │       │   │   │       ├── behaviors
        │   │   │       │   │   │       ├── cli
        │   │   │       │   │   │       ├── conditions
        │   │   │       │   │   │       ├── contacts
        │   │   │       │   │   │       ├── event_handlers
        │   │   │       │   │   │       └── system
        │   │   │       │   │   └── test
        │   │   │       │   │       └── configs
        │   │   │       │   │           ├── child_events
        │   │   │       │   │           ├── child_polls
        │   │   │       │   │           ├── complex
        │   │   │       │   │           ├── contact
        │   │   │       │   │           ├── daemon_events
        │   │   │       │   │           ├── daemon_polls
        │   │   │       │   │           ├── degrading_lambda
        │   │   │       │   │           ├── keepalive
        │   │   │       │   │           ├── lifecycle
        │   │   │       │   │           ├── matias
        │   │   │       │   │           ├── running_load
        │   │   │       │   │           ├── stop_options
        │   │   │       │   │           ├── stress
        │   │   │       │   │           └── task
        │   │   │       │   │               └── logs
        │   │   │       │   ├── rake-0.9.2.2
        │   │   │       │   │   └── bin
        │   │   │       │   └── rdoc-3.9.5
        │   │   │       │       └── bin
        │   │   │       └── specifications
        │   │   ├── site_ruby
        │   │   │   └── 1.9.1
        │   │   │       ├── rbconfig
        │   │   │       ├── rubygems
        │   │   │       │   ├── commands
        │   │   │       │   ├── ext
        │   │   │       │   ├── package
        │   │   │       │   │   └── tar_reader
        │   │   │       │   └── ssl_certs
        │   │   │       └── x86_64-linux
        │   │   └── vendor_ruby
        │   │       └── 1.9.1
        │   │           └── x86_64-linux
        │   └── terminfo -> ../share/terminfo
        ├── man
        │   ├── man1
        │   ├── man3
        │   ├── man5
        │   └── man7
        ├── share
        │   ├── man
        │   │   ├── man1
        │   │   ├── man3
        │   │   └── man5
        │   ├── tabset
        │   └── terminfo
        │       ├── 1
        │       ├── 2
        │       ├── 3
        │       ├── 4
        │       ├── 5
        │       ├── 6
        │       ├── 7
        │       ├── 8
        │       ├── 9
        │       ├── a
        │       ├── A
        │       ├── b
        │       ├── c
        │       ├── d
        │       ├── e
        │       ├── E
        │       ├── f
        │       ├── g
        │       ├── h
        │       ├── i
        │       ├── j
        │       ├── k
        │       ├── l
        │       ├── L
        │       ├── m
        │       ├── M
        │       ├── n
        │       ├── N
        │       ├── o
        │       ├── p
        │       ├── P
        │       ├── q
        │       ├── Q
        │       ├── r
        │       ├── s
        │       ├── t
        │       ├── u
        │       ├── v
        │       ├── w
        │       ├── x
        │       ├── X
        │       └── z
        └── ssl
            ├── certs
            ├── man
            │   ├── man1
            │   ├── man3
            │   ├── man5
            │   └── man7
            ├── misc
            └── private

401 directories
```

## godのインストール

もちろんこのパッケージをインストールすればgodが利用できます。

```shell:god_version
# /opt/god/bin/god -V
Version: 0.13.2
Polls: enabled
Events: netlink
```

## 終わりに

サービスのコアコンポーネントを独自で作ったりしているけど配布にお悩みの場合、このような富豪的デプロイもありですね。
