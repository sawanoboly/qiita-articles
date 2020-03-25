<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[Travis-CI](https://travis-ci.org/)はビルド実行ツール。コードのリポジトリと連携して色々自動でやらせます。

さて、Travis-CIはGithubなどと連携して、リビジョン毎にBuildするのが一般的です。
でも定期的にBuild走らせるのもありかなと、herokuのスケジューラと組み合わせて叩いてみます。


## よそのBashをテストするスクリプト

とりあえずなんでもいいんですが、ChefのOmunibus InstallerのSyntaxチェックをするスクリプトでも作ってGithubに公開します。

https://github.com/OpsRockin/travis_bash_sample

```shell:test.sh
#!/usr/bin/env bash
curl -s -L https://www.getchef.com/chef/install.sh | bash -n && echo 'syntax OK'
```

要はリポジトリのリビジョンに関係ない外部の何かをチェックするタスクです。


`.travis.yml`はこんな感じ。デフォルトではrubyを用意してくれるけど気にしない。

```yaml:.travis.yml
script: "./test.sh"
```

これをTravis,ciに登録します。

https://travis-ci.org/OpsRockin/travis_bash_sample

しました。

## Travis-CIのRubyライブラリで最終のビルドをリスタートする

そしたらTravis-CIのAPIを叩く準備をします。

Rubyのライブラリがあったので使いましょう。

https://github.com/travis-ci/travis


ライブラリを使って、最新のビルドをリスタートするRakeタスクを作りました。

```ruby:Rakefile
require 'bundler/setup'
require 'travis'

desc 'call travis to build restart'
task :default do
  Travis.access_token = ENV['TRAVIS_TOKEN']
  build = Travis::Repository.find(ENV['TRAVIS_BUILD'])
  build.last_build.restart
end
```

`task :default do`から`end`までがリスタートの処理です。使いやすくていいですね。

このタスクを実行すると、最新のビルドがリスタートします。
UIから見てると分かりやすい。

![スクリーンショット_12_15_13__2_16_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/0093e70c-e47e-a31c-0542-46cdfe2738d2.png "スクリーンショット_12_15_13__2_16_PM.png")


ここのRakeタスクはherokuに叩かせるので、Gitリポジトリにします。ついでにGithubへもPush。

https://github.com/OpsRockin/travis-ci_scheduler


## herokuのスケジューラに継続的に実行させる

実行するタスクがデキたので、herokuにコードを持って行きましょう。


### 新規アプリの作成

herokuコマンドでちょいちょいとアプリケーションを登録しました。

```shell:
$ heroku apps:create travis-bash-sample
Creating travis-bash-sample... done, stack is cedar
http://travis-bash-sample.herokuapp.com/ | git@heroku.com:travis-bash-sample.git
```

ではRakefileをherokuに置きます。

```shell:
$ git remote add heroku git@heroku.com:travis-bash-sample.git
$ git push heroku
Initializing repository, done.
Counting objects: 6, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (5/5), done.
Writing objects: 100% (6/6), 1.12 KiB | 0 bytes/s, done.
Total 6 (delta 0), reused 0 (delta 0)

-----> Ruby app detected
-----> Compiling Ruby
# -- 以下略
```

はいご苦労さん。

じゃあ`heroku config`で必要な環境変数を登録して、rakeを実行してみましょう。

### Rakeタスクをherokuで実行(おためし)

```shell:
$ heroku config:add TRAVIS_TOKEN="YOUR_TOKEN"
$ heroku config:add TRAVIS_BUILD="OpsRockin/travis_bash_sample"
$ heroku run rake
Running `rake` attached to terminal... up, run.7639
```

Travis-CIのUIを眺めていると、Buildがリスタートしました。OKね。

![スクリーンショット_12_15_13__2_32_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/992f356a-fa65-657a-e717-fb0268011086.png "スクリーンショット_12_15_13__2_32_PM.png")


### herokuのスケジューラアドオンを作成

herokuはアドオンにスケジューラを持っているので、定期的にrakeコマンドを叩いてもらいます。

`heroku addons`でアプリにスケジューラを追加します。

```shell:
$ heroku addons:add scheduler:standard
Adding scheduler:standard on travis-bash-sample... done, v5 (free)
This add-on consumes dyno hours, which could impact your monthly bill. To learn more:
http://devcenter.heroku.com/addons_with_dyno_hour_usage
To manage scheduled jobs run:
heroku addons:open scheduler
Use `heroku addons:docs scheduler` to view documentation.
```

追加できたので、Jobの編集画面を開きます。


```shell:
$ heroku addons:open scheduler
Opening scheduler:standard for travis-bash-sample... done
```

ブラウザで開くので、実行するコマンドとスケジュールを登録。


![スクリーンショット_12_15_13__4_17_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/f94abfaf-5c57-9e62-fb34-8723453bee05.png "スクリーンショット_12_15_13__4_17_PM.png")

1時間に一回でいいかな。

これで指定したスケジュールでTravis-CIの最終ビルドが再実行されます。



### スケジューラの実行状況を確認する

スケジューラホンマに動いてんの？って思うならログを見てもいいです。

```
$ heroku logs --ps scheduler
2013-12-15T07:27:38.479645+00:00 heroku[scheduler.1943]: Starting process with command `bundle exec rake`
2013-12-15T07:27:44.896760+00:00 heroku[scheduler.1943]: Process exited with status 0
2013-12-15T07:27:44.916523+00:00 heroku[scheduler.1943]: State changed from up to complete
2013-12-15T07:27:39.142666+00:00 heroku[scheduler.1943]: State changed from starting to up
2013-12-15T07:36:32.726247+00:00 heroku[scheduler.8651]: Starting process with command `bundle exec rake`
2013-12-15T07:36:33.395186+00:00 heroku[scheduler.8651]: State changed from starting to up
2013-12-15T07:36:40.094286+00:00 heroku[scheduler.8651]: Process exited with status 0
2013-12-15T07:36:40.100911+00:00 heroku[scheduler.8651]: State changed from up to complete
```


やってるようだね。
ログの間隔が10分なのは撮影用にチェックしたからです。

## おわりに

herokuだけだと通知の仕組みがいる、Travis-CIだけだとスケジュール定期実行はしない。
なら、できることを殿様食いしましょう。

