
今日Webで見た記事が面白かった。これらです。

- [AWSのCloudWatchで取得できるBillingの情報を毎日Slackに通知させて費用を常に把握する - さよならインターネット](http://blog.kenjiskywalker.org/blog/2015/04/20/aws-cloudwatch-billing-chatops/ "AWSのCloudWatchで取得できるBillingの情報を毎日Slackに通知させて費用を常に把握する - さよならインターネット")
- [AWS CLIの処理をAWS Data Pipelineで自動化する ｜ Developers.IO](http://dev.classmethod.jp/cloud/aws/automate-operation-of-aws-cli-by-aws-data-pipeline/ "AWS CLIの処理をAWS Data Pipelineで自動化する ｜ Developers.IO")

では合体させるぞ。

## AWS Data Pipeline?

今回は変な使い方なので、この使い方に必要なポイントを。

- 起動するインスタンスはAmazon Linux 2013.03でした。
    - t1.micro(変更可)が起動して、処理が済んだらTerminate.
    - スクリプトの事前確認はEC2で2013.03を起動しよう。
- InのS3バケット内のオブジェクトは、インスタンス内の`INPUT1_STAGING_DIR`などの環境変数にセットされたパスにおかれる。
    - S3に置いたスクリプトファイルをほぼ無策で読める。
- Outに指定したS3バケットはインスタンス内の`OUTPUT1_STAGING_DIR `などの環境変数にセットされたパスに出力すればアップロードしてくれる。
    - スクリプトファイルの出力をS3に保存できる。
- AWS-CLIのほか、ただのシェルも実行できる。
    - Outに何か出力しないとバリデーションエラー？ (細かくは検証していない)
    - stdoutに何か吐くのはNGなのかも

本来の想定用途にくらべると相当しょぼいのですが、そのへんはご了承ねがいます。


## RubyスクリプトをS3に置く

じゃあInのバケットに、 [AWSのCloudWatchで取得できるBillingの情報を毎日Slackに通知させて費用を常に把握する - さよならインターネット](http://blog.kenjiskywalker.org/blog/2015/04/20/aws-cloudwatch-billing-chatops/ "AWSのCloudWatchで取得できるBillingの情報を毎日Slackに通知させて費用を常に把握する - さよならインターネット") のRubyファイルおいたらよくね？と。

ちょい書き換えて。

```ruby:aws_billing_slackit.rb
# coding: utf-8
require 'date'

# 今日の日付
d = Time.now

# 昨日の 00:00:00 ~ 23:59:59 の間のデータを利用して
start_time = DateTime.new(d.year, d.month, d.day) - 1
end_time = DateTime.new(d.year, d.month, d.day, 23, 59, 59) - 1

# 一日分の値から Average をとって
period = '86400'

# profileがあったら指定(ローカルデバッグ用)
profile = ENV["AWS_PROFILE"] ? "--profile #{ENV['AWS_PROFILE']}" : ""


# CloudWatchの値を取得してきて
# Billingのデータを持ってくる
allnum = `aws cloudwatch --region us-east-1 get-metric-statistics \
           --namespace 'AWS/Billing' \
           --dimensions '{"Name":"Currency","Value":"USD"}' \
           --metric-name EstimatedCharges \
           --start-time #{start_time} \
           --end-time #{end_time} \
           --period #{period} --statistics 'Average' #{profile} \
           | jq '.Datapoints[].Average'`.chomp

## 転送量だけ
transnum = `aws cloudwatch --region us-east-1 get-metric-statistics \
           --namespace 'AWS/Billing' \
           --dimensions '[{"Name":"Currency","Value":"USD"},{"Name":"ServiceName","Value":"AWSDataTransfer"}]' \
           --metric-name EstimatedCharges \
           --start-time #{start_time} \
           --end-time #{end_time} \
           --period #{period} --statistics 'Average' #{profile} \
           | jq '.Datapoints[].Average'`.chomp

## EC2だけ
ec2num = `aws cloudwatch --region us-east-1 get-metric-statistics \
           --namespace 'AWS/Billing' \
           --dimensions '[{"Name":"Currency","Value":"USD"},{"Name":"ServiceName","Value":"AmazonEC2"}]' \
           --metric-name EstimatedCharges \
           --start-time #{start_time} \
           --end-time #{end_time} \
           --period #{period} --statistics 'Average' #{profile} \
           | jq '.Datapoints[].Average'`.chomp

## 丸ごとポスト
`curl -X POST --data-urlencode 'payload={
  "attachments": [
    {
      "fallback": "今月のAWSの利用費は累計で #{allnum} ドルです。",
      "pretext": "今月のAWSの利用費は.... ",
      "color": "good",
      "fields": [
        {
          "title": "全サービス合計",
          "value": "#{allnum}",
          "short":true
        },
        {
          "title": "転送量",
          "value": "#{transnum}",
          "short":true
        },
        {
          "title": "EC2",
          "value": "#{ec2num}",
          "short":true
        }
      ]
    }
  ]
}' #{ENV['SLACK_WEBHOOK']}`
```

### S3へ 

で、さっきのRubyスクリプトと、今回はaws-cli用のクレデンシャルをS3においてみた。

![s3.png](https://qiita-image-store.s3.amazonaws.com/0/7454/9ff0620d-95ea-205a-9123-3e01c2875f8a.png "s3.png")

あとでインスタンスロールを使えるようにしておこう。


## Pipelineに渡すShellスクリプト

つぎはこちらから。パイプラインの説明を読みましょう。[AWS CLIの処理をAWS Data Pipelineで自動化する ｜ Developers.IO](http://dev.classmethod.jp/cloud/aws/automate-operation-of-aws-cli-by-aws-data-pipeline/ "AWS CLIの処理をAWS Data Pipelineで自動化する ｜ Developers.IO")
で、AWS CLIの代わりに、その一個上にある`Getting Started using ShellCommandActivity`を使います。


スクリプトサンプルがこちら。実際には空行なしのコメント無しで貼り付けました。

```
sudo yum -y install jq ruby22 aws-cli >> ${OUTPUT1_STAGING_DIR}/output.txt

## キーのコンフィグ。Roleで代用も可
mkdir ~/.aws >> ${OUTPUT1_STAGING_DIR}/output.txt
cp ${INPUT1_STAGING_DIR}/config ~/.aws/ >> ${OUTPUT1_STAGING_DIR}/output.txt

## Ruby実行
SLACK_WEBHOOK='https://hooks.slack.com/services/WEBHOOKのURI' ruby2.2 ${INPUT1_STAGING_DIR}/aws_billing_slackit.rb >> ${OUTPUT1_STAGING_DIR}/output.txt
```

パイプライン作成の様子がこちら。

![pipe3.png](https://qiita-image-store.s3.amazonaws.com/0/7454/df1a601f-4a97-37ed-889b-66794fc975c8.png "pipe3.png")

これでOK、使い続けるならスケジュールも決めましょう。あとは見守ります。


### Slackに来た様子を

だいたい5分位後、Slackに。

![Slack.png](https://qiita-image-store.s3.amazonaws.com/0/7454/afb1230e-61d7-cd48-e556-413ae8cc589d.png "Slack.png")

きたきた。
Average
