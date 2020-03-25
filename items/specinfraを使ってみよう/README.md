
specinfraは汎用コマンド実行フレームワーク。RubyGemsとしてで配布されています。

> 追記：これは１の頃の話なので全体的に古いです。
> この書籍が一番詳しい。 => [O'Reilly Japan - Serverspec](http://www.oreilly.co.jp/books/9784873117096/ "O'Reilly Japan - Serverspec")

ソースはこちら [https://github.com/serverspec/specinfra](https://github.com/serverspec/specinfra)

specinfraが目指す所は、情報処理学会研究報告の **serverspec: 宣言的記述でサーバの状態をテスト可能な 汎用性の高いテストフレームワーク** という論文を見ると良いでしょう。

論文もソースコードと同様にGithubに公開されています。

[https://github.com/mizzy/serverspec-thesis](https://github.com/mizzy/serverspec-thesis)


## 概要

同じメソッドで任意のOS用のコマンド実行文字列を取得したり実行して結果をとったりします。

さわった感じこんな挙動

- バックエンドの形式を選ぶ
- バックエンドを元にインスタンスを作成したら、あとは同じメソッドでどこでもOK


## ローカルホストを対象に実行する

チュートリアルはありませんが、serverspecのソースといつもの`spec_helper.rb`を見れば大体つかめます。

```ruby:
require 'specinfra'

include SpecInfra::Helper::DetectOS
include SpecInfra::Helper::Exec

## exec(ローカル実行)をバックエンドにインスタンスを生成する
i = Backend.backend_for('exec')

## こんな感じで共通コマンドが実行できる
i.check_os
```

----

`>> 追記１`

specinfra開発[mizzy/@gosukenator](https://twitter.com/gosukenator)さんによると、単純に使う場合はインスタンス作るまでもなかったようです。


<blockquote class="twitter-tweet" lang="ja"><p><a href="https://twitter.com/sawanoboly">@sawanoboly</a> include SpecInfra::Helper::Exec してる場合は、バックエンドインスタンスを明示的に生成しなくても、<a href="https://t.co/HMFVbb2ehY">https://t.co/HMFVbb2ehY</a> こんな感じでいけますね。（serverspec はこのやりかたです。）</p>&mdash; Gosuke Miyashita (@gosukenator) <a href="https://twitter.com/gosukenator/statuses/443966832882368512">2014, 3月 13</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


[こういう事](https://gist.github.com/mizzy/9521892)、との事。

```ruby:gist.github.com/mizzy/9521892
require 'specinfra'
 
include SpecInfra::Helper::DetectOS
include SpecInfra::Helper::Exec
 
backend.check_os
```



あらほんとだ。

```
> backend.check_os
=> {:family=>"Darwin", :release=>nil}
```

[lib/specinfra/helper/backend.rb](https://github.com/serverspec/specinfra/blob/master/lib/specinfra/helper/backend.rb)の動的なメソッド定義はそういうことだったのね。


`<< 追記1`

----


### 実行の様子

Pryから叩いてみます。

```
> i.check_os
=> {:family=>"Darwin", :release=>nil}
```

メソッドで成否判定をする場合はこんな感じ。

```
> i.check_resolvable('www.google.com', 'dns')
=> true
```

実行されるコマンドを取得する時は`#commands`から同じことをすると良いみたいです。

```
> i.commands.check_resolvable('www.google.com', 'dns')
=> "nslookup -timeout=1 www.google.com"
```

確認しつつ実行とか。

```
> i.commands.check_file('/tmp')
=> "test -f /tmp"

> i.check_file('/tmp')
=> false
```

#### run_command

`#run_command`で任意のコマンドを実行してくる事も可能。
これもバックエンドをちゃんと選択してくれます。


```
> i.run_command('ls')
=> #<SpecInfra::CommandResult:0x007fdd634b3640
 @exit_signal=nil,
 @exit_status=0,
 @stderr="",
 @stdout=
  "Gemfile\nGemfile.lock\nGuardfile\nLICENSE.txt\nREADME.md\nRakefile\nci\nconfig\nenv.sh\nfeatures\njackalope\nlib\npast_features\nrake\nslayer_kitchen\nspec\ntmp\n">
```


メソッドの中には直接実行すると、次のように結果が期待しているのではなかったりします。

※これはUbuntuで実行してます

> 追記: ここのくだりは修正されています。
> 

```
> i.get_package_version('tmux')
=> true
```

いやバージョン欲しいねんけど、って時にも`#run_command`をかますといいのかもしれない？

> 追記: `check_*`以外は`CommandResult`をそのまま返してくれるように変更されたので、`get_package_version`の戻りがこの次のサンプルとおなじになります。

```
> i.commands.get_package_version('tmux')
=> "dpkg-query -f '${Status} ${Version}' -W tmux | sed -n 's/^install ok installed //p'"


> i.run_command(i.commands.get_package_version('tmux'))
=> #<SpecInfra::CommandResult:0x007fa0a2283cd8
 @exit_signal=nil,
 @exit_status=0,
 @stderr="",
 @stdout="1.6-1ubuntu1">
```



### 実行できるメソッド達

[lib/specinfra/command](https://github.com/serverspec/specinfra/tree/master/lib/specinfra/command)を参考にします。

たいていはBaseに定義されているので、ざっと眺めることも出来ます。

```
> SpecInfra::Command::Base.instance_methods(false)
=> [:escape,
 :check_enabled,
 :check_yumrepo,
 :check_yumrepo_enabled,
 :check_mounted,
 :check_routing_table,
 :check_reachable,
 :check_resolvable,
 :check_file,
 :check_socket,
 :check_directory,
 :check_user,
 :check_group,
 :check_installed,
 :check_service_installed,
 :check_service_start_mode,
 :check_listening,
 :check_listening_with_protocol,
 :check_running,
 :check_running_under_supervisor,
 :check_running_under_upstart,
 :check_monitored_by_monit,
 :check_monitored_by_god,
 :check_process,
 :get_process,
 :check_file_contain,
 :check_file_contain_with_regexp,
 :check_file_contain_with_fixed_strings,
 :check_file_checksum,
 :check_file_md5checksum,
 :check_file_sha256checksum,
 :check_file_contain_within,
 :check_mode,
 :check_owner,
 :check_grouped,
 :check_cron_entry,
 :check_link,
 :check_installed_by_gem,
 :check_installed_by_npm,
 :check_installed_by_pecl,
 :check_installed_by_pear,
 :check_installed_by_pip,
 :check_installed_by_cpan,
 :check_belonging_group,
 :check_gid,
 :check_uid,
 :check_login_shell,
 :check_home_directory,
 :check_authorized_key,
 :check_iptables_rule,
 :check_zfs,
 :get_mode,
 :check_ipfilter_rule,
 :check_ipnat_rule,
 :check_svcprop,
 :check_svcprops,
 :check_selinux,
 :check_access_by_user,
 :check_kernel_module_loaded,
 :check_ipv4_address,
 :check_mail_alias,
 :get_file_content,
 :check_container,
 :check_cotainer_running,
 :get_package_version]
```


## SSH越しのホストを対象に実行する

コマンドをSSH越しに実行してくるには、`SpecInfra.configuration.ssh`に`Net::SSH.start`で作成されるインスタンスを入れておけば良いみたいです。



```ruby:
require 'specinfra'
require 'net/ssh'

include SpecInfra::Helper::DetectOS
include SpecInfra::Helper::Ssh

## コンフィグにNet::SSHのインスタンスを設定
# optionsの指定等はserverspecが作るspec_helperを参考にしましょう
SpecInfra.configuration.ssh = Net::SSH.start('example.com', 'root', {})


## sshをバックエンドにインスタンスを生成する
i = Backend.backend_for('ssh')

## こんな感じで共通コマンドが実行できる
i.check_os
```


```
> i.class
=> SpecInfra::Backend::Ssh


> i.check_os
=> {:family=>"Ubuntu", :release=>nil}
```

あとはもはやローカルと同じです。

```
i.commands.check_resolvable('www.google.com', 'dns')
=> "nslookup -timeout=1 www.google.com"

> i.commands.check_resolvable('www.google.com', nil)
=> "getent hosts www.google.com"
```


OSに合わせてメソッドの内容も調整されています。
この様子も[lib/specinfra/command](https://github.com/serverspec/specinfra/tree/master/lib/specinfra/command)で確認できます。

パッケージやサービスの確認をしてみました。

```
> i.commands.check_installed('nagios3')
=> "dpkg-query -f '${Status}' -W nagios3 | grep '^install ok installed$'"

> i.check_installed('nagios3')
=> true


> i.check_running('nagios3')
=> true

> i.commands.check_running('nagios3')
=> "service nagios3 status && service nagios3 status | grep 'running'"
```


## おわりに

今回はRSpecではなく、他のテストフレームワークから似たようなことをするためにspecinfraをさわってみました。
目的がテストだと、結局serverspecみたいになりますね。
