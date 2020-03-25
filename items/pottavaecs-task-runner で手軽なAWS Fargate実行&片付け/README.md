この記事は[Mobingi Advent Calendar 2018](https://qiita.com/advent-calendar/2018/mobingi)の24日目の記事(遅刻)です。

## 動機・背景など

AWSのVPCで任意のタスクを実行したい
=> タスクの定義はバージョン管理したい、定義変更時のみ実行したい
=> CPUリソースの利用はタスク実行時のみにしたい

のようなとき、選択肢が結構限られるなあと思いました。

そんな中で使えるひとつ、AWS Fargateで同期っぽくカジュアルにタスクを実行できるツール `ecs-task-runner` というのがありまして。

- [pottava/ecs-task-runner: A synchronous task runner for AWS Fargate on Amazon ECS](https://github.com/pottava/ecs-task-runner)
  - ※ `ecs-task-runner` は同名の別物もあるので注意

ちょっと使ってみましょう。

## pottava/ecs-task-runner のインストール

Go言語で書かれたツールで、ビルド済みのバイナリが[リリース](https://github.com/pottava/ecs-task-runner/releases)に置いてあります。

[リリース](https://github.com/pottava/ecs-task-runner/releases)から普通にダウンロードしてもよいですし、この手のツールには[ghg](https://github.com/Songmu/ghg) が便利なので、とりあえず手元から試すにはghgでいれてしまうのもよいでしょう。

- [Songmu/ghg: Get the executable from github releases easily](https://github.com/Songmu/ghg)

```
$ ghg get pottava/ecs-task-runner
fetch the GitHub release for pottava/ecs-task-runner
install pottava/ecs-task-runner version: 2.1
download https://github.com/pottava/ecs-task-runner/releases/download/2.1/ecs-task-runner_darwin_amd64

...

done!

$ ecs-task-runner --version
2.1-92d36da (built at 2018-11-01)
```

## pottava/ecs-task-runner がやること

実行するAWS環境の認証情報は`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`を使用します。その他必要なものは、とりあえずなしです。

> ロール/プロファイルのことをスルーしているのは、現行の使われ方としてAWS外での実行が多いんだろうかなと。

AWS Fargateの利用にはECSクラスタ等が必要なのでは。。と思いつつ実行してみると、いきなりいけます。

```
$ ecs-task-runner run boxfuse/flyway --command="-v"
{
  "container-1": [
    "2018-12-25T18:01:46+09:00: Flyway Community Edition 5.2.4 by Boxfuse"
  ]
}
```

この環境には設定済みのECSクラスタは無いんですが。。

ではデフォルトのecs-task-runnerが何をしたのか、CloudTrailのeventから`ecs-task-runner run`実行後のAPIコールの一覧を抜粋してみます。

```
DescribeSecurityGroups, ec2.amazonaws.com
DescribeVpcs, ec2.amazonaws.com
DescribeSubnets, ec2.amazonaws.com
CreateCluster, ec2.amazonaws.com
DescribeClusters, ec2.amazonaws.com
RegisterTaskDefinition, ec2.amazonaws.com
CreateLogGroup, ec2.amazonaws.com
GetRole, ec2.amazonaws.com
RunTask, ec2.amazonaws.com
DescribeTasks, ec2.amazonaws.com

... (しばらくDescribeTasks)

DescribeTasks, ec2.amazonaws.com
DeleteLogGroup, ec2.amazonaws.com
DeregisterTaskDefinition, ec2.amazonaws.com
DescribeNetworkInterfaces, ec2.amazonaws.com
DeleteCluster, ec2.amazonaws.com
```

全部つくるし、全部片付けてるね！ 実行結果(STDOUT/STDERR)なんかはCloudWatch Logsから取り出して、その後LogGroupをしれっと消しています。

これらはもちろん使いたいリソースを指定できるので、任意のVPCとかサブネットとかを使ったり、Entrypoint, Commandを指定してコンテナを実行したりもできます。


## pottava/ecs-task-runner のよいとこ

全体的に、手ぶらでふらっと寄れる感がよい。それでいて持ち込みもOK的な。

- 実行に不足しているリソースは都度調達＆廃棄してくれる
- リソースを指定すればそれを使ってくれる
  - 守備範囲外のものは勝手に作ったりはしないあたりも弁え感
- 設定が親切
  - なんでも環境変数で設定できる
  - 実行時オプションで上書きが可能
- コンテナのexit_statusが取れてるっぽい
  - 一部例外あり..? (※ entrypointがそもそも嘘、とかの普通は無いケース)
- 非同期実行形式ならポートListenするサーバーも起動/停止できる
  - README例でのHTTPサーバとか
  - ツール入り踏み台とかにも？ [AWS Fargateのタスクでsshd、コンテナにシェルで入ってみる - Qiita](https://qiita.com/sawanoboly/items/2766c8b0760ad9f9be99)

その他、`--extended-output` オプションをつけて実行すると実行の様子をレポートしてくれて、色々と参考になります。


## 雑記

そもそも今回、実際何をしようとしていたのかというと、Amazon RDS上のインスタンスが持つDBのスキーマをflywayで管理したいという要件が発端でした。

- [Flyway by Boxfuse • Database Migrations Made Easy.](https://flywaydb.org/)
  - [flyway/flyway-docker: Official Flyway Docker images](https://github.com/flyway/flyway-docker)

ジョブを実行できそうなお手軽系のリソースとしては、Lambda, CodeBuildなどが使えそうですが、これらはVPCで立ち上げると外部へのアクセスにNAT GW/インスタンスが必要です。
今回はALL AWSのつもり、段取りを次のように考えていきました。

- CodeCommitにRDBのテーブル定義(SQL文)をpush
- CodeBuildでテストして...
  - CodeBuild上で直接[Flyway](https://flywaydb.org/)? <= 対象がVPCだと、外部アクセスにNAT必須なのが地味にめんどい
  - CodeDeployにつなげてFargateへ <= コンテナ実行が正常終了で停止するケースがイマイチスコープ外っぽい
- CodeBuildでイメージ作成して...
  - ECR push
  - そのままecs-task-runnerへ <= failも分かるし
- どっちにしろ実行後の現状スキーマをartifactにダンプしておくとよさそう

すでに使い回せるNATゲートウェイがある環境ならば、CodeBuildから直接Flywayでも良さそうに思えてきますね。
その場合、外部アクセスを全くしなくてよいようにFlywayバイナリごとリポジトリに入れてしまうなどの調整が必要で、テストとかに必要なツールも全部入れる必要があります。気軽に変更しづらくなりそうで、あまり好みではないかなあ。


ということで、CodeBuildからecs-task-runnerを叩く形に落ち着きそうなので、ついでにツール紹介してしまおうということにしました。

あとはこれができたら完璧なのかな・・？

- [タスクのプライベートレジストリの認証 - Amazon Elastic Container Service](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/private-auth.html)

など。
