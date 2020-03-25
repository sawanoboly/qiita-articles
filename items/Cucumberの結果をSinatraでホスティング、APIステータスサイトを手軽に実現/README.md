<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[githubのステータスサイト](https://status.github.com/)など、WebでAPIを提供するプロバイダならユーザから各機能が正常なのかをお知らせしたいですね。

何かの機能がそのふるまいをできているかどうか？ これはCucumberの出番ですかね。
最新結果がいつでも誰にでも分かるようになっていればベストですが、そのためのサーバを立てて管理して・・・となると面倒極まりない。

スケジューラもあることだし[Heroku](http://www.heroku.com/)にやってもらうか。

今回のサンプルはこちらにおいてます > [https://github.com/higanworks/heroku_cuucmber_example](https://github.com/higanworks/heroku_cuucmber_example)

## 概要

- cucumberのhtml出力を定期的に実行する
- 結果のhtmlをherokuでホスティングする


### Gemfileの作成

`Sinatra`、`Cucumber`など関連するGemsを入れるため`Gemfile`を作成します、Bundleしておきましょう。`Gemfile.lock`も忘れずにgitに入れておきます。


```ruby:Gemfile
source "https://rubygems.org"

gem "sinatra"
gem "cucumber"
gem "rake"
gem "thin"
```

## sinatraでindex.htmlを表示する

`/`が呼ばれたら`public/index.html`のファイルの内容をReadすればいいので、`cunfig.ru`にまるごと書いてしまえばいいです。

```ruby:config.ru
require 'sinatra'
get('/') { open('public/index.html').read }
run Sinatra::Application
```

必要十分。多言語対応ならjaとかenとかのクエリを貰って分岐しますわ。


## heroku用にForemanのファイルを用意する

herokuはForemanでアプリを起動してくれるので、foreman用の定義をつくります。


```text:Procfile
web: bundle exec rackup -p $PORT
```




## herokuへ

`features`を適当に書いたら、herokuへアップします。

```shell:Shell
heroku create
git push heroku master
```

### cucumberをheroku上で実行

出力を`public/index.html`にして更新します。

```shell:Shell
heroku run cucumber -f html -o public/index.html
```

これで`index.html`がcucumberの出力に差し替わります。


## とりあえずホスティングした結果

[http://mighty-springs-8010.herokuapp.com/](http://mighty-springs-8010.herokuapp.com/)

問題なく見れました。

![cucumber_on_heroku](https://dl.dropbox.com/u/3524956/quiita/Cucumber_heroku_image01.jpg)

あとは定期的に実行すればOKと。

## スケジューラの追加と登録

アプリにスケジューラアドオンを追加します。

```shell:Shell
heroku addons:add scheduler:standard
```

先程のcucumberをタスクに登録すれば定期的に`index.html`が更新されていきます。

## 終わりに

出力のフォーマットをカスタマイズして、ユーザへの自動情報提供ページがつくれますね。
ビヘイビアテストとユーザ告知を同時に実現できる、なかなか良い仕組みだと思います。
