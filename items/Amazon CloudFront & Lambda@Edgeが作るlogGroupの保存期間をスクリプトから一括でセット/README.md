Lambda@Edgeでは実行ログの出力先が特殊で、CloudFrontが処理したエッジリージョンのCloudWatch Logsをつかいます。
デフォルトの保存期間がNever Expiresで、かつ初回アクセス発生時に作られるので、あとから保存期間をスクリプトで一括セットしたい。


## コード

logGroupNameのprefixはどのリージョンでも`/aws/lambda/us-east-1/{function_name}`になるのに注意ですかね。


```python:set_log_retention_days.py
#!/usr/bin/env python
import boto3


ec2_client = boto3.client('ec2')
all_region_items = ec2_client.describe_regions()
all_regions = [x['RegionName'] for x in all_region_items['Regions']]

policies = [
    {
        'prefix': '/aws/lambda/us-east-1.myservice-production-',
        'days': 90
    },
    {
        'prefix': '/aws/lambda/us-east-1.myservice-development-',
        'days': 14
    },
]

for pol in policies:
    for r in all_regions:
        print(r)
        logs_client = boto3.client('logs', region_name=r)
        log_groups = logs_client.describe_log_groups(logGroupNamePrefix=pol['prefix'])
        for log_group in log_groups['logGroups']:
            print(log_group['logGroupName'])
            result = logs_client.put_retention_policy(
                logGroupName=log_group['logGroupName'],
                retentionInDays=pol['days']
            )
            print(result['ResponseMetadata']['HTTPStatusCode'])
```

## 実行してみた

リリースしてしばらくしたら実行。現状どうなってるかは気にしなくて良いタイプ。

```shell-session
$ ./scripts/set_log_retention_days.py 
ap-south-1
eu-west-3
eu-west-2
eu-west-1
ap-northeast-2
/aws/lambda/us-east-1.myservice-production-origin_request
200
/aws/lambda/us-east-1.myservice-production-origin_response
200
/aws/lambda/us-east-1.myservice-production-viewer_request
200
ap-northeast-1
/aws/lambda/us-east-1.myservice-production-origin_request
200

...

```



先立って作っておく、でも良いのかもですが、リージョン自体が増えたりするのでスクリプトにしておくほうがよいかなと。
