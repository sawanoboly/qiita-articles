
AWSのElastic BeanstalkにRailsプロジェクトをおいていて、非同期のジョブを[Sidekiq](http://sidekiq.org)にやらせている。
で、原因は色々とあるようだがたまにsidekiqプロセスが居なくなることがあった。

取り急ぎ、止まっていたら起動するように2つの方針を考えた。

- monitでモニタリング&restart
- Upstartでrespawn
    - inspeqtorでモニタリング

EBのRuby(puma)コンテナではRails用のpumaがUpstartで動いている。
monit(+M/Monit)の柔軟設定とゆるさは鉄板だが(飽きてきたので)、ここはsidekiqの作者[Mike Perham](https://github.com/mperham)さんのプロダクトであるinspeqtorと組み合わせてみよう。パーハム祭りである。

参考： [Inspeqtor - application infrastructure monitoring](http://contribsys.com/inspeqtor/ "Inspeqtor - application infrastructure monitoring")


## SidekiqをUpstart配下にする

まずは 『Sidekiq x Upstart』。
設置したファイルは3つです。Upstart用のファイルはほぼEBのpumaから流用。

- `/opt/elasticbeanstalk/hooks/appdeploy/post/50_restart_sidekiq`
- `/opt/elasticbeanstalk/hooks/appdeploy/pre/03_mute_sidekiq`
- `/opt/elasticbeanstalk/support/conf/sidekiq.conf` (`/etc/init/sidekiq.conf`にシンボリックリンク作成)


まず必要なファイルを設置。
pumaのコンフィグで特に参考になったのは`exec /bin/bash <<"EOT"`。その手があったかー。
Sidekiqの起動オプションはお好みで。muteは一応。

```yaml:15_sidekiq_initial.config
---
files:
  "/opt/elasticbeanstalk/hooks/appdeploy/post/50_restart_sidekiq":
    mode: "000755"
    content: |
      #!/bin/bash
      initctl restart sidekiq || initctl start sidekiq

      ln -sf /var/app/current/log/sidekiq.log /var/app/containerfiles/logs/sidekiq.log

  "/opt/elasticbeanstalk/hooks/appdeploy/pre/03_mute_sidekiq":
    mode: "000755"
    content: |
      #!/bin/bash

      . /opt/elasticbeanstalk/support/envvars

      PIDFILE=/var/app/containerfiles/pids/sidekiq.pid
      if [ -f ${PIDFILE} ]; then
        if [ -d /proc/`cat ${PIDFILE}` ]; then
          kill -USR1 `cat ${PIDFILE}`
        fi
      fi

  "/opt/elasticbeanstalk/support/conf/sidekiq.conf":
    mode: "000644"
    content: |
      description "Elastic Beanstalk Sidekiq Upstart Manager"

      start on runlevel [2345]
      stop on runlevel [!2345]

      # explained above
      respawn
      respawn limit 3 30

      script
      # scripts run in /bin/sh by default
      # respawn as bash so we can source in rbenv
      exec /bin/bash <<"EOT"
        EB_SCRIPT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k script_dir)
        EB_SUPPORT_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k support_dir)

        . $EB_SUPPORT_DIR/envvars
        . $EB_SCRIPT_DIR/use-app-ruby.sh

        EB_APP_DEPLOY_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_deploy_dir)
        EB_APP_PID_DIR=$(/opt/elasticbeanstalk/bin/get-config container -k app_pid_dir)
        cd $EB_APP_DEPLOY_DIR

        exec su -s /bin/bash -c "sidekiq -e ${RACK_ENV} -c ${SIDEKIQ_WORKERS:-2} -L ${EB_APP_DEPLOY_DIR}/log/sidekiq.log -C ${EB_APP_DEPLOY_DIR}/config/sidekiq.yml -P ${EB_APP_PID_DIR}/sidekiq.pid" webapp
      EOT
      end script
```

pumaの真似をしてリンク作成、initcrlをリロード。

```yaml:16_sidekiq_setup.config
---
files:
  "/etc/init/sidekiq.conf" :
    mode: "120400"
    content: "/opt/elasticbeanstalk/support/conf/sidekiq.conf"

commands:
  reload_initctl_for_sidekiq:
    command: "initctl reload-configuration"
```

これでeb deployすればsidekiqがUpstart配下で起動する。

### SidekiqがUpstart配下にあることを確認する

一応見に行こう。

```
$ sudo initctl status sidekiq
sidekiq start/running, process 22745
```

killしてもリスポーンすることもついでに確認しておくとよいかも。


## inspeqtorをElsticBeanstalkのRubyコンテナに導入する

リポジトリを登録して、yumからインストール。
ここは別ファイルにしないとpackagesが先に実行されるみたい。

```yaml:10_inspeqtor_repo.config
---
commands:
  01_download_inspeqtor_repo_register:
    command: "/usr/bin/curl -L https://bit.ly/InspeqtorRPM -o /tmp/InspeqtorRPM"
  02_setup_inspeqtor_repo:
    command: "/bin/bash /tmp/InspeqtorRPM"
```

```yaml:11_inspeqtor_install.config
---
packages:
  yum:
    nc: []
    inspeqtor: []
```

このrpmはインストール後に自動でinspeqtorが起動する。操作はinitctlで。


## SidekiqをInspectorでモニタリングする

次は『sidekiq x inspector』。
ファイルを置いてinspeqtorをリスタートしよう。

```21_monitor_sidekiq_by_inspeqtor.config 
---
files:
  "/etc/inspeqtor/services.d/sidekiq.inq":
    content: |
      check service sidekiq
        if memory:rss > 500m then restart
        if cpu:user > 60% for 2 cycles then restart

commands:
  reload_inspeqtor:
    command: "initctl restart inspeqtor || initctl start inspeqtor"
```

メール通知の行き先は適当に。自分はこういうのはPagerdutyに送る。


### InspectorがSidekiqの状態を見ていることを確認。

`inspeqtorctl`で状況を確認できる。

```
$ sudo inspeqtorctl status
Inspeqtor 0.8.1, uptime: 47m47.786741131s, pid: 3685

Host: ip-xx-xx-xx-xx
  cpu                                 0.8%           
  cpu:iowait                          0.1%           
  cpu:steal                           0.1%           
  cpu:system                          0.2%           
  cpu:user                            0.5%           
  disk:/                              50.0%           90%
  load:1                              0.00           
  load:15                             0.05           
  load:5                              0.01            10
  swap                                1.0%            20%

Service: sidekiq [Up/22745]
  cpu:system                          0.0%           
  cpu:total_system                    0.0%           
  cpu:total_user                      0.0%           
  cpu:user                            0.0%            60%
  memory:rss                          1.42m           500m
```

よし見てるね。これで仕様は次の通り。

- 止まってたらリスタート
- よーわからん高負荷だったらリスタート(アラート)


## ほか

`Sidekiq::Stats`なデータはカスタムメトリクスでClowdWatchに送るなりもOK。
ついでにNew RericとAirBrakeも使っている。

Sidekiq用のRedisはアプリ間で共有が必須かどうかなどの要件に応じて、[Redis To Go](http://redistogo.com/ "Redis To Go")、ローカルRedis(epelからさくっと入れることができる)、ElastiCache から選ぶといい。
