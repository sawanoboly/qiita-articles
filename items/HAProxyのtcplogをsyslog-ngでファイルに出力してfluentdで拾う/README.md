<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


HAProxyの出力はsyslogなので、syslog-ngで単一のファイルにまとめる。
ついでにfluentdにも送るのでばらしてRubyのMatchDataにしよう。


## syslog-ngの設定
設定を作ってインポートしよう、fluentdとかへの使い回しを考えなければファイル名に日付マクロを使ってもいい。

```syslog-ng_haproxy.conf
template t_haproxy {
  template("$MSGHDR$PRIORITY $MSG\n");
  template_escape(no);
};

destination d_haproxy {
  file("/var/log/haproxy/haproxy.log",
  create_dirs(yes),
  template(t_haproxy));
};

filter f_haproxy {
  program(haproxy);
};

log {
  source(s_src);
  filter(f_haproxy);
  destination(d_haproxy);
};
```

haproxy側ではsyslog-ngのs_srcにログが届くようにしておく、ローカルなら/dev/log宛など。

## ログの出力例

HAProxyのドキュメントにはPriorityが無いが、わかりやすいので出しておく。

    haproxy[27508]: info 127.0.0.1:45111 [12/Jul/2012:15:19:03.258] wss-relay wss-relay/local02_9876 0/0/50015 1277 cD 1/0/0/0/0 0/0


### fluentdのためにMatchDataへ

fluentdに送るだけなら実はテンプレート変更をせずにsyslog形式で送ればよいのだが、HAProxyのtcplogフォーマットが$MSGで一緒くたにされる。
公式ドキュメントを参考にRubyのMatchDataにバラすテスト。

```ruby:irb(pry)
> m = "haproxy[27508]: info 127.0.0.1:45111 [12/Jul/2012:15:19:03.258] wss-relay wss-relay/local02_9876 0/0/50015 1277 cD 1/0/0/0/0 0/0"


> m = txt.match(/(?<ps>\w+)\[(?<pid>\d+)\]: (?<pri>\w+) (?<c_ip>[\w\.]+):(?<c_port>\d+) \[(?<a_date>.+)\] (?<f_end>[\w-]+) (?<b_end>[\w-]+)\/(?<b_server>[\w-]+) (?<tw>\d+)\/(?<tc>\d+)\/(?<tt>\d+) (?<bytes>\d+) (?<t_state>[\w-]+) (?<actconn>\d+)\/(?<feconn>\d+)\/(?<beconn>\d+)\/(?<srv_conn>\d+)\/(?<retries>\d+) (?<srv_queue>\d+)\/(?<backend_queue>\d+)/)
> m.names.each {|x| puts "#{x} => #{m[x]}" }
ps => haproxy
pid => 27508
pri => info
c_ip => 127.0.0.1
c_port => 45111
a_date => 12/Jul/2012:15:19:03.258
f_end => wss-relay
b_end => wss-relay
b_server => local02_9876
tw => 0
tc => 0
tt => 50015
bytes => 1277
t_state => cD
actconn => 1
feconn => 0
beconn => 0
srv_conn => 0
retries => 0
srv_queue => 0
backend_queue => 0
```

後の解析のためといえ数が多いと大変ね。

時間が変な形式なのでRubyでTimeに変換するならstrptimeで。

    puts DateTime.strptime(m["a_date"],"%d/%B/%Y:%H:%M:%S")  #=> 2012-07-12T15:19:03+00:00

よってfluentdのtime_formatはこうなる。

    time_format "%d/%B/%Y:%H:%M:%S"
