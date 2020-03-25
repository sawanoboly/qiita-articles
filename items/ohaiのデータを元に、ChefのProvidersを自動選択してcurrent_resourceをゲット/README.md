<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

前回の記事 [ChefのProvidersを活用してServerSpecめいたものを作る、とりあえず一歩目](http://qiita.com/items/26971eb6f4a7efb5bb69) で始めたサーバー構築テストに向けてのChef資産の再利用の続きです。

課題だったProvidersの再利用ができました、ソースリーディング協力 HT:  [@ogomr](http://qiita.com/users/ogomr),[@sutetotanuki](http://qiita.com/users/sutetotanuki)

## 特異メソッド`Chef::Platform.find_provider`

名前からしてコレっぽいという理由でソースからピックアップしたのが`find_provider`です。

```ruby:lib/chef/platform/provider_mapping.rb
# -- snip --
def find_provider(platform, version, resource_type)
  provider_klass = explicit_provider(platform, version, resource_type) ||
                   platform_provider(platform, version, resource_type) ||
                   resource_matching_provider(platform, version, resource_type)

  raise ArgumentError, "Cannot find a provider for #{resource_type} on #{platform} version #{version}" if provider_klass.nil?

  provider_klass
end
# --snip --
```

なるほどコレっぽい。

## Providerの特定

早速叩いてみよう特異メソッドなので直接ひっぱたいてOK、`resource_type `はシンボルで渡します。

```ruby;Irb-Out
> Chef::Platform.find_provider(ohai.data[:platform], ohai.data[:platform_version],:service)
=> Chef::Provider::Service::Solaris

> Chef::Platform.find_provider(ohai.data[:platform], ohai.data[:platform_version],:package)
=> Chef::Provider::Package::SmartOS
```

自動判別OK!


## 自動判別からのCurrentResourceチェック

前回決め打ちだったProvidersを`Chef::Platform.find_provider`に置き換えてみます。

### 自動判別

```ruby;Irb-Out
> new_resource = Chef::Resource::Service.new('redis')
> current_resource = Chef::Resource::Service.new('redis')

> provider = Chef::Platform.find_provider(ohai.data[:platform], ohai.data[:platform_version],:service).new(new_resource, run_context)
=> #<Chef::Provider::Service::Solaris:0x00000002272a70
 @action=nil,
 @converge_actions=nil,
 @current_resource=nil,
 @enabled=nil,
 @init_command="/usr/sbin/svcadm",
 @new_resource=
  <service[redis] @name: "redis" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "redis" @enabled: nil @running: nil @parameters: nil @pattern: "redis" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>,
 @run_context=
  #<Chef::RunContext:0x00000002ad69f0
   @cookbook_collection={},
   @definitions={},
   @delayed_notification_collection={},
   @events=#<Chef::EventDispatch::Dispatcher:0x00000002c02770 @subscribers=[]>,
   @immediate_notification_collection={},
   @loaded_attributes={},
   @loaded_recipes={},
   @node=node[],
   @resource_collection=
    #<Chef::ResourceCollection:0x00000002ad69c8
     @insert_after_idx=nil,
     @resources=[],
     @resources_by_name={}>>,
 @status_command="/bin/svcs -l">
```

クラスを自動判別して目当てのオブジェクトができました。 `=> #<Chef::Provider::Service::Solaris:0x00000002272a70`

### CurrentResouceチェック

```ruby;Irb-Out
> provider.load_current_resource
=> <service[redis] @name: "redis" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "redis" @enabled: true @running: true @parameters: nil @pattern: "redis" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>

> provider.load_current_resource.enabled
=> true
```

できました、これでChef資産の再利用に目処がたちましたね。
