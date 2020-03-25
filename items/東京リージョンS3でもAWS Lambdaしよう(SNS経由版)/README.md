
> Lambda東京来ましたので、この記事はお役御免となりましたね。

Lambdaの使用例として、よくAmazon S3との連携を見ますが、S3でTokyoリージョンを選択しているとLambdaに通知できない。
さてS3の通知機能、Lambdaは出ないがSNSは出る。LambdaはSNSをSubscribeできる。じゃあ一旦そっち経由すっかと。


参考: Cross-Regionでやりたい時はクラスメソッドさんのこちらの記事で。 [Amazon S3のCross-Region Replicationを使ってAWS Lambdaを発火させる ｜ Developers.IO](http://dev.classmethod.jp/cloud/aws/lambda-execution-by-using-cross-region-replication/ "Amazon S3のCross-Region Replicationを使ってAWS Lambdaを発火させる ｜ Developers.IO")


## S3 to Lambdaのイベント例

Lambdaテンプレートからそのまま拝借、S3のサンプルイベントはこうなっています。

```
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "1970-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "userIdentity": {
        "principalId": "EXAMPLE"
      },
      "requestParameters": {
        "sourceIPAddress": "127.0.0.1"
      },
      "responseElements": {
        "x-amz-request-id": "C3D13FE58DE4C810",
        "x-amz-id-2": "FMyUVURIY8/IgAtTv8xRjskZQpcIZ9KG4V5Wp6S7S/JRWeUWerMUE5JgHvANOjpD"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "sourcebucket",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws:s3:::mybucket"
        },
        "object": {
          "key": "HappyFace.jpg",
          "size": 1024,
          "eTag": "d41d8cd98f00b204e9800998ecf8427e"
        }
      }
    }
  ]
}
```

これをサンプルコードでは`event.Records[0]`としてイベント詳細を取得していますね。

## S3 to Lambda via SNSのイベント例

で、さっきのS3イベントがSNSを通ってくると次のようになりました。
最初の`S3 to Lambda`のサンプルが、丸ごとMessageの中に入ってやってきます。

```
{
  "Records": [
    {
      "EventSource": "aws:sns",
      "EventVersion": "1.0",
      "EventSubscriptionArn": "arn:aws:sns:EXAMPLE",
      "Sns": {
        "TopicArn": "arn:aws:sns:EXAMPLE",
        "Subject": "TestInvoke",
        "UnsubscribeUrl": "EXAMPLE",
        "Type": "Notification",
        "Message": "{\"Records\":[{\"eventVersion\":\"2.0\",\"eventSource\":\"aws:s3\",\"awsRegion\":\"us-east-1\",\"eventTime\":\"1970-01-01T00:00:00.000Z\",\"eventName\":\"ObjectCreated:Put\",\"userIdentity\":{\"principalId\":\"EXAMPLE\"},\"requestParameters\":{\"sourceIPAddress\":\"127.0.0.1\"},\"responseElements\":{\"x-amz-request-id\":\"C3D13FE58DE4C810\",\"x-amz-id-2\":\"FMyUVURIY8/IgAtTv8xRjskZQpcIZ9KG4V5Wp6S7S/JRWeUWerMUE5JgHvANOjpD\"},\"s3\":{\"s3SchemaVersion\":\"1.0\",\"configurationId\":\"testConfigRule\",\"bucket\":{\"name\":\"sourcebucket\",\"ownerIdentity\":{\"principalId\":\"EXAMPLE\"},\"arn\":\"arn:aws:s3:::mybucket\"},\"object\":{\"key\":\"HappyFace.jpg\",\"size\":1024,\"eTag\":\"d41d8cd98f00b204e9800998ecf8427e\"}}}]}",
        "Signature": "EXAMPLE",
        "MessageId": "95df01b4-ee98-5cb9-9903-4c221d41eb5e",
        "Timestamp": "1970-01-01T00:00:00.000Z",
        "SigningCertUrl": "EXAMPLE",
        "Type" : "Notification",
        "SignatureVersion": "1",
        }
      }
    }
  ]
}
```


## S3直接と同等に変換する。

S3から直接で来た`event`と同様に扱うため、こんな感じで加工しました。

```javascript:
exports.handler = function(event, context) {
  message = JSON.parse(event.Records[0].Sns.Message);

...以下略

```

これ以降、messageを元の世にあるS3連携サンプル達のeventと同様に扱えばOKです。

