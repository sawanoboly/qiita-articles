<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


Ruby製のデプロイツールCapistranoは何かと便利でよくつかう。Railsにかぎらずコードベースから何かを持ってくる用途に向いている。

capコマンドから実行するのもいいが、適当なSinatraアプリに組み込んだりできると良いかもしれないと思い、感触をつかむためirbなんかでインタラクティブに叩いてみた。

単純にCapistranoのWEB-UIが必要ならrailsで作られた[Webistrano](https://github.com/peritor/webistrano)というのがある。元は古いけど幾つかのフォークされたものは新しめの環境でも動作するようだ。

Capistranoソースのチェックには[メタプログラミングRuby](http://www.amazon.co.jp/dp/4048687158)をかなり参考にした。

## まずは Capistrano::Configuration.new

今回はRubyの実行に[Pry](http://pryrepl.org/)を使う。

どうやらCapistranoのすべては`Capistrano::Configuration`クラスのオブジェクトに集まる模様、インスタンスを作成する。

```ruby:Pry-Console
> require 'capistrano'
=> true

> config = Capistrano::Configuration.new
=> #<Capistrano::Configuration:0x00000001f26678
 @callbacks={},
 @debug=false,
 @dry_run=false,
 @load_paths=
  [".",
   "/usr/local/rvm/gems/ruby-1.9.3-p286@cap/gems/capistrano-2.13.5/lib/capistrano/recipes"],
 @loaded_features=[],
 @logger=
  #<Capistrano::Logger:0x00000001f264c0
   @device=#<IO:<STDERR>>,
   @disable_formatters=nil,
   @level=0,
   @options={}>,
 @name=nil,
 @namespaces={},
 @original_procs={},
 @parent=nil,
 @preserve_roles=false,
 @roles={},
 @tasks={},
 @variable_locks=
  {:ssh_options=>%<8d5c2f16Mutex:0x00000001f26150>,
   :logger=>#<Mutex:0x00000001f25fc0>,
   :default_environment=>#<Mutex:0x00000001f25368>,
   :default_run_options=>#<Mutex:0x00000001f251b0>},
 @variables=
  {:ssh_options=>{},
   :logger=>
    #<Capistrano::Logger:0x00000001f264c0
     @device=#<IO:<STDERR>>,
     @disable_formatters=nil,
     @level=0,
     @options={}>,
   :default_environment=>{},
   :default_run_options=>{}}>
```

Capistranoの`Capfile, deploy.rb`などに出てくる単語がインスタンス変数としてたくさん出てくる。

### 継承チェック

各モジュールやクラスから、CLIツールでデプロイする時のCapitranoの各挙動が垣間見れる。

```ruby:Pry-Console
> config.class.ancestors
=> [Capistrano::Configuration,
 Capistrano::Configuration::Callbacks,
 Capistrano::Configuration::Actions::FileTransfer,
 Capistrano::Configuration::Actions::Inspect,
 Capistrano::Configuration::Actions::Invocation,
 Capistrano::Configuration::AliasTask,
 Capistrano::Configuration::Connections,
 Capistrano::Configuration::Execution,
 Capistrano::Configuration::Loading,
 Capistrano::Configuration::LogFormatters,
 Capistrano::Configuration::Namespaces,
 Capistrano::Configuration::Roles,
 Capistrano::Configuration::Servers,
 Capistrano::Configuration::Variables,
 Object,
 PP::ObjectMixin,
 Kernel,
 BasicObject]
```

### 出力を詳細に変更

ここで`@logger`をTRACEレベルにして、Capistranoが実行する内容を逐一表示するように設定。

```ruby:Pry-Console
> Capistrano::Logger.constants
=> [:IMPORTANT, :INFO, :DEBUG, :TRACE, :MAX_LEVEL, :COLORS, :STYLES]

> config.logger.level = Capistrano::Logger::TRACE
=> 3
```


## Namespaceとタスクを定義

この段階ではタスクは何もない。
confingについた`#namespace`はタスクを動的に定義する、これはどういうことか。

```ruby:Pry-Console
> config.namespace :hello do
*   task :world do
*     run "echo Hello World ?"
*   end  
* end  
=> #<Proc:0x000000022dadf0@/usr/local/rvm/gems/ruby-1.9.3-p286@cap/gems/capistrano-2.13.5/lib/capistrano/configuration/namespaces.rb:83 (lambda)>
```

Procだった、どこかでcallする必要がある。

### タスクProcの格納先

configオブジェクトをたどっていく。

```ruby:Pry-Console
> config.namespaces.keys
=> [:hello]

> config.namespaces[:hello].tasks.keys
=> [:world]

> config.namespaces[:hello].tasks[:world].body
=> #<Proc:0x000000022db110@(pry):9>
```

あった、`#call`を叩いてみよう。中身の`run`が呼ばれる。

```ruby:Pry-Console
> config.namespaces[:hello].tasks[:world].body.call
  * executing "echo Hello World ?"
Capistrano::NoMatchingServersError: no servers found to match {:eof=>true}
from /usr/local/rvm/gems/ruby-1.9.3-p286@cap/gems/capistrano-2.13.5/lib/capistrano/configuration/connections.rb:174:in `execute_on_servers'
```

`run`は対象のサーバを必要とするのでサーバのリストを取れるようにする。


## Role(server)の追加

namespaceの方法から推測できるが、これもレシピ通りに定義出来る。`config.`を除けばCapistrano使いにはおなじみの内容。
こういう内部DSLの実装は[メタプログラミングRuby](http://www.amazon.co.jp/dp/4048687158)をみたら何となくわかる。

```ruby:Pry-Console
> config.server "localhost", :web
=> [:web]
```

空だった`@roles`にサーバリストが代入された。

```ruby:Pry-Console
> config.roles          
=> {:web=>
  #<Capistrano::Role:0x00000002427870
   @dynamic_servers=[],
   @static_servers=[localhost]>}
```

### タスクの実行を2通りの書式で

まず先ほどコケた`#call`。

```ruby:Pry-Console
> config.namespaces[:hello].tasks[:world].body.call
  * executing "echo Hello World ?"
    servers: ["localhost"]
    [localhost] executing command
 ** [out :: localhost] Hello World ?
    command finished in 171ms
=> nil
```

これは次の書式でも一緒になる、要はCapistranoレシピと一緒だ。

```ruby:Pry-Console
> config.hello.world 
  * 2012-11-07 09:12:24 executing `hello:world'
  * executing "echo Hello World ?"
    servers: ["localhost"]
    [localhost] executing command
 ** [out :: localhost] Hello World ?
    command finished in 7ms
=> nil
```

namespaceによってメソッドとして定義されてるのでconfigから呼べる。

```ruby:Pry-Console
> config.methods(false)
=> [:hello,
 :_cset,
 :scm_default,
 :depend,
 :with_env,
 :run_locally,
 :try_sudo,
 :try_runner]
```

これでひとまずRoleへのタスク実行が出来た。


## いつもの cap deploy をloadしてみる

`@load_paths`には組込みレシピへのパスがある。

```ruby:Pry-Console
> config.load_paths
=> [".", "/usr/local/rvm/gems/ruby-1.9.3-p286@cap/gems/capistrano-2.13.5/lib/capistrano/recipes"]
```

`#load`を使うと組み込める、これでいつもの`deploy`系タスクが定義される。

```ruby:Pry-Console
> config.load "deploy"
=> ["deploy"]

> config.namespaces.keys
=> [:hello, :deploy]
```

CLI実行を補足するとcapifyで作成したCapfileにある`load "deploy"`でこれと同じことを行なっている事を確認できる。

### 組み込みのDeployを実行してみる

ロードした`deploy`レシピのタスク一覧を確認すると、おなじみの面々が並んでいることがわかる。

```ruby:Pry-Console
> config.namespaces[:deploy].tasks.keys
=> [:default,
 :setup,
 :update,
 :update_code,
 :finalize_update,
 :symlink,
 :create_symlink,
 :upload,
 :restart,
 :migrate,
 :migrations,
 :cleanup,
 :check,
 :cold,
 :start,
 :stop]
```

#### 軽そうなdeploy:setupから

もちろん`#set` が使えるので、setupに必須のパラメータ`:deploy_to`を定義したら`setup`実行。

```ruby:Pry-Console
> config.set :deploy_to, "/opt/deploy/hello"
=> "/opt/deploy/hello"


> config.deploy.setup
  * 2012-11-07 09:34:17 executing `deploy:setup'
  * executing "sudo -p 'sudo password: ' mkdir -p /opt/deploy/hello /opt/deploy/hello/releases /opt/deploy/hello/shared /opt/deploy/hello/shared/system /opt/deploy/hello/shared/log /opt/deploy/hello/shared/pids"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 14ms
  * executing "sudo -p 'sudo password: ' chmod g+w /opt/deploy/hello /opt/deploy/hello/releases /opt/deploy/hello/shared /opt/deploy/hello/shared/system /opt/deploy/hello/shared/log /opt/deploy/hello/shared/pids"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 11ms
=> nil


> Dir.glob("/opt/deploy/hello/**/")
=> ["/opt/deploy/hello/",
 "/opt/deploy/hello/shared/",
 "/opt/deploy/hello/shared/system/",
 "/opt/deploy/hello/shared/log/",
 "/opt/deploy/hello/shared/pids/",
 "/opt/deploy/hello/releases/"]
```

Capistranoぽいディレクトリツリーが出来た。

### コードベースからデプロイする(update_code, create_symlink)

`deploy`レシピを組み込んだことにより作成されたAttributeをチェック。

```ruby:Pry-Console
> config.namespaces[:deploy].variables.keys
=> [:ssh_options,
 :logger,
 :default_environment,
 :default_run_options,
 :application,
 :repository,
 :scm,
 :deploy_via,
 :deploy_to,
 :revision,
 :rails_env,
 :rake,
 :source,
 :real_revision,
 :strategy,
 :release_name,
 :version_dir,
 :shared_dir,
 :shared_children,
 :current_dir,
 :releases_path,
 :shared_path,
 :current_path,
 :release_path,
 :releases,
 :current_release,
 :previous_release,
 :current_revision,
 :latest_revision,
 :previous_revision,
 :run_method,
 :latest_release]
```

要るものだけ上書き＆定義する、コードベースの情報があればOK。

```ruby:Pry-Console
> config.set :scm, 'git'
=> "git"

> config.set :repository, 'https://github.com/higanworks/Files_and_Tasks.git'
=> "https://github.com/higanworks/Files_and_Tasks.git"


> config.deploy.check          
  * 2012-11-07 10:27:26 executing `deploy:check'
  * executing "test -d /opt/deploy/hello/releases"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 7ms
  * executing "test -w /opt/deploy/hello"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 5ms
  * executing "test -w /opt/deploy/hello/releases"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 5ms
  * executing "which git"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 7ms
You appear to have all necessary dependencies installed
=> nil
```

#### update_code

`Github`からコードを設置。

```ruby:Pry-Console
> config.deploy.update_code
  * 2012-11-07 10:11:19 executing `deploy:update_code'
  * executing "git clone -q https://github.com/higanworks/Files_and_Tasks.git /opt/deploy/hello/releases/20121107100343 && cd /opt/deploy/hello/releases/20121107100343 && git checkout -q -b deploy ea5f5e0f9e782305dcd3c27a091fe6c8387760c6 && (echo ea5f5e0f9e782305dcd3c27a091fe6c8387760c6 > /opt/deploy/hello/releases/20121107100343/REVISION)"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 6293ms
  * 2012-11-07 10:11:26 executing `deploy:finalize_update'
  * executing "chmod -R -- g+w /opt/deploy/hello/releases/20121107100343 && rm -rf -- /opt/deploy/hello/releases/20121107100343/public/system && mkdir -p -- /opt/deploy/hello/releases/20121107100343/public/ && ln -s -- /opt/deploy/hello/shared/system /opt/deploy/hello/releases/20121107100343/public/system && rm -rf -- /opt/deploy/hello/releases/20121107100343/log && ln -s -- /opt/deploy/hello/shared/log /opt/deploy/hello/releases/20121107100343/log && rm -rf -- /opt/deploy/hello/releases/20121107100343/tmp/pids && mkdir -p -- /opt/deploy/hello/releases/20121107100343/tmp/ && ln -s -- /opt/deploy/hello/shared/pids /opt/deploy/hello/releases/20121107100343/tmp/pids"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 22ms
  * executing "find /opt/deploy/hello/releases/20121107100343/public/images\\ /opt/deploy/hello/releases/20121107100343/public/stylesheets\\ /opt/deploy/hello/releases/20121107100343/public/javascripts -exec touch -t -- 201211071011.26 {} ';'; true"
    servers: ["localhost"]
    [localhost] executing command
*** [err :: localhost] find: `/opt/deploy/hello/releases/20121107100343/public/images /opt/deploy/hello/releases/20121107100343/public/stylesheets /opt/deploy/hello/releases/20121107100343/public/javascripts': No such file or directory
    command finished in 17ms
=> nil
```

#### create_symlink

`load "deploy"`で自動的に定義されたattributeは、各タスクで使い回されている。

```ruby:Pry-Console
> config.release_name
=> "20121107100343"

> config.release_path      
=> "/opt/deploy/hello/releases/20121107100343"
```


```ruby:Pry-Console
> config.deploy.create_symlink 
  * 2012-11-07 10:22:00 executing `deploy:create_symlink'
  * executing "rm -f /opt/deploy/hello/current && ln -s /opt/deploy/hello/releases/20121107100343 /opt/deploy/hello/current"
    servers: ["localhost"]
    [localhost] executing command
    command finished in 11ms
=> nil
```


## まとめ
`Capistrano::Configuration`をうまいこと操作できれば、Dev向けにシンプルな共有デプロイツールを作れそうな気がする。

