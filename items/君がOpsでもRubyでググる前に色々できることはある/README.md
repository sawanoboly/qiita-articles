<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

君がOpsでもシリーズ。

Opscode Chefをつかう過程で身についたRubyの使い方を書いておく。ネタの仕入れ元はほとんど[メタプログラミングRuby](http://ascii.asciimw.jp/books/books/detail/978-4-04-868715-7.shtml)です。


## irb (pry) を立ち上げる

Rubyはスクリプト型なのでインタプリタが使える。

### Irb

とりあえず付属のIrb(Interactive Ruby)を使い、文法など確認しよう。
`ruby`と同じパスに`irb`があるはずだ。

```ruby:Irb_Prompt
$ irb
irb(main):001:0> puts "hoge"
hoge
=> nil

irb(main):002:0> require 'chef'
=> true

irb(main):003:0> ohai = Ohai::System.new
=> #<Ohai::System:0x00000001d67698 @data={}, @seen_plugins={}, @providers={}, @plugin_path="", @hints={}>
```

単純な書式やその辺のクラスを確認するならIrbを。

### Pry

より高機能なPryもある。 ＞ [http://pryrepl.org](http://pryrepl.org)
環境が許せばPry-Docと一緒につかおう。

`gem install pry pry-doc --no-ri --no-rdoc`

```ruby:Pry_Prompt1
$ pry
[1] pry(main)> puts 'hogehoge'
hogehoge
=> nil

[2] pry(main)> require 'chef'
=> true

[3] pry(main)> ohai = Ohai::System.new
=> #<Ohai::System:0x0000000437cec8
 @data={},
 @hints={},
 @plugin_path="",
 @providers={},
 @seen_plugins={}>
```

cdでインスタンスの中に潜り込める。

```ruby:Pry_Prompt2
[4] pry(main)> cd ohai
[5] pry(#<Ohai::System>):1> ls
Ohai::Mixin::FromFile#methods: from_file
Ohai::Mixin::Command#methods: run_command_backend  run_command_unix  run_command_windows
Ohai::System#methods: 
  []   _require_plugin  attribute?        collect_providers  data=  from             get_attribute  hints   json_pretty_print  provides         require_plugin  seen_plugins=  set_attribute
  []=  all_plugins      attributes_print  data               each   from_with_regex  hint?          hints=  method_missing     refresh_plugins  seen_plugins    set            to_json      
self.methods: __pry__
instance variables: @data  @hints  @plugin_path  @providers  @seen_plugins
locals: _  __  _dir_  _ex_  _file_  _in_  _out_  _pry_
```

`$`記号をつけてメソッドを叩こうとすると、表示できる時はソースを見ることができる。

```ruby:Pry_Prompt3
[11] pry(main)> $ ohai.from

From: /opt/chef/embedded/lib/ruby/gems/1.9.1/gems/ohai-6.18.0/lib/ohai/system.rb @ line 65:
Owner: Ohai::System
Visibility: public
Number of lines: 5

def from(cmd)
  status, stdout, stderr = run_command(:command => cmd)
  return "" if stdout.nil? || stdout.empty?
  stdout.strip
end
```


## クラスを調べる

Rubyはほとんどが何が何でもクラスになっているので、とりあえずクラスについて調べるのが吉。

とりあえずIrbで。

`#class`でインスタンスのクラスが分かる。

```ruby
> [].class
=> Array
```

なんかややこしくなってきたらとりあえずクラスを見る。

```ruby
> require 'mixlib/shellout'
=> true
> shell_out = Mixlib::ShellOut.new('pwd')
=> <Mixlib::ShellOut#17592600: command: 'pwd' process_status: nil stdout: '' stderr: '' child_pid: nil environment: {"LC_ALL"=>"C"} timeout: 600 user:  group:  working_dir:  >

> shell_out.class
=> Mixlib::ShellOut
```

`#inspect`でインスタンス変数やらをちらほら見れる。

```
 shell_out.inspect
=> "<Mixlib::ShellOut#17592600: command: 'pwd' process_status: nil stdout: '' stderr: '' child_pid: nil environment: {\"LC_ALL\"=>\"C\"} timeout: 600 user:  group:  working_dir:  >"
```

`#ancestors `でクラスの継承がわかる。

```
> shell_out.class.ancestors
=> [Mixlib::ShellOut, Mixlib::ShellOut::Unix, Object, Kernel, BasicObject]

```


## #methods, #instance_methods, #instance_variables で大体なんとかなる

この3つのメソッドがあれば一覧の為にマニュアルを引く手間はとても省ける。

### instance_variables さん

インスタンス変数の一覧が見れるぞ。
`instance_variable_get/set`もついでに使えるようになる。

```
0> shell_out.instance_variables
=> [:@stderr, :@stdout, :@live_stream, :@input, :@log_level, :@log_tag, :@environment, :@cwd, :@valid_exit_codes, :@command]
```

クラスの大体の役割がわかって良いね。

```
> shell_out.instance_variables
=> [:@stderr, :@stdout, :@live_stream, :@input, :@log_level, :@log_tag, :@environment, :@cwd, :@valid_exit_codes, :@command]
```


### methods さん

特異メソッドがわかります。
変なことをしてなければ、クラスに対して実行するので`#class`をいっこはさむ。
`(false)`をつけると継承したものを除外できてよいよい。

```
> [].class.methods(false)
=> [:[], :try_convert]
```

instance_methodsほどは使わないけどたまに見る。

### instance_methods さん

インスタンスが持っているメソッドを見れるぞ。これも`#class`をいっこはさむ。
`(false)`をつけると継承したものを除外というのも同じ。

というか`(false)`付けないとつらい。

```
> [].class.instance_methods(false)
=> [:inspect, :to_s, :to_a, :to_ary, :frozen?, :==, :eql?, :hash, :[], :[]=, :at, :fetch, :first, :last, :concat, :<<, :push, :pop, :shift, :unshift, :insert, :each, :each_index, :reverse_each, :length, :size, :empty?, :find_index, :index, :rindex, :join, :reverse, :reverse!, :rotate, :rotate!, :sort, :sort!, :sort_by!, :collect, :collect!, :map, :map!, :select, :select!, :keep_if, :values_at, :delete, :delete_at, :delete_if, :reject, :reject!, :zip, :transpose, :replace, :clear, :fill, :include?, :<=>, :slice, :slice!, :assoc, :rassoc, :+, :*, :-, :&, :|, :uniq, :uniq!, :compact, :compact!, :flatten, :flatten!, :count, :shuffle!, :shuffle, :sample, :cycle, :permutation, :combination, :repeated_permutation, :repeated_combination, :product, :take, :take_while, :drop, :drop_while, :pack]
```

irbの時はputsとかでちょっと見やすくする。

```
>> puts [].class.instance_methods(false).sort
&
*
+
-
<<
<=>
==
[]
[]=
assoc
at
clear
collect
collect!
combination
compact
compact!
concat
count
cycle
delete
delete_at
delete_if
drop
drop_while
each
each_index
empty?
eql?
fetch
fill
find_index
first
flatten
flatten!
frozen?
hash
include?
index
insert
inspect
join
keep_if
last
length
map
map!
pack
permutation
place
pop
pretty_print
pretty_print_cycle
product
push
rassoc
reject
reject!
repeated_combination
repeated_permutation
replace
reverse
reverse!
reverse_each
rindex
rotate
rotate!
sample
select
select!
shelljoin
shift
shuffle
shuffle!
size
slice
slice!
sort
sort!
sort_by!
take
take_while
to_a
to_ary
to_s
transpose
uniq
uniq!
unshift
values_at
zip
|
=> nil
```

一覧を眺めるだけで、できそうなこと、そうでもないことが何となく分かる。


## Chefんとき

`chef-shell`(Chefのインタプリタ)でさっきの3つを叩いて、ソースを見に行ったりする。

Pryでリソースに指定できる属性をマニュアル代わりに引いたりする。

```
pry(main)> require 'chef'
=> true

pry(main)> svc = Chef::Resource::Service.new('dummy')
=> <service[dummy] @name: "dummy" @noop: nil @before: nil @params: {} @provider: nil @allowed_actions: [:nothing, :enable, :disable, :start, :stop, :restart, :reload] @action: "nothing" @updated: false @updated_by_last_action: false @supports: {:restart=>false, :reload=>false, :status=>false} @ignore_failure: false @retries: 0 @retry_delay: 2 @source_line: nil @elapsed_time: 0 @resource_name: :service @service_name: "dummy" @enabled: nil @running: nil @parameters: nil @pattern: "dummy" @start_command: nil @stop_command: nil @status_command: nil @restart_command: nil @reload_command: nil @init_command: nil @priority: nil @startup_type: :automatic>

[4] pry(main)> ls svc
Chef::DSL::DataQuery#methods: data_bag  data_bag_item  search
Chef::Mixin::ParamsValidate#methods: lazy  set_or_return  validate
Chef::DSL::PlatformIntrospection#methods: platform?  platform_family?  value_for_platform  value_for_platform_family
Chef::DSL::RegistryHelper#methods: registry_data_exists?  registry_get_subkeys  registry_get_values  registry_has_subkeys?  registry_key_exists?  registry_value_exists?
Chef::Mixin::ConvertToClassName#methods: convert_to_class_name  convert_to_snake_case  filename_to_qualified_string  snake_case_basename
Chef::Mixin::Deprecation#methods: deprecated_ivar
Chef::Resource#methods: 
  action               defined_at             immediate_notifications  not_if_args           provider=                        retry_delay   subscribes              updated_by_last_action?
  after_created        delayed_notifications  inspect                  notifies              provider_for_action              retry_delay=  to_hash                 validate_action        
  allowed_actions      elapsed_time           is                       notifies_delayed      recipe_name                      run_action    to_json                 validate_resource_spec!
  allowed_actions=     enclosing_provider     load_prior_resource      notifies_immediately  recipe_name=                     run_context   to_s                  
  as_json              enclosing_provider=    method_missing           only_if               resolve_notification_references  run_context=  to_text               
  cookbook_name        epic_fail              name                     only_if_args          resource_name                    should_skip?  updated               
  cookbook_name=       events                 node                     params                resources                        source_line   updated=              
  cookbook_version     identity               noop                     params=               retries                          source_line=  updated?              
  customize_exception  ignore_failure         not_if                   provider              retries=                         state         updated_by_last_action
Chef::Resource::Service#methods: enabled  init_command  parameters  pattern  priority  reload_command  restart_command  running  service_name  start_command  status_command  stop_command  supports
instance variables: 
  @action           @elapsed_time    @init_command  @noop     @parameters  @priority        @resource_name    @retry_delay  @service_name   @startup_type    @supports              
  @allowed_actions  @enabled         @name          @not_if   @params      @provider        @restart_command  @run_context  @source_line    @status_command  @updated               
  @before           @ignore_failure  @node          @only_if  @pattern     @reload_command  @retries          @running      @start_command  @stop_command    @updated_by_last_action
```


ソース見に行ったり。

```ruby
> $ svc

From: /opt/chef/embedded/lib/ruby/gems/1.9.1/gems/chef-11.6.0/lib/chef/resource/service.rb @ line 24:
Class name: Chef::Resource::Service
Number of lines: 153

class Service < Chef::Resource

  identity_attr :service_name

  state_attrs :enabled, :running

  def initialize(name, run_context=nil)
    super
    @resource_name = :service
    @service_name = name
    @enabled = nil
    @running = nil
    @parameters = nil
    @pattern = service_name
    @start_command = nil
    @stop_command = nil
    @status_command = nil
    @restart_command = nil
    @reload_command = nil
    @init_command = nil
    @priority = nil
    @action = "nothing"
    @startup_type = :automatic
    @supports = { :restart => false, :reload => false, :status => false }
    @allowed_actions.push(:enable, :disable, :start, :stop, :restart, :reload)
  end
  
  def service_name(arg=nil)
    set_or_return(
      :service_name,
      arg,
      :kind_of => [ String ]
    )
  end
  
  0;34# regex for match against ps -ef when !supports[:has_status] && status == nil
  def pattern(arg=nil)
    set_or_return(
      :pattern,
      arg,
      :kind_of => [ String ]
    )
  end

-- 以下略 --
```



method missingには弱い。

