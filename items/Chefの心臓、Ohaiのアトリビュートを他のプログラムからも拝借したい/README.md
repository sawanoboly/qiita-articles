<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


アプリケーションプラットフォームの構築運用を少人数で行うためのツールとして、だいぶ浸透してきた感のある[OpscodeのChef](http://www.opscode.com/)。


chef-clientのクロスプラットフォーム性を支えるのが同じくOpscodeが保守するOhaiだ。

[Github::opscode/ohai](https://github.com/opscode/ohai)


Ohaiに聞くとプラットフォームの情報が取れる
---

Ohaiはインベントリ収集ツール。
レシピ(リソース/プロバイダ)を複数のプラットフォームに対応させるため、Chef-Clientが今から作用する対象を判断する情報をリストアップする。

Ohaiの利点は全てのプラットフォームで同じ問い合わせが出来ることだ、折角なので他のプログラムからも使えるように動作をチェックしてみる。


ohaiコマンド
---
gemで`ohai`をインストールした後なら、コマンドラインからohaiを叩ける。
これは`stdout`にJSONを出すだけだが、Chefが`node`のインベントリとして表示する情報がそのまま出ているのがわかる。


```bash::bash-cli
$ ohai | head
{
  "languages": {
    "ruby": {
      "platform": "x86_64-darwin12.2.0",
      "version": "1.9.3",
      "release_date": "2012-04-20",
      "target": "x86_64-apple-darwin12.2.0",
      "target_cpu": "x86_64",
      "target_vendor": "apple",
      "target_os": "darwin12.2.0",
```

結構これでも十分だ。

Ohai::Systemオブジェクト
----

OhaiをPryからインタラクティブに実行して、Ohaiプラグインを使ってインベントリを収集していく。
各アトリビュートは`Ohai::System`から作成したインスタンスの`@data`に溜まっていく。

```ruby::Pry-CLI
> require 'ohai'
=> true

> ohai = Ohai::System.new
=> #<Ohai::System:0x007fc7ec050440
 @data={},
 @hints={},
 @plugin_path="",
 @providers={},
 @seen_plugins={}>
```

プラグインパスが空に見えるが、デフォルトでOhai同梱のものへパスが通っており使用出来る。

```ruby::Pry-CLI
> Ohai::Config[:plugin_path]
=> ["/Users/sawanoboriyu/.rvm/gems/ruby-1.9.3-p194@chef-labs/gems/ohai-6.14.0/lib/ohai/plugins"]
```

### require_pluginで基本のOSプラグインを実行

プラグインの実行には`require_plugin`メソッドを使う、まずOS情報を取らないと他のプラグインは満足に動かないので初めに実行。

```ruby::Pry-CLI
> ohai.require_plugin "os"
=> true

> ohai.data
=> {"languages"=>
  {"ruby"=>
    {"platform"=>"x86_64-darwin12.2.0",
     "version"=>"1.9.3",
     "release_date"=>"2012-04-20",
     "target"=>"x86_64-apple-darwin12.2.0",
     "target_cpu"=>"x86_64",
     "target_vendor"=>"apple",
     "target_os"=>"darwin12.2.0",
     "host"=>"x86_64-apple-darwin12.2.0",
     "host_cpu"=>"x86_64",
     "host_os"=>"darwin12.2.0",
     "host_vendor"=>"apple",
     "bin_dir"=>"/Users/sawanoboriyu/.rvm/rubies/ruby-1.9.3-p194/bin",
     "ruby_bin"=>"/Users/sawanoboriyu/.rvm/rubies/ruby-1.9.3-p194/bin/ruby",
     "gems_dir"=>"/Users/sawanoboriyu/.rvm/gems/ruby-1.9.3-p194@chef-labs",
     "gem_bin"=>"/Users/sawanoboriyu/.rvm/rubies/ruby-1.9.3-p194/bin/gem"}},
 "kernel"=>
  {"name"=>"Darwin",
   "release"=>"12.2.0",
   "version"=>
    "Darwin Kernel Version 12.2.0: Sat Aug 25 00:48:52 PDT 2012; root:xnu-2050.18.24~1/RELEASE_X86_64",
   "machine"=>"x86_64",
   "modules"=>{}},
 "os"=>"darwin",
 "os_version"=>"12.2.0"}
```

裏で色々と泥臭い処理が走った結果、見事にOSに関するインベントリが収集されている。
相手の**OSが何でも同じ構造のハッシュに格納**してくれるのは色々と助かるはず。


### プラグインの追加実行でアトリビュート追加

"OS"のインベントリ採取が終わった後なら他のプラグインも適当に実行できる。
`platform`や`ohai(plugin)`を実行して、関連するアトリビュートを`@data`に追加しておく。

```ruby::Pry-CLI
> ohai.require_plugin "platform"
=> true


> ohai.require_plugin "platform"
=> true
> ohai.data
=> {"languages"=>
  {"ruby"=>

-- snip --

 "os"=>"darwin",
 "os_version"=>"12.2.0",
 "platform"=>"mac_os_x",
 "platform_version"=>"10.8.2",
 "platform_build"=>"12C60",
 "platform_family"=>"mac_os_x"}



> ohai.require_plugin "ohai"

> ohai.data[:chef_packages]
=> {"ohai"=>
  {"version"=>"6.14.0",
   "ohai_root"=>
    "/Users/sawanoboriyu/.rvm/gems/ruby-1.9.3-p194@chef-labs/gems/ohai-6.14.0/lib/ohai"}}
```


### パスにあるすべてのプラグインを実行
`all_plugins`はすべてのプラグインを実行する。


```ruby::Pry-CLI
> ohai.all_plugins

> ohai.seen_plugins
=> {"os"=>true,
 "kernel"=>true,
 "ruby"=>true,
 "languages"=>true,
 "c"=>true,
 "chef"=>true,
 "cloud"=>true,
 "ec2"=>true,
 "hostname"=>true,
 "darwin::hostname"=>true,
 "network"=>true,
 "darwin::network"=>true,
 "rackspace"=>true,
 "eucalyptus"=>true,
 "command"=>true,
 "dmi"=>true,
 "dmi_common"=>true,
 "erlang"=>true,
 "groovy"=>true,
 "ip_scopes"=>true,
 "java"=>true,
 "keys"=>true,
 "lua"=>true,
 "mono"=>true,
 "network_listeners"=>true,
 "ohai"=>true,
 "ohai_time"=>true,
 "passwd"=>true,
 "perl"=>true,
 "php"=>true,
 "platform"=>true,
 "darwin::platform"=>true,
 "python"=>true,
 "uptime"=>true,
 "virtualization"=>true,
 "darwin::virtualization"=>true,
 "darwin::filesystem"=>true,
 "darwin::kernel"=>true,
 "darwin::ps"=>true,
 "darwin::ssh_host_key"=>true,
 "darwin::system_profiler"=>true,
 "darwin::uptime"=>true}
```


Ohaiプラグインを作成する
---

インベントリを取得したいノードによってはなにかしらアトリビュートを追加したい時もある。
他のプラグインにならえば、Ohaiプラグインはプロバイダと取得するコマンドラインを指定する形で追加になる。


### プロバイダを追加

Pryのまま行う、実際は`プラグイン名.rb`ファイルに`ohai.`を除いた形で書いておく。

```ruby::Pry-CLI
> ohai.provides "brew_path", "brew_ver"
=> ["brew_path", "brew_ver"]

> ohai
=> #<Ohai::System:0x007f8ecbbef940
 @data={},
 @hints={},
 @plugin_path="",
 @providers=
  {"brew_path"=>{"_providers"=>[""]}, "brew_ver"=>{"_providers"=>[""]}},
 @seen_plugins={}>
```

### プロバイダの取得方法を定義

簡単な方法の一つ、`Ohai::Ststem`のインスタンスメソッド`from`をつかう。
マルチプラットフォーム対応で実行したコマンドの標準出力を格納できる。

```ruby::Pry-CLI
> ohai.brew_path ohai.from("which brew")
=> "/usr/local/bin/brew"

> ohai.brew_ver ohai.from("#{ohai.brew_path} -v")
=> "Homebrew 0.9.3"

> ohai.data
=> {"brew_path"=>"/usr/local/bin/brew", "brew_ver"=>"Homebrew 0.9.3"}
```


おわりに
---

Chefは全体的にまずガサ入れをしてからなんぼというツールだが、特にこのOhaiの泥臭さはなかなかだ。

