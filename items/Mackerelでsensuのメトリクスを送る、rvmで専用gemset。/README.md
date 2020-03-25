<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[@matsumotory](http://blog.matsumoto-r.jp/)がMackerelを申し込み、inviteされたのでエージェントを入れてみた。

[【ベータテスター募集中】Mackerel: 新しいアプリケーションパフォーマンスマネジメント](https://mackerel.io/ "【ベータテスター募集中】Mackerel: 新しいアプリケーションパフォーマンスマネジメント")

Mackerel-Agentの設定ファイルを見るとsensu形式の出力も送信できるとなっていたのでやってみた。

```
-- snip --
# Configuration for Sensu Plugins
# Sensu checks (ref. http://sensuapp.org/docs/0.12/adding_a_check)
# Currently, metric type command can be used
# [sensu.checks.vmstat]
# command = "ruby /etc/sensu/plugins/system/vmstat-metrics.rb"
-- snip --
```

## RVMでRubyを入れとく

sensuのプラグインは`require 'sensu-plugin/metric/cli'`します。

対象サーバにはroot用にRubyを入れていなかったので、RVMを入れておくことにした。

[RVM: Ruby Version Manager - RVM Ruby Version Manager - Documentation](https://rvm.io/ "RVM: Ruby Version Manager - RVM Ruby Version Manager - Documentation")

```
~# rvm install 2.1.2
Searching for binary rubies, this might take some time.

-- snip --

Install of ruby-2.1.2 - #complete 
Ruby was built without documentation, to build it run: rvm docs generate-ri
```

### Gemsetで専用環境を用意する

グローバルにgemを入れるのはあまりやらないので、sensu専用のgemsetを作ることにした。

```
# rvm gemset create sensu
ruby-2.1.2 - #gemset created /home/ubuntu/.rvm/gems/ruby-2.1.2@sensu
ruby-2.1.2 - #generating sensu wrappers.........

# rvm use 2.1.2@sensu
Using /home/ubuntu/.rvm/gems/ruby-2.1.2 with gemset sensu

# gem install sensu-plugin --no-ri --no-rdoc 
Fetching: mixlib-cli-1.5.0.gem (100%)
Successfully installed mixlib-cli-1.5.0
Fetching: sensu-plugin-0.3.0.gem (100%)
Successfully installed sensu-plugin-0.3.0
2 gems installed

# gem which sensu-plugin
/home/ubuntu/.rvm/gems/ruby-2.1.2@sensu/gems/sensu-plugin-0.3.0/lib/sensu-plugin.rb
```

これで`ruby-2.1.2@sensu`の環境で`sensu-plugin`がロードできる。


### rvm-execで指定gemsetでスクリプト実行

Mackerel-Agentのサンプルでは`"ruby /etc/sensu/plugins/system/vmstat-metrics.rb"`の用になっていたので、この形式で任意のgemsetを使う場合`rvm-exec`が楽だ。

```
# /usr/local/rvm/bin/rvm-exec 2.1.2@sensu  /home/ubuntu/github/sensu-community-plugins/plugins/system/vmstat-metrics.rb 
higanworks.example.com.vmstat.procs.waiting	0	1400045692
higanworks.example.com.vmstat.procs.uninterruptible	0	1400045692
higanworks.example.com.vmstat.memory.swap_used	36788	1400045692
higanworks.example.com.vmstat.memory.free	125280	1400045692
higanworks.example.com.vmstat.memory.buffers	24232	1400045692
higanworks.example.com.vmstat.memory.cache	1018052	1400045692
higanworks.example.com.vmstat.swap.in	0	1400045692
higanworks.example.com.vmstat.swap.out	0	1400045692
higanworks.example.com.vmstat.io.received	0	1400045692
higanworks.example.com.vmstat.io.sent	0	1400045692
higanworks.example.com.vmstat.system.interrupts_per_second	50	1400045692
higanworks.example.com.vmstat.system.context_switches_per_second	107	1400045692
higanworks.example.com.vmstat.cpu.user	0	1400045692
higanworks.example.com.vmstat.cpu.system	0	1400045692
higanworks.example.com.vmstat.cpu.idle	100	1400045692
higanworks.example.com.vmstat.cpu.waiting	0	1400045692
```

出力できているようだね。

## mackerel-agentの設定に反映する

ということで、設定は次のようになった。

```mackerel-agent.conf
-- snip --
# Currently, metric type command can be used
[sensu.checks.vmstat]
# command = "ruby /etc/sensu/plugins/system/vmstat-metrics.rb"
command = "/usr/local/rvm/bin/rvm-exec 2.1.2@sensu  /home/ubuntu/github/sensu-community-plugins/plugins/system/vmstat-metrics.rb"
type = "metric"
-- snip --
```

Mackerel側にもメトリクスがあがりました。vmstatではデフォルトの内容と大体かぶっている事は一旦置いときましょう。

![スクリーンショット_5_14_14__3_35_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/df4d4a37-f304-a800-5853-3dc6d3e3d0f2.png "スクリーンショット_5_14_14__3_35_PM.png")

