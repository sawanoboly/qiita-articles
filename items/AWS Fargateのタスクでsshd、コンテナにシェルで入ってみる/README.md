
とりあえずやってみたくなるやつ。

今回使用したDockerイメージのソース、タスク用のJSONなどはこちら。

- https://github.com/sawanoboly/amazonlinux-sshd

一回ログインしたらセッション切断時にタスクも終了する(sshdが`-d`で起動している)ようにしています。

## ログインしてみる

`ssh root@52.90.198.xxx` で。環境変数`ROOT_PW`をrun-taskで指定していなければパスワード`rooooot`で。

一応amazonlinuxをベースにしており、ログインしたときの環境変数はこんな感じ。 (※ 元の環境変数から`AWS_*`のみ自動でexportするように指定済。)

```bash:
$ ssh root@34.236.216.xxx
root@34.236.216.xxx's password: 
There was 1 failed login attempt since the last successful login.
debug1: PAM: reinitializing credentials
debug1: permanently_set_uid: 0/0

Environment:
  USER=root
  LOGNAME=root
  HOME=/root
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/aws/bin
  MAIL=/var/mail/root
  SHELL=/bin/bash
  SSH_CLIENT=xxxxxxxxx 57401 22
  SSH_CONNECTION=xxxxxxxxx 57401 10.0.1.188 22
  SSH_TTY=/dev/pts/0
  TERM=xterm-256color
  AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/2bf9b4ff-70a8-4d84-9733-xxxxxxx
  AWS_DEFAULT_REGION=us-east-1
  AWS_REGION=us-east-1

-bash-4.2# 
```

普通のメタデータは取れません。

```
# ec2-metadata 
[ERROR] Command not valid outside EC2 instance. Please run this command within a running EC2 instance.
```

Task Roleの取得はOK

```bash:
-bash-4.2# curl 169.254.170.2$AWS_CONTAINER_CREDENTIALS_RELATIVE_URI | jq .
{
  "RoleArn": "arn:aws:iam::xxxxxxxx:role/ecsTaskExecutionRole",
  "AccessKeyId": "ASIAXXXXXXXXX",
  "SecretAccessKey": "XXXXXXXXXX",
  "Token": "xxx==",
  "Expiration": "2017-12-04T11:37:56Z"
}
```


という程度の軽い動作チェック程度には使えるというのと、一応このまま何かを頑張って構築しても稼働はするはず。。※コンテナポートはtask-definitionに追加で。
ホストのファイルシステムをマウントとかできないのであまり探るところもないけれど、気になるところは各自でチェックという感じで。



## ふろく: AWS-CLIでrun-task

CLIの実行例にTask Roleはつけていないので、必要なら`register-task-definition`か、`run-task`のoverrideで付与します。

task-definitionを作成する。

```bash:
$ EXEC_ROLE_ARN=<ロールのARN> # AmazonECSTaskExecutionRolePolicyが付いたなにかしらのRole
$ aws ecs register-task-definition \
  --execution-role-arn $EXEC_ROLE_ARN \
  --family ssh-login \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --container-definitions "`cat example/task-def-sshlogin.json`"

> {
>    "taskDefinition": {
>        "taskDefinitionArn": "arn:aws:ecs:us-east-1:xxxxxxxx:task-definition/ssh-login:2",
>        "containerDefinitions": [
...
```

taskを実行する。

```bash:
$ CLUSTER_NAME=<クラスタの名前>
$ SUBNET_ID=<コンテナを実行するサブネット>
$ SG_ID=<TCP/22が開いているSG>
$ aws ecs run-task \
  --cluster $CLUSTER_NAME \
  --launch-type FARGATE \
  --task-definition ssh-login:1 \
  --network-configurationawsvpcConfiguration="{subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"

#=> タスクのIDが返ってくるで変数に入れておくと良い
```

ENIのPublicIPはrun-taskの時点ではすぐにわからないので、ENIのIDがわかるまで適当にdescribe。

```bash:
$ TASK_ARN=arn:aws:ecs:us-east-1:xxxxxxxxxxx:task/677c0132-9086-4087-bd18-xxxxxxxxxxx
$ aws ecs --region us-east-1 describe-tasks \
  --cluster $CLUSTER_NAME \
  --tasks arn:aws:ecs:us-east-1:xxxxxxxxxxx:task/677c0132-9086-4087-bd18-xxxxxxxxxxx

# =>
...

                {
                    "id": "xxxxxxxxxxx",
                    "type": "ElasticNetworkInterface",
                    "status": "ATTACHED",
                    "details": [
                        {
                            "name": "subnetId",
                            "value": "subnet-xxxxxxx"
                        },
                        {
                            "name": "networkInterfaceId",
                            "value": "eni-xxxxxxxx"
                        },
...

```

ENIのIDが分かったらPublicIPをとります。

```bash:
$ aws ec2 --region us-east-1 describe-network-interfaces --network-interface-ids eni-xxxxxx | jq .NetworkInterfaces[].Association.PublicIp
"52.90.198.xxx"
```

ENIあたりはManagementConsole見ながらのほうが楽ですけどね。
