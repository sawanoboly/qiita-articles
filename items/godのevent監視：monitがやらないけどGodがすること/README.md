<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


プロセスの死活監視は多くがpollingベースで動作する、monitのように○分おきにチェックとか。

rubygemsのプロセス監視デーモン、[god](http://godrb.com/)は対象プロセスの状態変化をイベントシステムで追うやり方を提供している。

## godがイベントシステム対応でインストールされているか確認する

```
$ god check
using event system: netlink
-- snip --
```

`event system`が`none`でなければ使える。


## サンプルの監視対象プロセスを用意する

単純に終わらないスクリプトを用意。

```ruby:sample.rb 
#!/usr/bin/env ruby
loop do
  puts 'Hello'
  sleep 10
end
```


## godのコンフィグ

監視対象を起動しつつ、`process_exits`を検知したらstartを実行してup状態を保てという設定。

```ruby:god.conf 
God.watch do |w|
  w.name = "sample"
  w.start = "/root/sandbox/god/sample.rb"
  w.keepalive

  # 状態監視のジョブ
  w.transition(:up, :start) do |on|
    on.condition(:process_exits)
  end
end
```



## godが検知する様子

眺めるためにフォアグラウンドで起動。

```
# god -D -c god.conf
I [2012-08-09 20:03:36]  INFO: Loading god.conf
I [2012-08-09 20:03:36]  INFO: Syslog enabled.
I [2012-08-09 20:03:36]  INFO: Using pid file directory: /var/run/god
I [2012-08-09 20:03:36]  INFO: Started on drbunix:///tmp/god.17165.sock
I [2012-08-09 20:03:36]  INFO: sample move 'unmonitored' to 'init'
I [2012-08-09 20:03:36]  INFO: sample moved 'unmonitored' to 'init'
I [2012-08-09 20:03:36]  INFO: sample [trigger] process is running (ProcessRunning)
I [2012-08-09 20:03:36]  INFO: sample move 'init' to 'up'
I [2012-08-09 20:03:36]  INFO: sample registered 'proc_exit' event for pid 21539
I [2012-08-09 20:03:36]  INFO: sample registered 'proc_exit' event for pid 21539
I [2012-08-09 20:03:36]  INFO: sample moved 'init' to 'up'
```

先ほどのプロセスをkill。

```
# date && kill 21539
Wed Aug  9 20:05:21 PDT 2012
```

godはprocessのexitを検知して、コンディションをstartに変更する。

```
[2012-08-09 20:05:21]  INFO: sample [trigger] process 21539 exited {:pid=>21539, :exit_code=>15, :exit_signal=>17, :thread_group_id=>21539} (ProcessExits)
I [2012-08-09 20:05:21]  INFO: sample move 'up' to 'start'
I [2012-08-09 20:05:21]  INFO: sample deregistered 'proc_exit' event for pid 21539
I [2012-08-09 20:05:21]  INFO: sample deregistered 'proc_exit' event for pid 21539
I [2012-08-09 20:05:21]  INFO: sample start: /root/sandbox/god/sample.rb
I [2012-08-09 20:05:21]  INFO: sample moved 'up' to 'start'
I [2012-08-09 20:05:21]  INFO: sample [trigger] process is running (ProcessRunning)
I [2012-08-09 20:05:21]  INFO: sample move 'start' to 'up'
I [2012-08-09 20:05:21]  INFO: sample registered 'proc_exit' event for pid 22979
I [2012-08-09 20:05:21]  INFO: sample registered 'proc_exit' event for pid 22979
I [2012-08-09 20:05:21]  INFO: sample moved 'start' to 'up'
```


プロセスの終了から再開までは一秒未満、pollingではインターバル分のリードタイムがあるので、状況によって使い分けができる。

