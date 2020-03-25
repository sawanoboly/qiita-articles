<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

node.jsで作ったアプリの運用を始めるので管理の方法を検討した。

## 方針
node.jsはnave.shでバージョン管理する。PATHさえ通せば使えるのでusemainもしない。多分nodebrewもPATHだけなので同様の方法が使える模様。

node.jsは単体ではでdaemonizeしない、依存増えそうなので。
プロセスの監視はmonit。手法としてはUpstartでのrespawnがよく紹介されているが、node.jsはゆるりと起動してくればよいのでmonit。


## 起動＆停止スクリプト
任意バージョンのnode.jsで /opt/myapp にあるプログラムを操作するスクリプト。

### 起動

```start.sh
export PATH="home/node/.nave/installed/0.8.2/bin:$PATH"
export NODE_ENV=production
start-stop-daemon -S \
 -p /var/run/myapp.pid \
 -d /opt/myapp \
 -c node
 -m -b \
 -x/home/node/.nave/installed/0.8.2/bin/node -- ./app.js --applabel myapp
```

ちなみに最後のapplabelは複数のnodejsアプリが動作するときにpsで判別しやすくする目的でつけているので単体で動かすなら必要ない。

ログなどを標準出力に出している場合は最後にリダイレクトで適当なファイルに出力すればOK。その場合のログローテーションはcopytruncateにしておくと良い。

### 停止

停止はpidを渡すだけ、特にENVはいらないけど気分的に追加。

```stop.sh
#!/usr/bin/env bash
export PATH="/home/node/.nave/installed/0.8.2/bin:$PATH"
export NODE_ENV=production
start-stop-daemon -K \
 -p /var/run/myapp.pid \
 -R=TERM/30/KILL/5
```

Capistranoでデプロイする場合はスクリプトとpidをsharedの下に置き、-dに#{current_path}を指定するとよい。



## monitのコンフィグと操作
monit自身は遠慮無くUpStartで起動しておこう。
最低限の設定としてはこのくらいか。

```myapp_monit.conf 
check process myapp with pidfile /var/run/myapp.pid
  group myapp
  start program "/opt/myapp/bin/start.sh"
  stop program "/opt/myapp/bin/stop.sh"
  if 2 restarts within 3 cycles then unmonitor 
```

グループ指定をしておくと

### monitでnode.jsのアプリを操作
起動するとき

    monit start myapp

停止するとき

    monit stop myapp


これで死活監視とリスポーンとnode.jsのバージョン変更が楽に行える。
