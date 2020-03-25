<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

OpsWorksでCustom Cookbooksを使う際はやや癖があるため、インスタンスの中で色々試したくなりますのでやりましょう。


## カスタムクックブックを更新したい

```shell:Shell
$ cd /opt/aws/opsworks/current/site-cookbooks
$ git pull origin master
```

## ec2-**なコマンドを使いたい

amazon linuxなら`/opt/aws`に`amitools/apitools`入っており、パスが通っているので使いましょう。
デフォルトの権限はレイヤ設定`iam role`に基づきます。


## chef-shellで作業したい

`/opt/aws/opsworks/current`で作業すれば、普通のChefクライアントを操作出来ます。

### 11.4 の場合

```shell:Shell
$ cd /opt/aws/opsworks/current
$ bundle exec chef-shell -s -c conf/solo.rb 
loading configuration: conf/solo.rb
Session type: solo
Loading.......done.

This is the chef-shell.
 Chef Version: 11.4.4
 http://www.opscode.com/chef
 http://wiki.opscode.com/display/chef/Home

run `help' for help, `exit' or ^D to quit.

Ohai2u ec2-user@single1.localdomain!
chef > Chef::VERSION 
 => "11.4.4" 
```

### 11.10 の場合

```shell:Shell
$ cd /opt/aws/opsworks/current
$ /opt/aws/opsworks/current/bin/chef-client -c /var/lib/aws/opsworks/client.rb -v
Chef: 11.10.4

$ sudo -i
# /opt/aws/opsworks/current/bin/chef-shell -j /var/lib/aws/opsworks/chef/2016-08-26-14-10-03-01.json

### ↑jsonのファイル名は動的なので適当に置き換えてください
```

## スタックの情報を確認したい

`opsworks-agent-cli` が便利です。

```shell:Shell
# opsworks-agent-cli get_json
{
  "opsworks_rubygems": {
    "version": "1.8.24"
  },
  "deploy": {
  },
  "ssh_users": {
  },

-- snip --
```

## opsworks-agentを停止したい

monit配下にいるので任意で停止できます。

```shell:Shell
$ monit stop opsworks-agent
$ monit summary
The Monit daemon 5.2.5 uptime: 29m 

Process 'opsworks-agent'            not monitored
Process 'opsworks-agent-master-running' running
File 'opsworks-agent-statistic-daemons-log' accessible
File 'opsworks-agent-process-command-daemons-log' accessible
File 'opsworks-agent-keep-alive-daemons-log' accessible
System 'system_single1.localdomain' running
```

## Ohaiプラグインを追加したい

`/opt/aws/opsworks/current/plugins`に追加するとOKです。

## デプロイされたアプリの場所

`/srv/www/`以下にアプリ名でディレクトリが作られています。

## ログが見たい

`/var/lib/aws/opsworks/chef/`にChefRun時のログが出ています。

## ChefRunに使われたJsonが確認したい

`/var/lib/aws/opsworks/chef/`にChefRun時に使われたJSONが置いてあります、ログと対のタイムスタンプがついています。

jsonがあるのでこの様にリトライしてみることができます。Chef 11.4 なら、

```shell:Shell
$ bundle exec chef-solo -c conf/solo.rb -j /var/lib/aws/opsworks/chef/2013-08-09-04-24-42-01.json
```

11.10 なら、

```shell:Shell
$ /opt/aws/opsworks/current/bin/chef-client -c /var/lib/aws/opsworks/client.rb -j /var/lib/aws/opsworks/chef/2013-08-09-04-24-42-01.json
```


### 再取得したい

`opsworks-agent-cli get_json` で表示できるのでリダイレクトしてファイルにすれば`chef-solo`に渡すJsonとして利用できます。

## ライフサイクルイベントを手動で実行したい

`opsworks-agent-cli run_command` 

configureをやってみます。

```shell:Shell
$ opsworks-agent-cli run_command configure
[2013-08-09 05:13:15] INFO [opsworks-agent (5946)] About to re-run 'configure' from 2013-08-09T04:24:42
Waiting for process 5950
Starting Chef Client, version 11.4.4

-- snip --
```

## 更新したカスタムクックブックのレシピを実行したい

ChefSoloを普通に実行することができます。Chef 11.4 なら、

```shell:Shell
$ bundle exec chef-solo -c conf/solo.rb -o 'my_pkgs::memcached'
Starting Chef Client, version 11.4.4
[2013-08-09T05:15:00+00:00] INFO: *** Chef 11.4.4 ***
[2013-08-09T05:15:03+00:00] DEBUG: Building node object for single1.localdomain
[2013-08-09T05:15:03+00:00] DEBUG: Extracting run list from JSON attributes provided on command line
[2013-08-09T05:15:03+00:00] DEBUG: Applying attributes from json file
[2013-08-09T05:15:03+00:00] DEBUG: Platform is amazon version 2013.03
[2013-08-09T05:15:03+00:00] WARN: Run List override has been provided.
[2013-08-09T05:15:03+00:00] WARN: Original Run List: []
[2013-08-09T05:15:03+00:00] WARN: Overridden Run List: [recipe[my_pkgs::memcached]]
-- snip --
```

11.10 なら、

```shell:Shell
$ /opt/aws/opsworks/current/bin/chef-client -c /var/lib/aws/opsworks/client.rb -o 'my_pkgs::memcached'
```

##  chef-applyしたい

そのままです。

```shell:Shell
# echo "package 'nginx'" | bundle exec chef-apply -s
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * package[nginx] action install
    - install version 1.4.2-1.el6.ngx of package nginx
```

Chefバージョンが`11.4.4`なので、why_runにはコツが要ります。

```shell:Shell
$ echo "Chef::Config[:why_run] = true ; package 'nginx'" | bundle exec chef-apply -s
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * package[nginx] action install
    - Would install version 1.4.2-1.el6.ngx of package nginx
```

----

他にも役に立ちそうなことを思いついたら追記します。
