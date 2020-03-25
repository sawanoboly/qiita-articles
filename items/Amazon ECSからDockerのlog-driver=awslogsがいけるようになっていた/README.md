
最近、Amazon ECSのコンソールに 

> ECS now supports Docker 1.9 ...

って出ます。

Dockerは1.9でlog-driverにAWSの[CloudWatch Logs](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/DeveloperGuide/CWL_GettingStarted.html)を指定できるようになった。

ただECSは1.7系だったので、しばらく次のようなプロバイダ逆転現象で使っていたんです。


- さくらのクラウド(等)においてあるDocker-Machine: Dockerのlog-driverでawslogs指定。
    - => CloudWatch Logsとの連携楽ちん
- Amazon ECSで動かしているサービス: syslog経由でlogsコンテナからawslogsへ。
    - => 連携がなんかめんどっちい


ベースAMIのDocker1.9への更新で、専用のlogsコンテナもういらなくなったはずだと思うので少し調べることにした。


## ECSで使われるAMIからインスタンスを作成

ECS用のAMIはマーケットプレイスにいるので、選んで起動します。

![EC2_Management_Console.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/484e4ed8-e356-34d4-d33d-243d9c005ac7.jpeg "EC2_Management_Console.jpg")

ついでに次のようなポリシーをアタッチしたInstance Roleで行きましょう。


```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```


## EC2インスタンスから、取り急ぎ送信の確認

```
$ yum -y install awscli awslogs

...

Complete!
```

log_groupとリージョンあたりを編集して、

```
# service awslogs start
Starting awslogs:                                          [  OK  ]
```

```
# logger -p local1.info LOGGER
```

ちょい連打してしまった。

![CloudWatch_Management_Console.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/1b627b6a-7741-41cd-a969-06a7ec4a3515.jpeg "CloudWatch_Management_Console.jpg")

まずインスタンスロールには問題ないことを確認。


## Dockerデーモンから直接CloudWatch Logsへ

DockerとCloudWatch Logsの連携、(AWS外から送信する場合)通常は`/etc/(sysconfig|default)/docker`などに環境設定、AWSアクセスキーなどを置いてからDockerデーモンを再起動する仕込みが必要だ。

今回はその段取りをまるっと無視しておもむろにlog-driver指定のdocker runしてみる。インスタンスロールが使われるならこれでいいはず。

ざっくりDockerfileを作ってビルド。

```Dockerfile 
FROM ubuntu

RUN apt-get -y update && apt-get install -y nginx

CMD ['/bin/bash']
```

その後、`log-driver`,`log-opt`でグループ等のオプションをわたしてrun。

```
$ sudo docker run -it --rm --log-driver=awslogs --log-opt awslogs-region=ap-northeast-1 --log-opt awslogs-group=ecs-test --log-opt awslogs-stream=ecs nginx nginx -V

## 以下出力

nginx version: nginx/1.4.6 (Ubuntu)
built by gcc 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04) 
TLS SNI support enabled
configure arguments: --with-cc-opt='-g -O2 -fstack-protector --param=ssp-buffer-size=4 -Wformat -Werror=format-security -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-debug --with-pcre-jit --with-ipv6 --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_addition_module --with-http_dav_module --with-http_geoip_module --with-http_gzip_static_module --with-http_image_filter_module --with-http_spdy_module --with-http_sub_module --with-http_xslt_module --with-mail --with-mail_ssl_module
```

↓お、イケるじゃないの。

![CloudWatch_Management_Console.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/85c5b887-c13e-1cd6-ba90-5b36b9514d5f.jpeg "CloudWatch_Management_Console.jpg")


## 結論

Amazon ECSでも(log-driver=awslogsのオプションが渡りさえすれば)Logs連携がいける。インスタンスロールでOK。
