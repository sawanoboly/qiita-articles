<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

chef-clientのエラーを逐一拾うのが面倒な時。

`knife status`を実行すれば各クライアントの最終正常終了時間がとれるので、ChefServer上で備え付けのクライアント`chef-webui`を使ってみる。

```shell:Shell-Out
# /usr/bin/knife status -k /etc/chef/webui.pem -u chef-webui
33 minutes ago, mmonit.example.com, mmonit.example.com, xxx.xxx.xxx.xxx, smartos 5.11.
20 minutes ago, mongodb08.example.jp, mongodb08.example.jp, xxx.xxx.xxx.xxx, ubuntu 12.04.
11 minutes ago, mongodb07.example.jp, mongodb07.example.jp, xxx.xxx.xxx.xxx, ubuntu 12.04.
9 minutes ago, rabbitmq01.example.jp, rabbitmq01.example.jp, xxx.xxx.xxx.xxx, smartos 5.11.
9 minutes ago, giraffi-chef.example.jp, giraffi-chef.example.jp, xxx.xxx.xxx.xxx, ubuntu 12.04.
3 minutes ago, mongodb06.example.jp, mongodb06.example.jp, xxx.xxx.xxx.xxx, ubuntu 12.04.
2 minutes ago, rabbitmq03.example.jp, rabbitmq03.example.jp, xxx.xxx.xxx.xxx, smartos 5.11.
1 minute  ago, rabbitmq02.example.jp, rabbitmq02.example.jp, xxx.xxx.xxx.xxx, smartos 5.11.
```

`status`に`-F Format`オプションは利かないので、出力が得られたらメールするなりパースして引っ掛けるなりします。`node attribtes`に対してならQUERYも可。
