
Docker for AWS(Azure)は`Moby Linux`と名付けられたbusyboxベースのOSに...
まあ詳しくはこれを見てくれ。 [Docker for AWS 試してみた その1 - Qiita](http://qiita.com/pottava/items/06aa817e581debd150bd)

以下、DockerForなんとかは `D4x`と呼称します。イメージの名前とかがそうなってるし。

## Docker APIへの標準的なアクセスについて。

さて、D4xの操作はドキュメントによるとマネージャノードの専用コンテナにSSHでログインするか、トンネルを貼るという案内だ。

- 参考: [Deploying Apps on AWS/Azure](https://beta.docker.com/docs/deploy/)

D4xはベータ申込者(AWSならアカウントID)に対してDocker社からShareされたAMIをCloudFormation(以下CFn)で起動する。
この特性上、テンプレートを操作してしまえばいかようにでもカスタマイズは可能のように思えるけど、そもそもbusyboxベースからカスタマイズするのはかったるい。

もちろん、コントロール用のDockerコンテナはホストのDockerデーモンをソケット経由で操作できるので、通常の利用ならまったく困らないという感じである。
なるべく仕組みに乗っかるためには、今回のようにAPIを外部公開してーなーという要件に対してはサービスを作ってしまうのが良さそう。


## サービス用Nginx

要は一点、`/var/run/docker.sock`をマウントしたNginxのコンテナにプロキシさせればよいのだった。

コード: https://github.com/sawanoboly/docker-socket-proxy-for-docker-for-aws_azure/tree/master
イメージ: https://hub.docker.com/r/sawanoboly/nginx-dsp/

とりあえずシンプルに`proxy_pass http://unix:/var/run/docker.sock;`でOK。

```
user  root;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    # include /etc/nginx/conf.d/*.conf;

    server {
      listen       80;
      server_name  _;

      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        proxy_pass http://unix:/var/run/docker.sock;
      }

      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
          root   /usr/share/nginx/html;
      }
    }
}
```

## Docker Service

イメージをDockerhubなりどこかにpushした後、bindマウントをつけながら起動しましょう。
特別なことは一点だけ。デプロイする対象の絞込を`node.role == manager`にするとmanager担当のノードでだけDockerコンテナが起動します。
Worker側に立ってしまうとSwarm系のAPIが使えないのよね。

```
docker service create \
  --name nginx-dsp \
  -p 80:80 \
  --replicas 3 \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock \
  --mount type=bind,source=/usr/bin/docker,target=/usr/bin/docker \
  --constraint 'node.role == manager' \
  sawanoboly/nginx-dsp:latest
```

`docker`コマンドはデバッグ用なので外してもOK。`/var/log/nginx`のあたりもお好みで。

### Curlでテスト

さて、サービスが作れたらば、Dockerサービス用のELBにリクエストしてみよう。`docker service ls`相当のAPIをコール。

```
$ curl -s http://xxxxxx-elb-xxxxxxxx.us-east-1.elb.amazonaws.com/services | jq .
[
  {
    "ID": "68tcz6xvcj9iw1yvy77gcajx5",
    "Version": {
      "Index": 148
    },
    "CreatedAt": "2016-08-16T05:08:26.089395158Z",
    "UpdatedAt": "2016-08-16T05:08:26.100058581Z",
    "Spec": {
      "Name": "nginx-dsp",
      "TaskTemplate": {
        "ContainerSpec": {
          "Image": "sawanoboly/nginx-dsp:latest",
          "Mounts": [
            {
              "Type": "bind",
              "Source": "/var/run/docker.sock",
              "Target": "/var/run/docker.sock"
            },
            {
              "Type": "bind",
              "Source": "/var/log/nginx",
              "Target": "/var/log/nginx"

...
```

できました。 ※認証はNginxでやろうね。。。

## おわりに

今回はrootでnginxを起動しちゃったが、CFnテンプレートを参考にして`/etc/passwd`などをマウントしてからdockerユーザーで起動するように変えておこう。

D4xはカスタマイズするより、k8sのように専用のサービスを追加する形がよいですね。スケジュールジョブとかもイメージを作ってサービスにしましょ。

----

追記： Basic認証の追加とユーザーをDockerに変更する処理をいれました。

使い方はこちら。 https://github.com/sawanoboly/docker-socket-proxy-for-docker-for-aws_azure
