
ちょっと嬉しい変更だったので記録。

[AWS Batch now sends job state changes to CloudWatch Events](https://aws.amazon.com/jp/about-aws/whats-new/2017/10/aws-batch-now-sends-job-state-changes-to-cloudwatch-events/)


いままでは次のようにしてjobの失敗を観測していました。

- ScheduleタイプのeventsでJobQueue内のJobを取得
- ステータス=`FAILED` が 前回取得より`+1以上`なら通知
  - ※ JobQueueのステータス保持は24時間なので、そのうち減る

メトリクスはdatadogに送って、差分のグラフを用いたモニタリングを設定。んー、大変不安な作りでしたね。
Statusの遷移がCloudWatch Eventsに流れることによって、もうちょっと信頼性のおける通知を仕込んでおけそうです。


## event from Batch

イベントは流れていますが、単品用のイベントはコンソールの選択肢にはまだでてきません。
チュートリアルの通り、カスタムイベントとしてsourceに`aws.batch`を含んだらイベント発生、とします。

[Tutorial: Listening for AWS Batch CloudWatch Events - AWS Batch](http://docs.aws.amazon.com/ja_jp/batch/latest/userguide/batch_cwet.html)


こんなルール。

```
{
  "source": [
    "aws.batch"
  ]
}
```

特定のステータスのみをトリガーにしたい場合は、SNSのチュートリアルにあるように、`detail.status`をルールに追加すればOKですね。


```
{
  "detail-type": [
    "Batch Job State Change"
  ],
  "source": [
    "aws.batch"
  ],
  "detail": {
    "status": [
      "FAILED"
    ]
  }
}
```

detailにはjobQueueのarnが含まれているので、振り分けが必要ならフィルタすればキュー別のイベントを作成することもできます。


## sample events

Jobが`SUCCEEDED`で終わったらこのように流れてきます。

```
{
    "version": "0",
    "id": "7266ead5-3f9e-aee4-6abd-7908713f9a16",
    "detail-type": "Batch Job State Change",
    "source": "aws.batch",
    "account": "************",
    "time": "2017-10-27T04:28:14Z",
    "region": "us-west-2",
    "resources": [
        "arn:aws:batch:us-west-2:************:job/fddb88dd-33f0-478b-933e-27799c94d1b0"
    ],
    "detail": {
        "jobName": "streamtest",
        "jobId": "fddb88dd-33f0-478b-933e-27799c94d1b0",
        "jobQueue": "arn:aws:batch:us-west-2:************:job-queue/mybatchjob",
        "status": "SUCCEEDED",
        "attempts": [
            {
                "container": {
                    "containerInstanceArn": "arn:aws:ecs:us-west-2:************:container-instance/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyy",
                    "taskArn": "arn:aws:ecs:us-west-2:************:task/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
                    "exitCode": 0,
                    "logStreamName": "mybatchjob/default/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
                },
                "startedAt": 1509078455720,
                "stoppedAt": 1509078493270,
                "statusReason": "Essential container in task exited"
            }
        ],
        "statusReason": "Essential container in task exited",
        "createdAt": 1509078431202,
        "retryStrategy": {
            "attempts": 1
        },
        "startedAt": 1509078455720,
        "stoppedAt": 1509078493270,
        "dependsOn": [],
        "jobDefinition": "arn:aws:batch:us-west-2:************:job-definition/mybatchjob:3",
        "parameters": {},
        "container": {
            "image": "************.dkr.ecr.us-west-2.amazonaws.com/mybatchjob:latest",
            "vcpus": 1,
            "memory": 512,
            "command": [
                "args1",
                "args2"
            ],
            "jobRoleArn": "arn:aws:iam::************:role/myBatchContainerRole",
            "volumes": [
                {
                    "host": {
                        "sourcePath": "/srv"
                    },
                    "name": "source"
                }
            ],
            "environment": [],
            "mountPoints": [
                {
                    "containerPath": "/srv",
                    "readOnly": true,
                    "sourceVolume": "source"
                }
            ],
            "readonlyRootFilesystem": true,
            "ulimits": [],
            "exitCode": 0,
            "containerInstanceArn": "arn:aws:ecs:us-west-2:************:container-instance/yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyy",
            "taskArn": "arn:aws:ecs:us-west-2:************:task/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
            "logStreamName": "mybatchjob/default/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx"
        }
    }
}
```

とりあえずSlack通知用のSNSに突っ込んでみます、無事に投稿されました。

![sns_slack.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/de8734b7-804f-ee5a-d02b-c3ad09337e7c.jpeg "sns_slack.jpg")


## おわりに

バッチの種別(朝まとめて確認すればOKとか)にもよりますが、`FAILED`をトリガーとして詳細を通知しやすくなったので使い勝手がよくなりました。


