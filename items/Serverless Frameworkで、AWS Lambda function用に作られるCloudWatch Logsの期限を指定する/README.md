
某所でServerless Frameworkが採用されており、コンソールでCloudWatch Logsをみると、`Never Expire`のlogsが沢山できていた。
これは真綿で首を絞めるような課金になるので、`serverless.yml`上で`logRetentionInDays`を設定するやり方を調べた。


```
$ sls --version
1.11.0
```

## 標準リソースの上書き機能を使う

そのまんまのサンプルが案内されていました。

[serverless/resources.md | serverless/serverless](https://github.com/serverless/serverless/blob/master/docs/providers/aws/guide/resources.md#override-aws-cloudformation-resource)


> You can override the specific CloudFormation resource to apply your own options. For example, if you want to set AWS::Logs::LogGroup retention time to 30 days, override it with above table's Name Template.


なるほど、標準でつくられるLogGroup(`/aws/lambda/{service_name}-{stage}-{function_name}`)の定義を上書きできるんだ。


上書き用名前指定の書式は、`{normalizedFunctionName}` + `LogGroup`となっている。

例えば次の例,function名を`main`としている場合、`MainLogGroup`に`RetentionInDays`を付与してあげればよい。

```serverless.yml
---
service: myservice

# ...

functions:
  main:
    handler: lambda_function.lambda_handler

```


```
resources:
  Resources:
    MainLogGroup:
      Properties:
        RetentionInDays: "30"
```

`--noDeploy(-n)`(1.12.x以降は`sls package`)でCFnテンプレートだけ作ってみると、ちゃんとそのようになっていることが確認できる。


```
  "Resources": {
    "MainLogGroup": {
      "Type": "AWS::Logs::LogGroup",
      "Properties": {
        "LogGroupName": "/aws/lambda/myservice-development-main",
        "RetentionInDays": "30"
      }
    },
```

function名に`-`や`_`が入っている場合、それぞれ予約文字列の`Dash`、`Underscore`を使えばよいとのこと。
名前自体が`Dash`だったりするケースは調べていません。


### 環境(stage)によって期限を変更したい場合

例えば`custom`属性を使って、stage名で引っ張ってくれば使い分けができますね。


```serverless.yml
custom:
  logRetentionInDays:
    development: "14"  # stage[development]は14日
    production: "90"   # stage[production]は90日
    default: "3"       # stageが見つからなかったらこれにfallbackするために設定

resources:
  Resources:
    MainLogGroup:
      Properties:
        RetentionInDays: ${self:custom.logRetentionInDays.${opt:stage, self:provider.stage}, self:custom.logRetentionInDays.default}
```

デフォルトにfallbackするかどうかはお好みかな。
