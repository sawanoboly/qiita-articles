

AWSリソース同士の連携でLambdaをつかうのは便利だけどeventが毎度複雑でたいへん。
今回はDynamoDB StreamをPythonで拾うとき、主要な処理以外をなんとか簡潔に書けるようにしたいチャレンジ。

[DynamoDB ストリーム と AWS Lambda のトリガー - Amazon DynamoDB](http://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/Streams.Lambda.html)


> この記事の内容はライブラリにしてPyPIに登録してあります。
> 
> - [dynamodb-stream-dispatcher · PyPI](https://pypi.org/project/dynamodb-stream-dispatcher/)
> - [github/higanworks/dynamodb_stream_dispatcher](https://github.com/higanworks/dynamodb_stream_dispatcher)


## 共通処理の仕様

このあたりの処理を使いまわせば楽になりそう。

- 対応する関数(lambda含む)を登録して、レコードごとに実行したい
  - 呼ばれる側はコンテキスト決め打ちでよいので、分岐とか不要
- Itemのデータを取り出しやすくする
  - ついでにDynamoおなじみの型付きでくるので、PythonのDictに変換する
- (Option) 例外処理


で、エントリーポイントは、こんな風に書けたらよいかなと。


```python:handler(予定)
def lambda_handler(event, context):
    ds = DynamoStreamDispatcher(event)
    ds.on_insert.append(lambda rec: print(rec.event_name)) # lambda OK
    ds.on_remove.append(after_remove1)
    ds.on_remove.append(after_remove2) #複数の処理 OK
    ds.on_modify.append(after_modify)


    ds.dispatch()
    return True
```


ハンドラを登録してdispatchする感じ。


## ざっくり作ってみる

とりあえず当初にきめたlambda_handlerに書きたい内容を実装してみた。


```python:lambda_function.py
from __future__ import print_function

import json
from boto3.dynamodb.types import TypeDeserializer
deser = TypeDeserializer()

print('Loading function')


class DeRecord:
    ## Itemをデシリアライズしたもの
    def __init__(self, rec):
        self.event_name = rec['eventName']
        self.old = self._desi(rec['dynamodb'].get('OldImage'))
        self.new = self._desi(rec['dynamodb'].get('NewImage'))

    def _desi(self, image):
        d = {}
        if image:
            for key in image:
                d[key] = deser.deserialize(image[key])
        return d

        
class DynamoStreamDispatcher:
    def __init__(self, event):
        self.on_insert = []
        self.on_remove = []
        self.on_modify = []
        self.records   = []
        for r in event['Records']:
            # レコードはdictに加工しちゃう。
            self.records.append(DeRecord(r))

        self.raw = event

    def dispatch(self):
        """
        synced dispatcher
        """
        results = []
        for r in self.records:
            try:
                for runner in getattr(self, 'on_' + r.event_name.lower()):
                    results.append(runner(r))
            except AttributeError:
                print("Unknown event " + r.event_name)
                continue
                    
        return results


## ここから、個別Lambda用処理の関数サンプル。引数は加工済みのレコード
def after_remove1(rec):
    print("deleted")
    return None

def after_remove2(rec):
    print(rec.old)
    return None


def after_modify(rec):
    print("key updated...")
    print(rec.old['Message'])
    print(rec.new['Message'])
    return None


def lambda_handler(event, context):
    ds = DynamoStreamDispatcher(event)
    ds.on_insert.append(lambda rec: print(rec.event_name))
    ds.on_remove.append(after_remove1)
    ds.on_remove.append(after_remove2)
    ds.on_modify.append(after_modify)

    ds.dispatch()
    return True
```

Sample event templateからDynamoDB Update(※末尾に付属)を流してみる。


```
START RequestId: 6ed79996-0ecc-11e7-8985-db0ca21254c3 Version: $LATEST
INSERT
key updated...
New item!
This item has changed
deleted
{u'Message': u'This item has changed', u'Id': Decimal('101')}
END RequestId: 6ed79996-0ecc-11e7-8985-db0ca21254c3
```

登録した関数がそれぞれ実行されてOK。

ほぼ自分用ライブラリだけど、PyPIに似たようなのがなければ登録して使いまわそうかな。
あとは差分とかをうまいこと格納したいね。

----

付録: DynamoDB Updateのサンプルイベント

```json:
{
  "Records": [
    {
      "eventID": "1",
      "eventVersion": "1.0",
      "dynamodb": {
        "Keys": {
          "Id": {
            "N": "101"
          }
        },
        "NewImage": {
          "Message": {
            "S": "New item!"
          },
          "Id": {
            "N": "101"
          }
        },
        "StreamViewType": "NEW_AND_OLD_IMAGES",
        "SequenceNumber": "111",
        "SizeBytes": 26
      },
      "awsRegion": "us-west-2",
      "eventName": "INSERT",
      "eventSourceARN": "arn:aws:dynamodb:us-west-2:account-id:table/ExampleTableWithStream/stream/2015-06-27T00:48:05.899",
      "eventSource": "aws:dynamodb"
    },
    {
      "eventID": "2",
      "eventVersion": "1.0",
      "dynamodb": {
        "OldImage": {
          "Message": {
            "S": "New item!"
          },
          "Id": {
            "N": "101"
          }
        },
        "SequenceNumber": "222",
        "Keys": {
          "Id": {
            "N": "101"
          }
        },
        "SizeBytes": 59,
        "NewImage": {
          "Message": {
            "S": "This item has changed"
          },
          "Id": {
            "N": "101"
          }
        },
        "StreamViewType": "NEW_AND_OLD_IMAGES"
      },
      "awsRegion": "us-west-2",
      "eventName": "MODIFY",
      "eventSourceARN": "arn:aws:dynamodb:us-west-2:account-id:table/ExampleTableWithStream/stream/2015-06-27T00:48:05.899",
      "eventSource": "aws:dynamodb"
    },
    {
      "eventID": "3",
      "eventVersion": "1.0",
      "dynamodb": {
        "Keys": {
          "Id": {
            "N": "101"
          }
        },
        "SizeBytes": 38,
        "SequenceNumber": "333",
        "OldImage": {
          "Message": {
            "S": "This item has changed"
          },
          "Id": {
            "N": "101"
          }
        },
        "StreamViewType": "NEW_AND_OLD_IMAGES"
      },
      "awsRegion": "us-west-2",
      "eventName": "REMOVE",
      "eventSourceARN": "arn:aws:dynamodb:us-west-2:account-id:table/ExampleTableWithStream/stream/2015-06-27T00:48:05.899",
      "eventSource": "aws:dynamodb"
    }
  ]
}
```

参考:

- [How to convert from DynamoDB wire protocol to native Python object manually with boto3? - Stack Overflow](http://stackoverflow.com/questions/36558646/how-to-convert-from-dynamodb-wire-protocol-to-native-python-object-manually-with)
