<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

タイムラインに`ServerSpec`の話題がそこそこ見えたので、サーバの受け入れテストに使えるかなと触ってみました。

考え方などとても良いと思いましたが、この先色々なプラットフォームへの対応を書いていくのはいささか大変に感じます。
なのでどうせRSpecを入れるなら、Chefの資産を活用する方向で似たようなものが作れないかと試して見ることにしました。

> 追記
> ServerSpecはサーバ側にRSpec入れるわけじゃないのね。
> https://github.com/mizzy/rspec-lxc-test-box/blob/master/spec/support/matchers/be_enabled.rb


## 概要

- Ohaiでプラットフォームを判別する
- プラットフォームからプロバイダを選択する
- テストのためにCurrentResourceをゲットする

## Ohaiでplatformを取得する

これはまあ、以前やった [Chefの心臓、Ohaiのアトリビュートを他のプログラムからも拝借したい](http://qiita.com/items/5ce72101f8dee906ccb4) ですよね。

`os, platform`プラグインの力を拝借しましょう。

```ruby:Irb-Out
> require 'chef'
=> true

> ohai = Ohai::System.new
=> #<Ohai::System:0x00000001c73318
 @data={},
 @hints={},
 @plugin_path="",
 @providers={},
 @seen_plugins={}>

> ohai._require_plugin('os')
=> true

> ohai._require_plugin('platform')
=> true

> ohai.data
=> {"languages"=>
  {"ruby"=>
    {"platform"=>"x86_64-solaris2.11",
     "version"=>"1.9.3",
     "release_date"=>"2013-02-22",
     "target"=>"x86_64-sun-solaris2.11",
     "target_cpu"=>"x86_64",
     "target_vendor"=>"sun",
     "target_os"=>"solaris2.11",
     "host"=>"x86_64-sun-solaris2.11",
     "host_cpu"=>"x86_64",
     "host_os"=>"solaris2.11",
     "host_vendor"=>"sun",
     "bin_dir"=>"/opt/local/bin",
     "ruby_bin"=>"/opt/local/bin/ruby193",
     "gems_dir"=>"/opt/local/lib/ruby/gems/1.9.3",
     "gem_bin"=>"/opt/local/bin/gem193"}},
 "kernel"=>
  {"name"=>"SunOS",
   "release"=>"5.11",
   "version"=>"joyent_20130226T234312Z",
   "machine"=>"i86pc",
   "modules"=>{}},
 "os"=>"solaris2",
 "os_version"=>"5.11",
 "platform_version"=>"5.11",
 "platform_build"=>"joyent_20130226T234312Z",
 "platform"=>"smartos",
 "platform_family"=>"smartos"}
```

```ruby:Irb-Out
> ohai.data[:platform]
=> "smartos"
```

判定OK。

## Platformから利用するProviderを選択する（課題 => 解決）

これどうしたら良いのかしら、Chefのソースを追う課題として置いときます。

> 追記：解決しました
> [ohaiのデータを元に、ChefのProvidersを自動選択してcurrent_resourceをゲット](http://qiita.com/items/e0ed969b49eb16fdd1f3)

## Providerを使ってCurrentResourceを取ってみる

気を取り直して、`service[redis]`が起動しているかをチェックしてみましょう。
必要なオブジェクトを適当にinitializeしていきます。



### `run_context`を作る

```ruby:Irc-Out
> node = Chef::Node.new
=> node[]

> events = Chef::EventDispatch::Dispatcher.new
=> #<Chef::EventDispatch::Dispatcher:0x00000002ab2460 @subscribers=[]>

> run_context = Chef::RunContext.new(node, {}, events)
=> #<Chef::RunContext:0x00000001a72398
 @cookbook_collection={},
 @definitions={},
 @delayed_notification_collection={},
 @events=#<Chef::EventDispatch::Dispatcher:0x00000002ab2460 @subscribers=[]>,
 @immediate_notification_collection={},
 @loaded_attributes={},
 @loaded_recipes={},
 @node=node[],
 @resource_collection=
  #<Chef::ResourceCollection:0x00000001a723e8
   @insert_after_idx=nil,
   @resources=[],
   @resources_by_name={}>>
```
### newとcurrentの`resource`を作る

いわゆる`service[redis]`リソースを作ります、`Chef::Resource::Service`のインスタンスですね。

```ruby:Irc-Out
> new_resource = Chef::Resource::Service.new('redis')
=> <service[redis] @name: "redis" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "redis" @enabled: nil @running: nil @parameters: nil @pattern: "redis" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>

> current_resource = Chef::Resource::Service.new('redis')
=> <service[redis] @name: "redis" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "redis" @enabled: nil @running: nil @parameters: nil @pattern: "redis" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>
```

### Providersを指定する

リソースを取得するためのプロバイダを作成します、今回は`Chef::Provider::Service::Solaris`を指定します。
これを`ohai`の結果と連動したいところです。

```ruby:Irc-Out
> provider = Chef::Provider::Service::Solaris.new(new_resource, run_context)
=> #<Chef::Provider::Service::Solaris:0x000000018d6020
 @action=nil,
 @converge_actions=nil,
 @current_resource=nil,
 @enabled=nil,
 @init_command="/usr/sbin/svcadm",
 @new_resource=
  <service[redis] @name: "redis" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "redis" @enabled: nil @running: nil @parameters: nil @pattern: "redis" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>,
 @run_context=
  #<Chef::RunContext:0x00000001a72398
   @cookbook_collection={},
   @definitions={},
   @delayed_notification_collection={},
   @events=#<Chef::EventDispatch::Dispatcher:0x00000002ab2460 @subscribers=[]>,
   @immediate_notification_collection={},
   @loaded_attributes={},
   @loaded_recipes={},
   @node=node[],
   @resource_collection=
    #<Chef::ResourceCollection:0x00000001a723e8
     @insert_after_idx=nil,
     @resources=[],
     @resources_by_name={}>>,
 @status_command="/bin/svcs -l">
```
`Solaris`らしくなってきました。


### CurrentResourceの状態をチェックする

では締めに、`service[redis]`の`current_resource`を採取します。

```ruby:Irc-Out
> provider.load_current_resource
=> <service[redis] @name: "redis" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "redis" @enabled: true @running: true @parameters: nil @pattern: "redis" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>

> provider.load_current_resource.enabled
=> true
```

取れました、これならChefとOhaiの資産が活用できそうです。`load_current_resource `に文句があったらChefにプルリクすれば済むわけで。

## RSpec的にチェックしたい場合

ヘルパーで色々やらせて、こんな感じでSpecに起こせればいい感じにサーバの受け入れテストが書けそうです。

`expect(service[hogehoge].load_current_resource.enabled).to be_true`

早いうちに残課題の`Providers`選択の所を追いたいですね。

> 追記：`Providers`解決しました
> [ohaiのデータを元に、ChefのProvidersを自動選択してcurrent_resourceをゲット](http://qiita.com/items/e0ed969b49eb16fdd1f3)
