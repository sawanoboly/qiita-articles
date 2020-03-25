<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[Redis Sentinel][sentinel_doc]で構成されたRedisのクラスタ環境はそこそこに複雑です、クライアントはRedisがSentinel環境であることを意識して専用のドライバを使うほうがよいでしょう。

[sentinel_doc]:http://redis.io/topics/sentinel
[sidekiq_url]:http://sidekiq.org
[redisn_url]:https://github.com/resque/redis-namespace
[rubysen_url]:https://github.com/flyerhzm/redis-sentinel

Rubyでよく使うRedisのライブラリ、[redis-namespace][redisn_url]と [Sidekiq][sidekiq_url]をRedis Sentinel 環境の環境で使ってみました。

----

修正情報：sidekiq対応がモンキーパッチでしたが、Sidekiq開発者のPerham氏から直々にツッコミを頂いたので記事を修正。

>  This is how you should configure Redis Sentinel.  Please don't monkeypatch. https://t.co/gKf98gnivd

----

[redis-sentinel][rubysen_url]というGemがあるので、それを利用することにします。

## redis-namespace with redis-sentinel

まず[redis-sentinel][rubysen_url]のサンプルで使い方を確認します。`Redis::Client`をオープンして`sentinel`関係の設定がうまいことやってくれるようになっていました。テスト環境起動用のコンフィグもついていたのですぐに試せます。

```ruby:example/test.rb 
require 'redis'
require 'redis-sentinel'

redis = Redis.new(:master_name => "example-test",
                  :sentinels => [
                    {:host => "localhost", :port => 26379},
                    {:host => "localhost", :port => 26380}
                  ])
redis.set "foo", "bar"

while true
  begin
    puts redis.get "foo"
  rescue => e
    puts "failed?", e
  end
  sleep 1
end
```

このサンプルから[redis-namespace][redisn_url]を使うようにしたのがこちら。
普段の使い方と通りに[Redis Sentinel][sentinel_doc]の環境に適応させることができました。

```ruby:example/test_name.rb 
require 'redis'
require 'redis-namespace'
require 'redis-sentinel'

redis_s = Redis.new(:master_name => "example-test",
                  :sentinels => [
                    {:host => "localhost", :port => 26379},
                    {:host => "localhost", :port => 26380}
                  ])

redis = Redis::Namespace.new(:myspace, :redis => redis_s)
redis.set "foo", "bar"

while true
  begin
    puts redis.get "foo"
  rescue => e
    puts "failed?", e
  end
  sleep 1
end
```

[redis-namespace][redisn_url]に関しては特に考慮なしでよさそうですね。


## Sidekiq with redis-sentinel

次は[Sidekiq][sidekiq_url]です、まずは付属のサンプル。
`sinatra`のWEBインターフェースからキューして、ワーカに処理させています。



```ruby:sidekiq/examples/sinkiq.rb 
# Make sure you have Sinatra installed, then start sidekiq with
# ./bin/sidekiq -r ./examples/sinkiq.rb
# Simply run Sinatra with
# ruby examples/sinkiq.rb
# and then browse to http://localhost:4567
#
require 'sinatra'
require 'sidekiq'
require 'redis'

$redis = Redis.new

class SinatraWorker
  include Sidekiq::Worker

  def perform(msg="lulz you forgot a msg!")
    $redis.lpush("sinkiq-example-messages", msg)
  end
end

get '/' do
  stats = Sidekiq::Stats.new
  @failed = stats.failed
  @processed = stats.processed
  @messages = $redis.lrange('sinkiq-example-messages', 0, -1)
  erb :index
end

post '/msg' do
  SinatraWorker.perform_async params[:msg]
  redirect to('/')
end

__END__

@@ layout
<html>
  <head>
    <title>Sinatra + Sidekiq</title>
    <body>
      <%= yield %>
    </body>
</html>

@@ index
  <h1>Sinatra + Sidekiq Example</h1>
  <h2>Failed: <%= @failed %></h2>
  <h2>Processed: <%= @processed %></h2>

  <form method="post" action="/msg">
    <input type="text" name="msg">
    <input type="submit" value="Add Message">
  </form>

  <a href="/">Refresh page</a>

  <h3>Messages</h3>
  <% @messages.each do |msg| %>
    <p><%= msg %></p>
  <% end %>
```

#### 起動コマンド

一応このサンプルの起動コマンドはヘッダに書いてあるとおりこんな感じです。

- sinatra_web: `(bundle exec) ruby ./sidekiq/examples/sinkiq.rb`
- sidekiq_worker: `(bundle exec) sidekiq  -r ./sidekiq/examples/sinkiq.rb -v `

sinatraは`localhost:4567`で起動するので、メッセージをキューしてワーカの出力を見るといいです。

### Sentinelを使う

SidekiqをSentinel経由にする方法は [Advanced Options | sidekiq wiki](https://github.com/mperham/sidekiq/wiki/Advanced-Options#complete-control) にありました。
Redisのインスタンスを自由に設定するやり方を参考に前述のサンプルを変更してみます。

```ruby:sinkiq_sentinel_with_stat.rb 
require 'sinatra'
require 'sidekiq'
require 'redis'
require 'redis-sentinel'

redis_conn = proc {
  Redis.new(:master_name => "example-test",
                  :sentinels => [
                    {:host => "localhost", :port => 26379},
                    {:host => "localhost", :port => 26380}
                  ])
}

Sidekiq.configure_server do |config|
  config.redis = ConnectionPool.new(size: 10, &redis_conn)
end

Sidekiq.configure_client do |config|
  config.redis = ConnectionPool.new(size: 5, &redis_conn)
end

class SinatraWorker
  include Sidekiq::Worker

  def perform(msg="lulz you forgot a msg!")
    $redis.lpush("sinkiq-example-messages", msg)
  end
end

get '/' do
  stats = Sidekiq::Stats.new
  @failed = stats.failed
  @processed = stats.processed
  @messages = $redis.lrange('sinkiq-example-messages', 0, -1)
  erb :index
end

post '/msg' do
  SinatraWorker.perform_async params[:msg]
  redirect to('/')
end

__END__

@@ layout
<html>
<head>
<title>Sinatra + Sidekiq</title>
<body>
<%= yield %>
</body>
</html>

@@ index
<h1>Sinatra + Sidekiq Example</h1>
<h2>Failed: <%= @failed %></h2>
<h2>Processed: <%= @processed %></h2>

<form method="post" action="/msg">
<input type="text" name="msg">
<input type="submit" value="Add Message">
</form>

<a href="/">Refresh page</a>

<h3>Messages</h3>
<% @messages.each do |msg| %>
<p><%= msg %></p>
<% end %>
```

※初版ではここはモンキーパッチでした、ちゃんとスマートに設定できたようです。


### サンプルログ

上記サンプルを使って、sidekiq_workerを `(bundle exec) sidekiq  -r ./sinkiq_sentinel.rb -v `で起動すればStat APIがデータストアにRedis Sentinel環境を使いだします。

ワーカはそれぞれID(@hash)を持っているようなので複数起動しても問題無さ気でした。

マスタ切替時のログはこの様に出力されます。

```shell:sidekiq
TID-oxgwckouo INFO: Running in ruby 2.0.0p247 (2013-06-27 revision 41674) [x86_64-darwin12.3.0]

TID-oxgwcg8yk ERROR: Error fetching message: Error connecting to Redis on 127.0.0.1:16380 (ECONNREFUSED)

# -- 接続切れた

TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/.rvm/gems/ruby-2.0.0-p247@redis-sentinel/gems/redis-3.0.4/lib/redis/client.rb:276:in `rescue in
TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/.rvm/gems/ruby-2.0.0-p247@redis-sentinel/gems/redis-3.0.4/lib/redis/client.rb:271:in `establish
TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/.rvm/gems/ruby-2.0.0-p247@redis-sentinel/gems/redis-3.0.4/lib/redis/client.rb:69:in `connect'
TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/github/others/redis-sentinel/lib/redis-sentinel/client.rb:25:in `block in connect_with_sentinel
TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/github/others/redis-sentinel/lib/redis-sentinel/client.rb:42:in `call'
TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/github/others/redis-sentinel/lib/redis-sentinel/client.rb:42:in `auto_retry_with_timeout'

# -- redis-sentinel gemによるマスタ再発見まわり

TID-oxgwcg8yk ERROR: /Users/sawanoboriyu/github/others/redis-sentinel/lib/redis-sentinel/client.rb:23:in `connect_with_sentinel'
TID-oxgwevm7g ERROR: The master: example-test is currently not available.
TID-oxgwevm7g ERROR: /Users/sawanoboriyu/github/others/redis-sentinel/lib/redis-sentinel/client.rb:75:in `discover_master'

# -- 復旧

TID-oxgwcg8yk INFO: Redis is online, 16.16007900238037 sec downtime
```


良い感じに切り替わりましたね。

