
いろいろな兼ね合いの折衷案で、ログを1行ずつのJSONでファイルに書いて行きたい。
フォーマッタ自体があまり難しくは無いためか適当なGemが無く、数年前のプロジェクトでRails用に使っていたやつを切り出してもらった。

[ZCloud-Firstserver/niboshi_json_formatter](https://github.com/ZCloud-Firstserver/niboshi_json_formatter "ZCloud-Firstserver/niboshi_json_formatter")

さあ、Let's [Log Everything as JSON. Make Your Life Easier. | Treasure Data Community Blog](http://blog.treasure-data.com/post/21881575472/log-everything-as-json-make-your-life-easier "Log Everything as JSON. Make Your Life Easier. | Treasure Data Community Blog")


## Gemを追加する。

```Gemfile
gem 'niboshi_json_formatter'
```


## RailsでJSON形式のログを出力

さっそくRailsプロジェクトで使おう。フォーマッタの指定でOK。


```config/environments/development.rb
Rails.application.configure do

... 色々Config

  config.log_formatter = Niboshi::JsonFormatter.new
end
```


適当にgenerateしたRailsプロジェクトで`rails s`したあとに`curl localhost:3000`して挙動を見ます。

```
% ./bin/rails server                   
=> Booting WEBrick
=> Rails 4.1.8 application starting in development on http://0.0.0.0:3000
=> Run `rails server -h` for more startup options
=> Notice: server is listening on all interfaces (0.0.0.0). Consider using 127.0.0.1 (--binding option)
=> Ctrl-C to shutdown server
[2014-12-05 12:46:15] INFO  WEBrick 1.3.1
[2014-12-05 12:46:15] INFO  ruby 2.1.5 (2014-11-13) [x86_64-darwin14.0]
[2014-12-05 12:46:15] INFO  WEBrick::HTTPServer#start: pid=35214 port=3000

## ここにHTTPリクエスト


{"time":1417751189.197508,"formatted_time":"2014-12-05T12:46:29.197+09:00","hostname":"sawambp01.local","pid":35214,"tid":70110014089680,"level":"DEBUG","program_name":null,"message":""}
{"time":1417751189.197817,"formatted_time":"2014-12-05T12:46:29.197+09:00","hostname":"sawambp01.local","pid":35214,"tid":70110014089680,"level":"DEBUG","program_name":null,"message":""}
{"time":1417751189.198165,"formatted_time":"2014-12-05T12:46:29.198+09:00","hostname":"sawambp01.local","pid":35214,"tid":70110014089680,"level":"INFO","program_name":null,"message":"Started GET \"/\" for 127.0.0.1 at 2014-12-05 12:46:29 +0900"}
{"time":1417751189.283526,"formatted_time":"2014-12-05T12:46:29.283+09:00","hostname":"sawambp01.local","pid":35214,"tid":70110014089680,"level":"INFO","program_name":null,"message":"Processing by Rails::WelcomeController#index as */*"}
{"time":1417751189.297908,"formatted_time":"2014-12-05T12:46:29.297+09:00","hostname":"sawambp01.local","pid":35214,"tid":70110014089680,"level":"INFO","program_name":null,"message":"  Rendered /Users/sawanoboriyu/.rvm/gems/ruby-2.1.5/gems/railties-4.1.8/lib/rails/templates/rails/welcome/index.html.erb (1.1ms)"}
{"time":1417751189.2984362,"formatted_time":"2014-12-05T12:46:29.298+09:00","hostname":"sawambp01.local","pid":35214,"tid":70110014089680,"level":"INFO","program_name":null,"message":"Completed 200 OK in 15ms (Views: 7.4ms | ActiveRecord: 0.0ms)"}
```

JSONだー。


## Ruby (Logger) でJSON形式のログを出力

`Niboshi::JsonFormatter`はもともとLoggerのFormatterなので、通常のRubyプロジェクトでもフォーマッタ指定でよい。


```
[1] pry(main)> require 'niboshi_json_formatter'
[2] pry(main)> log = Logger.new(STDOUT)
[3] pry(main)> log.formatter = Niboshi::JsonFormatter.new


[4] pry(main)> log.info "foo bar"
{"time":1417752284.592915,"formatted_time":"2014-12-05 13:04:44 +0900","hostname":"sawambp01.local","pid":35686,"tid":70247764089840,"level":"INFO","program_name":null,"message":"foo bar"}
=> true
```

JSONだー。


## ChefでJSON形式のログを出力

ChefのLoggerとFormtterは役割的に色々と独自の実装があって面倒。
設定ファイルにちょっと仕込んで、ファイルのフリして`log_location`に突っ込む。


```client.rb
require 'logger'
require 'niboshi_json_formatter'

Logger.class_eval do
  attr_accessor :sync, :formatter, :close
  def write(msg)
    data = msg.match(/(\[.+?\]) ([\w]+):(.*)$/)
    self.send(data[2].downcase.to_sym, data[3])
  end
end

logger = Logger.new(STDOUT)
logger.formatter = Niboshi::JsonFormatter.new


log_level :info
log_location logger
```


ローカルモードで試してみます。Chefはトランザクションが長くてログが細かいので生ログよりレポートのほうがいいんですがそれはそれで。


```
% ./b_bin/chef-client -z -c client.rb -Fnull 
ffi-yajl/json_gem is deprecated, these monkeypatches will be dropped shortly
{"time":1417752132.030346,"formatted_time":"2014-12-05 13:02:12 +0900","hostname":"sawambp01.local","pid":35552,"tid":70241644600300,"level":"WARN","program_name":null,"message":" No cookbooks directory found at or above current directory.  Assuming /Users/sawanoboriyu/Dev/worktmp/chef_jsonlog."}
{"time":1417752132.740726,"formatted_time":"2014-12-05 13:02:12 +0900","hostname":"sawambp01.local","pid":35552,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Starting chef-zero on host localhost, port 8889 with repository at repository at /Users/sawanoboriyu/Dev/worktmp/chef_jsonlog"}
{"time":1417752132.967211,"formatted_time":"2014-12-05 13:02:12 +0900","hostname":"sawambp01.local","pid":35552,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Forking chef instance to converge..."}
{"time":1417752132.975351,"formatted_time":"2014-12-05 13:02:12 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" *** Chef 11.16.4 ***"}
{"time":1417752132.97606,"formatted_time":"2014-12-05 13:02:12 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Chef-client pid: 35553"}
{"time":1417752136.541346,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Run List is []"}
{"time":1417752136.54162,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Run List expands to []"}
{"time":1417752136.54192,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Starting Chef Run for sawambp01.local"}
{"time":1417752136.542167,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Running start handlers"}
{"time":1417752136.542368,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Start handlers complete."}
{"time":1417752136.546394,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" HTTP Request Returned 404 Not Found : Object not found: /reports/nodes/sawambp01.local/runs"}
{"time":1417752136.550326,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Loading cookbooks []"}
{"time":1417752136.5517929,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"WARN","program_name":null,"message":" Node sawambp01.local has an empty run list."}
{"time":1417752136.586014,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Chef Run complete in 0.044041 seconds"}
{"time":1417752136.586442,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Running report handlers"}
{"time":1417752136.586716,"formatted_time":"2014-12-05 13:02:16 +0900","hostname":"sawambp01.local","pid":35553,"tid":70241644600300,"level":"INFO","program_name":null,"message":" Report handlers complete"}
```

JSONだー。

