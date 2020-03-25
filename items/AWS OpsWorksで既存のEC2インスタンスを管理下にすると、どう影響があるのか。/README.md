
OpsWorksは専用のインスタンスだけでなく、OpsWorks-Agentを導入することで既存のEC2インスタンスや他の環境にあるサーバを管理することができます。

追加費用がかかるかどうかは、対象の環境で決まります。実はEC2にOpsWorks-Agentを入れる場合は費用がかかりません。

- EC2インスタンス - 無料
- EC2でないサーバ - 有料

## OpsWorks管理下にして、よくなること

- ログインユーザの集中管理ができるようになる
    - loginのみとかsudoあり(Windowsならg:Administrator)とかのパーミッション
        - 権限はスタック単位で変更
    - ユーザはIAMと連携
- 運用タスクをまとめてといろいろ出来るようになる
    - コマンド実行
    - 構成変更(Chefのレシピ)
    - まとめられる単位はスタック全体、レイヤ。単品実行も一応OK。


等々、詳しいことはドキュメントとか、ムック本の『クラウド開発徹底攻略』を参照するとよかったりします。

> infrastructure-classをec2で使用するにはスタックのVPCとEC2のVPCをあわせる必要があります。infrastructure-classをon-premisesで登録する場合はVPCが関係ありませんが、無料かどうかは聞いていません。

## OpsWorks-Agent導入の影響

さて、それなら使ってみようか？となる際に、OpsWorks-Agentのインストールで少々構成が変わるのでまとめてみました。

現行のChef12スタックでは主に次の影響があります。

- mot.dのバナーが変わる
- monitが(コンフィグ上書きで)インストールされる
- `/home/`にあるユーザで、同名は公開鍵が上書きされる
- hostsがスタックのインスタンス書き換わる


で実際どんな感じでしょう？

わりと影響を受けるタイプのAMIとして、[WordPress powered by AMIMOTO (HVM)](https://aws.amazon.com/marketplace/pp/B00LWHVJH8)を例に確認してみます。

OpsWorks-Agentの導入は、ウィザードにそえばOKです。

![Register_Instances_-_oldman_–_AWS_OpsWorks.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/1fa45cea-d048-5eb2-f5f3-b1d8f3de9fde.jpeg "Register_Instances_-_oldman_–_AWS_OpsWorks.jpg")


なおaws-cliやChefも入りますが、/opt以下に閉じ込める形で同梱されています。大抵の環境では影響ないと思います。

### mot.dのバナーが変わる

OpsWorks-Agentはバナーを書き換えます。

導入前がこんなだったとしても、

```
Last login: Wed Apr  6 16:50:19 2016 from zaq3a55b3c8.zaq.ne.jp
      ___         _            __
     / _ | __ _  (_)_ _  ___  / /____
    / __ |/  ' \/ /  ' \/ _ \/ __/ _ \
   /_/ |_/_/_/_/_/_/_/_/\___/\__/\___/

https://aws.amazon.com/amazon-linux-ami/2016.03-release-notes/

 Nginx 1.9.15, PHP 5.6.21, Percona MySQL 5.6.29, WP-CLI 0.23.1

 amimoto     http://www.amimoto-ami.com/
 digitalcube https://en.digitalcube.jp/

12 package(s) needed for security, out of 36 available
Run "sudo yum update" to apply all updates.
```

導入後はこんな風に。そっけない。。
↓↓↓↓

```
Last login: Mon May 23 13:27:06 2016 from kd111107182054.au-net.ne.jp
 This instance is managed with AWS OpsWorks.

   ######  OpsWorks Summary  ######
   Operating System: amazon 2016.03
   OpsWorks Instance: ip-10-0-1-233
   OpsWorks Instance ID: 90ce398d-ab61-454e-b601-dfe772133ddf
   OpsWorks Layers: 
   OpsWorks Stack: old-instances
   EC2 Region: ap-northeast-1
   EC2 Availability Zone: ap-northeast-1c
   EC2 Instance ID: i-8af7ac05
   Public IP: 52.196.235.47
   Private IP: 10.0.1.233
   VPC ID: vpc-xxxxxxx
   Subnet ID: subnet-xxxxxxx

 Visit http://aws.amazon.com/opsworks for more information.
```

まあこれはよしとしましょう。

### monitが(コンフィグ上書きで)インストールされる

環境によってはこれが一番影響が大きいかな。monitを既に利用している環境だと、事前にいくらか調整をしておくのがよいです。


```
$ sudo monit summary
monit: Cannot translate 'ip-10-0-1-233' to FQDN name -- Name or service not known
The Monit daemon 5.2.5 uptime: 20m 

Process 'php-fpm'                   running
Directory 'php_tmp'                 accessible
Directory 'php_session'             accessible
Directory 'php_log'                 accessible
Process 'nginx'                     running
Directory 'nginx_cache'             accessible
Directory 'nginx_lib'               accessible
Directory 'nginx_lib_tmp'           accessible
Directory 'nginx_log'               accessible
Process 'mysql'                     running
Process 'memcached'                 running
Process 'crond'                     running
System 'system_ip-10-0-1-233'       running
```

OpsWorks-Agentインストール前のmonit.conf

```/etc/monit.conf 
## This file modified by chef for 

set daemon 30
  with start delay 5

set httpd port 2812 and
  use address localhost
  allow localhost


include /etc/monit.d/*
```

OpsWorks-Agentの導入後にmonit summaryを見ると、なんか少なくなってます。

```
$ sudo monit summary
The Monit daemon 5.2.5 uptime: 22m 

Process 'opsworks-agent'            running
File 'opsworks-agent-statistic-daemons-log' accessible
File 'opsworks-agent-keep-alive-daemons-log' accessible
System 'system_ip-10-0-1-233'       running
```




```
/etc/monit.conf 
set daemon 60
set mailserver localhost
set eventqueue
    basedir /var/monit
    slots 100
set logfile syslog
Include /etc/monit.d/*.monitrc
set httpd port 2812 and use the address localhost
  allow localhost
```


monitを既に導入している場合、include用のファイル名を`monit.d/*.monitrc`に適合するように変更すれば元々あったコンフィグをまた使うことができます。

とりあえず`/etc/monit.d/`以下はこんな状況になっていたりするので、

```
$ sudo find /etc/monit.d/ -type f
/etc/monit.d/
/etc/monit.d/opsworks-agent.monitrc
/etc/monit.d/memcached
/etc/monit.d/nginx
/etc/monit.d/crond
/etc/monit.d/mysql
/etc/monit.d/php-fpm
/etc/monit.d/logging
```

たとえばfindでごっそり変更してmonitをリロードします。

```
$ sudo find /etc/monit.d -name '*monitrc' -prune -o -type f -exec mv {}{,.monitrc} \;
$ find /etc/monit.d/
/etc/monit.d/
/etc/monit.d/crond.monitrc
/etc/monit.d/nginx.monitrc
/etc/monit.d/logging.monitrc
/etc/monit.d/memcached.monitrc
/etc/monit.d/php-fpm.monitrc
/etc/monit.d/opsworks-agent.monitrc
/etc/monit.d/mysql.monitrc
```

そしたら元からあったモニタリングも復活しますね。

```
$ sudo monit summary
The Monit daemon 5.2.5 uptime: 0m 

Process 'php-fpm'                   running
Directory 'php_tmp'                 accessible
Directory 'php_session'             accessible
Directory 'php_log'                 accessible
Process 'opsworks-agent'            running
File 'opsworks-agent-statistic-daemons-log' accessible
File 'opsworks-agent-keep-alive-daemons-log' accessible
Process 'nginx'                     running
Directory 'nginx_cache'             accessible
Directory 'nginx_lib'               accessible
Directory 'nginx_lib_tmp'           accessible
Directory 'nginx_log'               accessible
Process 'mysql'                     running
Process 'memcached'                 running
Process 'crond'                     running
System 'system_ip-10-0-1-233.localdomain' running
```

`M/Monit`等導入済みで、`monit.conf`がもうちょい複雑ー、というケースもありそうですが、そういう所は自分でなんとかできるでしょう。


### `/home/`にあるユーザで、ローカルとIAMが同名の場合公開鍵が上書きされる

これも段取り次第では戸惑うかもしれません。`ec2-user`は勝手に消えたりしないので大事にはならないと思います。

```
$ ls /home/
ec2-user
```

とりあえず上書きされる予定のユーザを作って、適当な公開鍵をおいておきます。

```
$ sudo useradd -m sawanoboly
[root@ip-10-0-1-233 ec2-user]# ls /home/
ec2-user  sawanoboly


$ sudo su sawanoboly -
$ cd
$ sh-keygen -t rsa -b 2048 
...

$ cat .ssh/id_rsa.pub | tee .ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC21yE8iItAbdZM9qIcirNkky5AuM6XbBsWZgkHlQ1khtNv8/C2ClYgUicCa0wfRBvq+HEdA9OLNx6kUAP9/Mu6m0+GYKP4u0SLz7+CYLch/R9A8wWJ/U3x5vC2zADvRM8SMcgxCU8Ne0ConAA/NeN8bgOs+xo5Jbv/BJ+DMlxeZhcOUJTST5df15ZdpseXjP3+ZWNnVYJfTUUec/Qb8UMjy83/6prL4x31eWKb7VvAeqSK69mJ3fHKeYTO6vs6Q2pJu3PYSDfXDxShMaqmeOaNaeH936UTuCm8RLb4KWTWWdtZMCfVyWct1gailCazi4eaOwLLz8MWysdo6YG4dQ+N sawanoboly@ip-10-0-1-233
```

で、OpsWorksのPermission画面でユーザ設定をちょこちょことつけてみます。

![Permissions_-_old-instances_–_AWS_OpsWorks.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/920e7235-13ca-006b-3cde-b8b1b2c390d5.jpeg "Permissions_-_old-instances_–_AWS_OpsWorks.jpg")


OpsWorks-AgentによりChefが実行されるので、すぐユーザが増えてます。

```
$ sudo ls /home/
ec2-user  sawanoboly  webnoboly
```

既存ユーザの公開鍵がOpsWorks登録のものに書き換わりました。

```
$ sudo cat /home/sawanoboly/.ssh/authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDK8O+7ANkBxa42yCVjd3xxofOhzUO1ThMDkdgnuw5AS.......
```

Permision操作の挙動は既存の`.ssh`ディレクトリの状況によって色々変わったりもするので、害のない範囲で試してからをおすすめします。


## おわりに

もし個別にSSHログインして、それぞれ同じ作業とかをするようなEC2インスタンスが残っているならば、OpsWorks-Agentを突っ込んで、まとめて適用できるようにするのもよいかもしれません。

