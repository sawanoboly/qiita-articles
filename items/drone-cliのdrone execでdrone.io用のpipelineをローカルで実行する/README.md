
[drone.io](http://drone.io)というCI/CDのサービスがあって、自前で運用できるOSS版が存在します。

さてOSS版、こちらhookを受けるserverと、タスクを実行するagentと分かれている感じで、案内通りのセットアップをしようとするとなにかしら専用のサーバが必要です。

ついでに、認証必須でどこかとOAuthの連携、ビルド基本的にwebhook(CLIでも一応可)なので、ちょっと感触を確かめたい程度の時にはserver/agentの構成はちょっと大げさでした。

ドキュメントを見ていたら`drone exec`ってのがあったので試してみます。


## drone-cliのインストール

[CLI Installation | docs.drone.io](http://docs.drone.io/cli-installation/) 

手元はmacOSなので、HomeBrewで。

```
brew tap drone/drone
brew install drone
```

`drone-cli`が使えるようになりました。

```
$ drone --version
drone version 0.7.0
```

一応付け加えておくと、Docker for Mac を動かしておきましょうね。

## drone execでローカル実行

ちょうど良さげなのでバックエンドにredisを使うコードのサンプルを実行してみます。

[Example using Redis | docs.drone.io](http://docs.drone.io/redis-example/)


`.drone.yml`と適当にファイルを設置。

```shell:
$ tree ./ -a
./
├── .drone.yml
└── hello.text

0 directories, 2 files
```


Example から`.drone.yml`の中身をこのようにします。


```.drone.yml
workspace:
  base: /pipeline
  path: src

pipeline:
  build:
    image: redis
    commands:
      - pwd
      - find ./
      - redis-cli -h redis ping
      - redis-cli -h redis set FOO bar
      - redis-cli -h redis get FOO

services:
  redis:
    image: redis
```


`workspace`設定はデフォルトですが、記述できるのかと思い足してみました。。

## drone execでローカル実行

あとは実行してみるだけ。


```shell:drone-exec
$ drone exec 
1:C 07 Jun 07:52:31.302 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.2.8 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

1:M 07 Jun 07:52:31.304 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 07 Jun 07:52:31.304 # Server started, Redis version 3.2.8
1:M 07 Jun 07:52:31.304 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 07 Jun 07:52:31.304 * The server is now ready to accept connections on port 6379
+ pwd
/pipeline/src
+ find ./
./
./.drone.yml
./hello.text
+ redis-cli -h redis ping
PONG
+ redis-cli -h redis set FOO bar
OK
+ redis-cli -h redis get FOO
bar

$ echo $?
0
```

できましたね。

`drone exec --help`をみると`--local`というオプションがあるけども、`DRONE_SERVER/DRONE_TOKEN`を指定しても無視されているので、`drone exec`自体はローカル専用とみて良さそう。

drone.io自体、docker-engineだよりなので、ローカルでもagent相当のことはできるでしょうと。ちゃんと機能があってよかった。
