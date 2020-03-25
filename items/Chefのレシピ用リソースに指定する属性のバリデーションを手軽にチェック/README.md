<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

OpscodeChefのレシピで使えるリソースは、属性のバリデーションをそれなりにやっています。
リソースが使えるケースでは、なるべく使ったほうがミスを防げるかもしれません。

じゃあどんなバリデーションなのか軽く確かめる方法。

## Chef::Resouceのインスタンスをつくる

Chef::Resouce以下のクラスは名前を渡せばパラメータを全部デフォルトでインスタンスが作成されます。

Cronリソースで色々確かめてみます。

```ruby:Pry_ChefResourceCron
> require 'chef'
=> true

> cron = Chef::Resource::Cron.new 'my_cron'
=>
<cron[my_cron]
  @name: "my_cron"
  @noop: nil
  @before: nil
  @params: {}
  @provider: nil
  @allowed_actions: [:nothing, :create, :delete]
  @action: :create @updated: false @updated_by_last_action: false
  @supports: {}
  @ignore_failure: false
  @retries: 0
  @retry_delay: 2
  @source_line: nil
  @elapsed_time: 0
  @resource_name: :cron
  @minute: "*"
  @hour: "*"
  @day: "*"
  @month: "*"
  @weekday: "*"
  @command: nil
  @user: "root"
  @mailto: nil
  @path: nil
  @shell: nil
  @home: nil
  @environment: {}
> 
```

## 属性に値を入れてみて確認する

属性の名前はそのままメソッドになっていますので叩けばOK。このように。

```ruby:Pry_CronResource
## 何も渡さなければ現在の値
> cron.minute
=> "*"

## なにか渡せば代入
> cron.minute 50
=> "50"

## バリデーションに引っかかる例１、99分
> cron.minute 99
RangeError: RangeError
from /Users/sawanoboriyu/Dev/Reading/chef/lib/chef/resource/cron.rb:56:in 'minute'

## バリデーションに引っかかる例２、コマンドラインがStringでない
> cron.command 12
Chef::Exceptions::ValidationFailed: Option command must be a kind of String!  You passed 12.
from /Users/sawanoboriyu/Dev/Reading/chef/lib/chef/mixin/params_validate.rb:150:in '_pv_kind_of'
```


## 過信は禁物

分のバリデーションをもう少し試してみましょう。

```ruby:Pry
> cron.minute "abcde"
=> "abcde"
```

そんなチェックで大丈夫か。


### Chef-Applyで実験

そらFailしますわ、でも強行しないようになってるだけマシと思おう。

```shell:chef-apply_with_invalid_parameter
# cat <<EOL | chef-apply -s
> cron 'hoge' do
>   minute 'abcde'
> end
> EOL
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * cron[hoge] action create
================================================================================
Error executing action `create` on resource 'cron[hoge]'
================================================================================

Chef::Exceptions::Cron
----------------------
Error updating state of hoge, exit: 1

Resource Declaration:
---------------------
# In /tmp/recipe-temporary-file20130920-9634-1ps5rqz

  1: cron 'hoge' do
  2:   minute 'abcde'
  3: end

Compiled Resource:
------------------
# Declared in /tmp/recipe-temporary-file20130920-9634-1ps5rqz:1:in `run_chef_recipe'

cron("hoge") do
  action :create
  retries 0
  retry_delay 2
  minute "abcde"
  hour "*"
  day "*"
  month "*"
  weekday "*"
  user "root"
  cookbook_name "(chef-apply cookbook)"
  recipe_name "(chef-apply recipe)"
end

[2013-09-20T05:18:47+00:00] FATAL: Stacktrace dumped to /var/chef/cache/chef-stacktrace.out
[2013-09-20T05:18:47+00:00] FATAL: Chef::Exceptions::Cron: cron[hoge] ((chef-apply cookbook)::(chef-apply recipe) line 1) had an error: Chef::Exceptions::Cron: Error updating state of hoge, exit: 1
```

ちなみに上のChefApplyでChefが何をしようとして、何が`exit 1`だったのかの答えはこちら。

```shell:bash
# echo '* abcde * * * ' | crontab -u root -
"-":0: bad hour
errors in crontab file, can't install.
```

