
ecs-agentの1.9.0以降では、DockerのLogDriverにawslogsを指定できます。

で、ECSを直接使う分には`logConfiguration`で指定すればよいのですが、ElasticBeanstalkから利用できるECS(に限らずですが)環境はいつもちょっと古く、この記事時点の最新でも1.8.2です。

待ってりゃ1.9.0にはなるんでしょうが。。ひとまず強引にやってみました。


## 段取りとebextensions

awslogsを使えるように設定しつつ、ecs-agentを更新する段取り。ひとまずこれでうまく動いたのでよしとします。

- ECS_AVAILABLE_LOGGING_DRIVERS設定を追加し、オプションの値にawslogsを許可する
    - すでにあれば何もしない
- amazon-ecs-agentの`v1.9.0`をpullし、無理やりlatestタグを付与する
    - ※ 別にlatestをそのままpullしても構いません。
- ecs-agentコンテナを削除して、リスタート

以上を踏まえた`ebextensions`がこちら。ecs-agent再起動へのステップで色々やっています、なくてもよい行も多分あるかなあ。

```.ebextensions/01-logdriver.config
files:
  "/root/setup-available-log-dirvers.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/bin/sh
      set -e
      if ! grep awslogs /etc/ecs/ecs.config &> /dev/null
      then
        echo 'ECS_AVAILABLE_LOGGING_DRIVERS=["json-file","syslog","awslogs"]' >> /etc/ecs/ecs.config
        /usr/bin/docker pull amazon/amazon-ecs-agent:v1.9.0
        /usr/bin/docker tag -f amazon/amazon-ecs-agent:v1.9.0 amazon/amazon-ecs-agent:latest
        /usr/libexec/amazon-ecs-init pre-stop
        /usr/bin/docker rm ecs-agent
        /usr/libexec/amazon-ecs-init pre-start
        /sbin/initctl start -n ecs
      fi

container_commands:
  01-configure-awslogs:
    command: /root/setup-available-log-dirvers.sh
```

これをデプロイした後にenvをクローンしても大丈夫だったので、まあなんとかなってるんでしょう。

このebextensionsの元ネタはこちら。 [http://stackoverflow.com/questions/35789111/how-to-use-fluentd-log-driver-on-elastic-beanstalk-multicontainer-docker](http://stackoverflow.com/questions/35789111/how-to-use-fluentd-log-driver-on-elastic-beanstalk-multicontainer-docker)

## サンプル Dockerrun.aws.json

では実際に使ってみるとして、指定の例を置いときます。`logs-group=eb-awslog-test`、ストリームはコンテナIDでログが投入されました。

```Dockerrun.aws.json
{
  "AWSEBDockerrunVersion": 2,
  "volumes": [
  ],
  "containerDefinitions": [
    {
      "name": "distribution",
      "image": "registry:2",
      "essential": true,
      "cpu": "1",
      "memory": 128,
      "mountPoints": [
      ],
      "portMappings": [
        {
          "hostPort": 80,
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "logConfiguration" : {
        "logDriver": "awslogs",
        "options": {
          "awslogs-region": "ap-northeast-1",
          "awslogs-group": "eb-awslog-test"
        }
      }
    }
  ]
}
```


## おわりに

もし使ってみようと思っても、真似する前に既にecs-agentのバージョンが1.9.0以降になってないか確認しましょう。

バージョンは`agent --version`で出てきます。

```
# docker exec -it 81f4c4d3e819 /agent --version
2016-05-23T10:23:25Z [INFO] Starting Agent: Amazon ECS Agent - v1.9.0 (0931217)
2016-05-23T10:23:25Z [INFO] Loading configuration
2016-05-23T10:23:25Z [WARN] Invalid value for task cleanup duration, will be overridden to 3h0m0s module="config" parsed value="0" minimum threshold="1m0s"
Amazon ECS Agent:
	Version: 1.9.0
	Commit: 0931217
	DockerVersion: 1.9.1
```
