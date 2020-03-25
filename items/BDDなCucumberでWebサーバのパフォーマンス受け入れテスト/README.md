<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[Cucumber](http://cukes.info/)は、"このようになっていてくれ"という内容を記述して、それを実際にやってみた結果を判断するツール。

## featureとしてシナリオを記述

```cucumber:features/websv.feature
# language: ja
機能: 静的コンテンツ配信
シナリオ: 秒辺りのレスポンスが規定以上かチェック
  前提: ウェブサーバアドレスが 127.0.0.1 のとき
  もし: 10 台のクライアントが同時に 1000 アクセスした
  ならば: サーバのレスポンスが 3000 rps以上で
  かつ: レスポンスコード 5xx がない
```

とりあえずシナリオを作成し、cucumberを実行する。


```
# cucumber 
Using the default profile...
# language: ja
機能: 静的コンテンツ配信

  シナリオ: 秒辺りのレスポンスが規定以上かチェック         # features/websv.feature:3
    前提 ウェブサーバアドレスが 127.0.0.1 のとき   # features/websv.feature:4
    もし 10 台のクライアントが同時に 1000 アクセスした # features/websv.feature:5
    ならば サーバのレスポンスが 3000 rps以上で     # features/websv.feature:6
    かつ レスポンスコード 5xx がない            # features/websv.feature:7

1 scenario (1 undefined)
4 steps (4 undefined)
0m0.002s

You can implement step definitions for undefined steps with these snippets:

前提 /^ウェブサーバアドレスが (\d+)\.(\d+)\.(\d+)\.(\d+) のとき$/ do |arg1, arg2, arg3, arg4|
  pending # express the regexp above with the code you wish you had
end

もし /^(\d+) 台のクライアントが同時に (\d+) アクセスした$/ do |arg1, arg2|
  pending # express the regexp above with the code you wish you had
end

ならば /^サーバのレスポンスが (\d+) rps以上で$/ do |arg1|
  pending # express the regexp above with the code you wish you had
end

ならば /^レスポンスコード (\d+)xx がない$/ do |arg1|
  pending # express the regexp above with the code you wish you had
end

If you want snippets in a different programming language,
just make sure a file with the appropriate file extension
exists where cucumber looks for step definitions.
```


**これらを定義しなさい**と促されるのでstepsを作成する。


### steps作成


```ruby:features/websv.steps.rb 
# coding: utf-8
前提 /^: ウェブサーバアドレスが ([\w\.-]+) のとき$/ do |arg1|
  pending # express the regexp above with the code you wish you had
end

もし /^: (\d+) 台のクライアントが同時に (\d+) アクセスした$/ do |arg1, arg2|
  pending # express the regexp above with the code you wish you had
end

ならば /^: サーバのレスポンスが (\d+) rps以上で$/ do |arg1|
  pending # express the regexp above with the code you wish you had
end

ならば /^: レスポンスコード (\d+)xx がない$/ do |arg1|
  pending # express the regexp above with the code you wish you had
end
```


"ウェブサーバアドレス" の所だけパラメータ抽出表現を変えて作成。
これで作成してみる。

```
# cucumber 
Using the default profile...
# language: ja
機能: 静的コンテンツ配信

  シナリオ: 秒辺りのレスポンスが規定以上かチェック         # features/websv.feature:3
    前提: ウェブサーバアドレスが 127.0.0.1 のとき   # features/websv.steps.rb:2
      TODO (Cucumber::Pending)
      ./features/websv.steps.rb:3:in `/^: ウェブサーバアドレスが ([\w\.-]+) のとき$/'
      features/websv.feature:4:in `前提: ウェブサーバアドレスが 127.0.0.1 のとき'
    もし: 10 台のクライアントが同時に 1000 アクセスした # features/websv.steps.rb:6
    ならば: サーバのレスポンスが 3000 rps以上で     # features/websv.steps.rb:10
    かつ: レスポンスコード 5xx がない            # features/websv.steps.rb:14

1 scenario (1 pending)
4 steps (3 skipped, 1 pending)
0m0.003s
```


テスト内容がpendingとなる。

### pendingを実装していく

ベンチマークにhttperfを使うので、rubyバインディング[httperfrb](https://github.com/rubyops/httperfrb)を準備してスタート。


#### ウェブサーバアドレスが ([\w\.-]+) のとき


```ruby:
# coding: utf-8
require 'open3'
require 'httperf'

前提 /^ウェブサーバアドレスが ([\w\.-]+) のとき$/ do |arg1|
   @perf = HTTPerf.new( "server" => arg1, "uri" => "/")     
end  
```


#### (\d+) 台のクライアントが同時に (\d+) アクセスした

```ruby:
もし /^(\d+) 台のクライアントが同時に (\d+) アクセスした$/ do |arg1, arg2|
  @perf.update_option("num-conns", arg1)
  @perf.update_option("num-calls", arg2)
end
```

#### サーバのレスポンスが (\d+) rps以上で

```ruby:
ならば /^サーバのレスポンスが (\d+) rps以上で$/ do |arg1|
  @results = HTTPerf::Parser.parse(@perf.run)
  puts "INFO: Request_rate is " + @results[:request_rate_per_sec]
  raise "server is too slow." if @results[:request_rate_per_sec].to_i < arg1.to_i
end
```

### レスポンスコード (\d+)xx がない

```ruby:
ならば /^レスポンスコード (\d+)xx がない$/ do |arg1|
  raise "Response code 5xx found!!" unless @results[:reply_status_5xx].to_i == 0
end
```


こんな感じで`features/websv.steps.rb`にて実装する。

## cucumber実行結果



```
# cucumber 
Using the default profile...
# language: ja
機能: 静的コンテンツ配信

  シナリオ: 秒辺りのレスポンスが規定以上かチェック       # features/websv.feature:3
    前提ウェブサーバアドレスが 127.0.0.1 のとき   # features/websv.steps.rb:5
    もし10 台のクライアントが同時に 1000 アクセスした # features/websv.steps.rb:9
    ならばサーバのレスポンスが 3000 rps以上で     # features/websv.steps.rb:14
      INFO: Request_rate is 16281.4
    かつレスポンスコード 5xx がない            # features/websv.steps.rb:20

1 scenario (1 passed)
4 steps (4 passed)
0m0.073s
```

想定仕様は3000rpsで、実際のレスポンスは16000rps。
非機能要件をクリアしていると言える。


## 付録：features/websv.steps.rb の全行

```
# coding: utf-8
require 'open3'
require 'httperf'

前提 /^ウェブサーバアドレスが ([\w\.-]+) のとき$/ do |arg1|
   @perf = HTTPerf.new( "server" => arg1, "uri" => "/")
end

もし /^(\d+) 台のクライアントが同時に (\d+) アクセスした$/ do |arg1, arg2|
  @perf.update_option("num-conns", arg1)
  @perf.update_option("num-calls", arg2)
end

ならば /^サーバのレスポンスが (\d+) rps以上で$/ do |arg1|
  @results = HTTPerf::Parser.parse(@perf.run)
  puts "INFO: Request_rate is " + @results[:request_rate_per_sec]
  raise "server is too slow." if @results[:request_rate_per_sec].to_i < arg1.to_i
end

ならば /^レスポンスコード (\d+)xx がない$/ do |arg1|
  raise "Response code 5xx found!!" unless @results[:reply_status_5xx].to_i == 0
end
```
