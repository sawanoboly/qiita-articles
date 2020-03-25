
先日こんな記事を見かけて、

- [Kubernetesの学習のためにMastodonを構築したら勉強になった | ハラミTech](https://blog.haramishio.xyz/post/000022/)

Web, Worker, Websocketと要素が揃っているMastodonのスタックをXXで動かしてみる、というのは感触をつかむのによいなと思いました。

で、じゃあAWS Fargateではどうかなと、このようにしてみました。

![Amazon ECS .png](https://qiita-image-store.s3.amazonaws.com/0/7454/f7faf581-7f7b-0ea6-69a0-c2968c4bcc0a.png "Amazon ECS .png")


## 事前準備

Dockerコンテナ以外で、マネージドサービスで揃えられるものは優先的にそちらを使用します。

- AWSで用意
  - VPS (既存でも新規でも)
  - RDS (Postgresで起動)
  - ACMで証明書 (ALBに適用する)
  - ALB
	  - TargetGroup-web (HTTP/3000)
	  - TargetGroup-stream (HTTP/4000)
  - S3バケット (PAPERCLIP Wikiのポリシー適用が楽)
- Redis => [Redis Labs](https://redislabs.com/) / 30MBまでならFreeプランでOK
  - ElastiCacheが使いたければそちらでも
- MailGunアカウント
  - 専用の認証情報をつくるとよいです

ALBは以下のルーティングを最後に割り当てます。

- `/` (default)=> TargetGroup-web
- `/api/v1/streaming*` TargetGroup-stream

こんな感じです。

![EC2 Management Console 2017-12-03 14-04-04.png](https://qiita-image-store.s3.amazonaws.com/0/7454/e11ad232-9aa0-8b4f-664a-72cb6a101f78.png "EC2 Management Console 2017-12-03 14-04-04.png")


## Fargateを考慮したTask definitions

事前準備の内容が揃っていれば、Dockerのコンテナとして動かすのは次の3要素。

- Web(Rails/puma)
- Sidekiq(Ruby)
- Streaming(Node.js)

これらをTask definitionsに起こすとして、全部のコンテナ定義を一つにまとめるとスケール変更が大げさになるので、それぞれを別のECSサービスとして登録できるようにバラします。
compose感はなくなりますが、Fargateならその方が良いように思います。

各タスクはmastodonリポジトリのdocker-compose.ymlをベースに、kubedonを参考にしつつアレンジ。

- [mastodon/docker-compose.yml at v2.0.0 · tootsuite/mastodon](https://github.com/tootsuite/mastodon/blob/v2.0.0/docker-compose.yml)
- [jviide/kubedon: A Mastodon conf for Kubernetes / Google Container Engine](https://github.com/jviide/kubedon)




### .env.production


大筋として本家のdocker-composeを使いまわすので、もともと用意されている`.env.production`から環境変数という仕組みをほぼ踏襲します。
今回このようにしました。

```bash:.env.production
RAILS_ENV=production
NODE_ENV=production
REDIS_URL=redis://:<RedisLabsのパスワード>@<RedisLabsのエンドポイント>

DB_HOST=<RDSのエンドポイント>
DB_USER=<RDSのユーザ>
DB_NAME=<RDSのDB名>
DB_PASS=<RDSのパスワード>
DB_PORT=5432

LOCAL_DOMAIN=<ALBに向けるドメイン>
LOCAL_HTTPS=true

PAPERCLIP_SECRET=<rake secretで作成>
SECRET_KEY_BASE=<rake secretで作成>
OTP_SECRET=<rake secretで作成>

VAPID_PRIVATE_KEY=<rake mastodon:webpush:generate_vapid_key で作成>
VAPID_PUBLIC_KEY=<rake mastodon:webpush:generate_vapid_key で作成>

DEFAULT_LOCALE=ja

SMTP_SERVER=smtp.mailgun.org
SMTP_PORT=465
SMTP_LOGIN=< MAILGUNの認証ユーザ >
SMTP_PASSWORD=< MAILGUNのパスワード >
SMTP_FROM_ADDRESS=noreply@< MAILGUNのドメイン >
SMTP_TLS=true

S3_ENABLED=true
S3_BUCKET=< S3のバケット >
AWS_ACCESS_KEY_ID=< S3用のIAMユーザのキー >
AWS_SECRET_ACCESS_KEY=< S3用のIAMユーザのシークレットキー >
S3_REGION=< S3のリージョン >
S3_PROTOCOL=https
S3_HOSTNAME=< S3のリージョン別エンドポイント >

STREAMING_CLUSTER_NUM=1
```


### web

web用に`docker-compose.web.yml`, `ecs-param.web.yml`を用意します。ecs-cliの取り回しを考えると、ディレクトリ分けて標準のファイル名を使う方が楽だったかも。
また、汎用性を考えたらloggingは分けて、--fileで追加指定してもよいはずです。

```yaml:docker-compose.web.yml
version: '2'
services:
  web:
    image: gargron/mastodon:v2.0.0
    env_file: .env.production
    command: ["/bin/ash", "-c", "rails db:migrate && rails assets:precompile && rails s -p 3000 -b 0.0.0.0"]
    ports:
      - "3000:3000"
    mem_limit: 2GB
    logging:
      driver: awslogs
      options:
        awslogs-group: mstdn-fargate-demo
        awslogs-region: us-east-1
        awslogs-stream-prefix: web

```

assign_public_ipはイメージの取得にDockerhubを使う場合ENABLEDでないとPullできません。(※他にルートを用意できればおそらく可)
SGはALBとさえ通信できればOKなので、特にきにせずENABLEDで良いでしょう。


```yaml:ecs-param.web.yml
version: 1
task_definition:
  ecs_network_mode: awsvpc
  task_execution_role: ecsTaskExecutionRole
  task_size:
    cpu_limit: 512
    mem_limit: 2GB
  services:
    web:
      essential: true
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - <YOUR_SUBNETa>
        - <YOUR_SUBNETb>
      security_groups:
        - <YOUR_SG>
      assign_public_ip: ENABLED
```


webは他に比べてスペック高、ほぼ`asset:precompile`のため(メモリが足りないと落ちる)です。

作成したファイルたちを指定して`ecs-cli compose`を使ってサービス登録するにはこんな感じ。
`CLUSTER_NAME`=既存ECSクラスタの名前、`TARGET_GROUP_WEB`=参加させるALBのターゲットグループに置き換えで。

```bash:
$ ecs-cli compose \
  --file docker-compose.web.yml \
  --ecs-params ecs-param.web.yml \
  --project-name mstdn-fargate-web \
  --cluster ${CLUSTER_NAME} \
  service up --launch-type FARGATE \
  --container-name web \
  --container-port 3000 \
  --target-group-arn ${TARGET_GROUP_WEB}
  # --timeout 0
```

`--target-group-arn`は後から変更できないので、変更が必要な場合はサービスを一旦消すか別名で作成です。

なお実行の際、CPUをケチる(<1024)と`--timeout 0`を付与しておかないとデフォルト待ちの5分ではassets生成が終わりません。
このあたり考慮すると、実際に運用する場合は起動時間のことも考えて、よそでprecompileして`public`を別に用意するのが得策でしょう。


### streaming

streaming用に`docker-compose.streaming.yml`, `ecs-param.streaming.yml`を用意します。ecs-cliの取り回し(以下略


```yaml:docker-compose.streaming.yml
version: '2'
services:
  streaming:
    image: gargron/mastodon:v2.0.0
    env_file: .env.production
    command: ["npm", "run", "start"]
    ports:
      - "4000:4000"
    logging:
      driver: awslogs
      options:
        awslogs-group: mstdn-fargate-demo
        awslogs-region: us-east-1
        awslogs-stream-prefix: streaming
```

streamingのパラメータ、サイズは最小。Fargateは[CPUとメモリの組み合わせに制限](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#task_size)があるようです。

```yaml:ecs-param.streaming.yml
version: 1
task_definition:
  ecs_network_mode: awsvpc
  task_execution_role: ecsTaskExecutionRole
  task_size:
    cpu_limit: 256
    mem_limit: 512
  services:
    streaming:
      essential: true
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - <YOUR_SUBNETa>
        - <YOUR_SUBNETb>
      security_groups:
        - <YOUR_SG>
      assign_public_ip: ENABLED
```

作成したファイルたちを指定して`ecs-cli compose`を使ってサービス登録するにはこんな感じ。webとほぼおなじですね。
`CLUSTER_NAME`=既存ECSクラスタの名前、`TARGET_GROUP_STREAMING`=参加させるALBのターゲットグループに置き換えで。

```bash:
$ ecs-cli compose \
  --file docker-compose.streaming.yml \
  --ecs-params ecs-param.streaming.yml \
  --project-name mstdn-fargate-streaming \
  --cluster ${CLUSTER_NAME} \
  service up --launch-type FARGATE \
  --container-name streaming \
  --container-port 4000 \
  --target-group-arn ${TARGET_GROUP_STREAMING}
```



### sidekiq


sidekiq用に`docker-compose.sidekiq.yml`, `ecs-param.sidekiq.yml`を用意します。ecs-cliの取り回し(以下略

```yaml:docker-compose.sidekiq.yml
version: '2'
services:
  sidekiq:
    image: gargron/mastodon:v2.0.0
    env_file: .env.production
    command: ["sidekiq", "-q", "default", "-q", "mailers", "-q", "pull", "-q", "push"]
    logging:
      driver: awslogs
      options:
        awslogs-group: mstdn-fargate-demo
        awslogs-region: us-east-1
        awslogs-stream-prefix: sidekiq
```

こちらも最小でOK。

```yaml:ecs-param.sidekiq.yml
version: 1
task_definition:
  ecs_network_mode: awsvpc
  task_execution_role: ecsTaskExecutionRole
  task_size:
    cpu_limit: 256
    mem_limit: 512
  services:
    sidekiq:
      essential: true
run_params:
  network_configuration:
    awsvpc_configuration:
      subnets:
        - <YOUR_SUBNETa>
        - <YOUR_SUBNETb>
      security_groups:
        - <YOUR_SG>
      assign_public_ip: ENABLED
```


`CLUSTER_NAME`=既存ECSクラスタの名前に置き換えで。

```shell:
$ ecs-cli compose \
  --file docker-compose.sidekiq.yml \
  --ecs-params ecs-param.sidekiq.yml \
  --project-name mstdn-fargate-sidekiq \
  --cluster ${CLUSTER_NAME} \
  service up --launch-type FARGATE
```



## ECS状況

```bash:
$ ecs-cli ps --cluster ${CLUSTER_NAME}
Name                                            State                                                                               Ports                          TaskDefinition
775a399c-19e1-4e2a-bd3d-98f8b4d7304e/streaming  RUNNING                                                                             34.204.191.189:4000->4000/tcp  mstdn-fargate-streaming:1
921f2d1a-0e4f-49a6-a5b5-434e2b91cf11/sidekiq    RUNNING                                                                                                            mstdn-fargate-sidekiq:2
edb1ba2e-533d-4afd-aaee-39db59ee616a/web        RUNNING                                                                             34.204.174.122:3000->3000/tcp  mstdn-fargate-web:15

$ aws ecs list-services --cluster ${CLUSTER_NAME} 
{
    "serviceArns": [
        "arn:aws:ecs:us-east-1:xxxxxxxxxxxx:service/mstdn-fargate-web", 
        "arn:aws:ecs:us-east-1:xxxxxxxxxxxx:service/mstdn-fargate-sidekiq", 
        "arn:aws:ecs:us-east-1:xxxxxxxxxxxx:service/mstdn-fargate-streaming"
    ]
}
```

それぞれ別のECSサービスとして動かせたので、個別にコンテナ数を変更できて良い感じのはず。



![Mastodon 2017-12-03 14-02-01.png](https://qiita-image-store.s3.amazonaws.com/0/7454/9d922ab5-83d3-f217-7161-3af162be15eb.png "Mastodon 2017-12-03 14-02-01.png")


## Mastodon admin登録

さて、Mastodonの管理者になるには`mastodon:make_admin`タスクを実行すればよいのですが、Fargate上のDockerコンテナで何か実行(docker exec相当)できるのだろうか。

さっとアタッチはできないぽいので、これもtask definitionからやりますか。

yamlをかいてー

```yaml:docker-compose.web_make_admin.yml
version: '2'
services:
  web:
    image: gargron/mastodon:v2.0.0
    env_file: .env.production
    command: ["/bin/ash", "-c", "rails mastodon:make_admin USERNAME=sawanoboly"]
    mem_limit: 512MB
    logging:
      driver: awslogs
      options:
        awslogs-group: mstdn-fargate-demo
        awslogs-region: us-east-1
        awslogs-stream-prefix: railstask
```

`ecs-cli compose up`と。


```bash:
$ ecs-cli compose \
  --file docker-compose.web_make_admin.yml \
  --ecs-params ecs-param.web.yml \
  --project-name mstdn-fargate-makeadmin \
  --cluster ${CLUSTER_NAME} \
  up --launch-type FARGATE
```

で、管理メニューアクセスオープン。

![レポート - Mastodon 2017-12-03 14-34-44.png](https://qiita-image-store.s3.amazonaws.com/0/7454/dc229af8-5ab5-fbc2-dd8e-6ffd662dbfae.png "レポート - Mastodon 2017-12-03 14-34-44.png")

少々手間だが、できるのでよいです。

## おわりに

これまではスケール変更を柔軟にしようと思ったら結局ECSクラスタなりk8s、docker swarmなどのノードを確保しておく必要があったことを考えると、リソース配分が目に見える分だけになるのでシンプルです。

Fargateで起動するコンテナと通常のEC2インスタンスを比較すると、数字上で同等のCPU/メモリのインスタンスを維持するよりFargateのほうがちょっと割高になるはずです。
しかし、EC2ではサービス用のプロセス以外にも色々動かしたりケアする場所も多いため、コンテナより余分に性能やらディスクやらが必要になります。Fargateは下限ギリギリを攻めてもよい分、トータルでどっちがよいかは動かしたいものによるでしょう。

スケールアウトの対応速度は比較にならないほどFargateの方がはやいです。※この記事の例での構成ではコンテナイメージの最適化をしてないので少々もたもたしますが。

Fargate専用のSpotFleetでないかなあ。
