
> こっちのほうがスマートなのでおすすめする。 > [CircleCI経由でElasticBeanstalkにデプロイする方法 - Qiita](http://qiita.com/mikamix/items/31849dc94a3e39575240 "CircleCI経由でElasticBeanstalkにデプロイする方法 - Qiita")
> ここの記事は参考程度に。

背景はさておき、PRをマージしたらさっさとElastic Beanstalkを更新しておいてもいいアプリもあるでしょう。
じゃあCircleCIからpushしておこう。

## circle.yml

circle.ymlはこんな感じ。
関連ツールのセットアップ＆キャッシュ、デプロイコマンドの組み合わせです。

```circle.yml
---
dependencies:
  pre:
    - ci/01_eb_command.sh
    - ci/02_credencial_for_eb.sh
  cache_directories:
    - ~/awstools
deployment:
  my_environment:
    branch: master
    commands:
      - git aws.push
```



## ebコマンドをインストールする (`01_eb_command.sh`)

> [更新]
> sudo pip install awsebcliで良かった模様。これはナイス。
> 参考： [CircleCI経由でElasticBeanstalkにデプロイする方法 - Qiita](http://qiita.com/mikamix/items/31849dc94a3e39575240 "CircleCI経由でElasticBeanstalkにデプロイする方法 - Qiita")

ツールのセットアップその、ebコマンド。良くある外部ツールの導入ですね。
Elastic Beanstalk的には`RepositorySetup`が大事。
このあたり、ビルドで毎回やることとキャッシュしておきたいこと、環境変数から取りたいことの住み分けで色々工夫の余地はあります。

```ci/01_eb_command.sh 
#!/usr/bin/env bash
set -ex

EB_VERSION="2.6.4"
EB_BASE="AWS-ElasticBeanstalk-CLI-${EB_VERSION}"
HOME=${HOME:-/home/ubuntu}

## botoがいるので入れておく
sudo pip install boto

mkdir -p ${HOME}/awstools
cd $HOME/awstools

## ebダウンロード済みならスキップ
if ! [ -d ${HOME}/awstools/${EB_BASE} ]; then
  wget https://s3.amazonaws.com/elasticbeanstalk/cli/${EB_BASE}.zip
  unzip ${EB_BASE}.zip
fi

## ebコマンドをパスが通っている場所にリンク
sudo ln -fs ${HOME}/awstools/${EB_BASE}/eb/linux/python2.7/eb ${HOME}/bin/

## gitのプラグインを`.git/AWSDevTools/`に入れる(eb init相当)
cd $HOME/${CIRCLE_PROJECT_REPONAME}
bash -ex ${HOME}/awstools/${EB_BASE}/AWSDevTools/Linux/AWSDevTools-RepositorySetup.sh
```

## eb用のcredentialsファイルを生成する (`02_credencial_for_eb.sh`)

CircleCIのプロジェクト設定にはAWSキーを入れるところがあり、`~/.aws/config`,`~/.aws/credentials`を自動で置いてくれる。
ですが、ebコマンドからは直接読めない(..はず)、環境変数にも入らない。

今回はありもの(`~/.aws/credentials`)を流用、整形することにしました。

```ci/02_credencial_for_eb.sh 
#!/usr/bin/env bash
set -ex

if [ -f ~/.aws/credentials ]; then
  sed -e 's/\[default\]//' -e 's/aws_access_key_id = /AWSAccessKeyId=/' -e 's/aws_secret_access_key = /AWSSecretKey=/' ~/.aws/credentials | sed ':loop; N; $!b loop; ;s/^\s*\n//' > ~/.aws/eb_credentials.txt
  chmod 0600 ~/.aws/eb_credentials.txt
fi
```

キー側を変更、余分な改行とスペースを消しています。


### 環境変数の`AWS_CREDENTIAL_FILE`をセットしておく

ebは環境変数の`AWS_CREDENTIAL_FILE`があればそのファイルからcredentialsを取り込むので、プロジェクトの設定に１つ追加しておきます。

```Environment_variables_on_CircleCI
AWS_CREDENTIAL_FILE=/home/ubuntu/.aws/eb_credentials.txt
```

## git aws.push

この場合、デフォルトではignoreされる`./.elasticbeanstalk/config`はGitリポジトリに含めています。

含めたくない場合でもブランチ名とEnvironmentの対応があればいいので、CircleCI上で生成してもかまいません。

参考: [Git ブランチを特定の環境にデプロイする - AWS Elastic Beanstalk](http://docs.aws.amazon.com/ja_jp/elasticbeanstalk/latest/dg/command-reference-branch-environment.html "Git ブランチを特定の環境にデプロイする - AWS Elastic Beanstalk")

これで任意のブランチがすぐEBにpushされるようになりました。

![circleci_eb_png.png](https://qiita-image-store.s3.amazonaws.com/0/7454/be989638-2116-2713-bdad-c50c343b29c4.png "circleci_eb_png.png")

