<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

ドキュメント作成に色々とよいSphinx、さてhtmlを作ったら何処でホスティングしましょうかね。

Sphinx詳細は[公式:http://sphinx-doc.org](http://sphinx-doc.org)

適当なレンタルサーバやs3やら、Giithub Pages等ありますが、heroku上でsinatraを使って配信する方法を紹介。

## sinatraの準備

`Gemfile`にsinatraを入れましょう、rackupにはthinを使ってみます。

```ruby:Gemfile
source "https://rubygems.org"

gem "sinatra"
gem "rake"
gem "thin"
```

で、`config.ru`に全部かいちゃいました。

```ruby:config.ru
require 'sinatra'
set :public_folder, File.dirname(__FILE__) + '/build/html'

get '/' do
  redirect "/index.html", 301
end

run Sinatra::Application
```

`:public_folder`をSphinxがhtmlを生成するディレクトリにすれば大体OKです、`:public_folder`に実ファイルがあればそれを配信するsinatraの機能に丸投げです。

ついでに`/`へのリクエストは`/index.html`に放り投げます。`DirctoryIndex`みたいな挙動にしようと。


### heroku上のrackup用

無くても良かったような気がしつつ、Foreman用の`Procfile`も設置。

```Procfile 
web: bundle exec rackup -p $PORT
```

これで準備OKです。

### ローカルで動作確認

一旦ローカルで動作を確認してみます。

```shell:execute-rackup
$ bundle exec rackup
>> Thin web server (v1.5.1 codename Straight Razor)
>> Maximum connections set to 1024
>> Listening on 0.0.0.0:9292, CTRL+C to stop
```

`http://localhost:9292`を開けばSpinxが作成したindexを表示できていると思います、スタイルもリンクもOK。

## Basic認証をかけてみる

herokuでホストするのはいいけど、一応軽く認証を掛けたいこともあるでしょう。
Basic認証を追加してみました。

```ruby:config.ru(basic_auth)
require 'sinatra'
set :public_folder, File.dirname(__FILE__) + '/build/html'


## >>>追記部分
use Rack::Auth::Basic do |username, password|
  username == 'hogehoge' && password == 'mogemoge'
end

before '*' do
  # authenticate!
end
## <<<<追記部分

get '/' do
  redirect "/index.html", 301
end

get '/:dir' do  ## 修正２,3
  unless request.path_info.end_with?('.ico')
    redirect "#{request.path_info}.html", 301
  end
end

run Sinatra::Application
```

----

> 修正： `authenticate!`部分は勘違い、不要でした。
> 修正2： 自動生成元によっては.htmlがついてないことがあるので付与してリダイレクト
> 修正3： `favicon.ico`の自動取得でリダイレクトループするので条件設定。

----


`before`で全てのリクエストに対して`authenticate!`を通すことにしました、これだけでOK。


## herokuにPush

適当な手段でherokuにアプリケーションを登録します。

```shell:execute-rackup
$ git push heroku
Counting objects: 38, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (37/37), done.
Writing objects: 100% (37/37), 148.40 KiB, done.
Total 37 (delta 4), reused 0 (delta 0)

-----> Ruby/Rack app detected
-----> Using Ruby version: ruby-2.0.0

-- snip --

-----> Compiled slug size: 26.2MB
-----> Launching... done, v4
```

これで認証付きのドキュメントをホストできました。
あとはJenkinsとでも連携して、継続的ドキュメントアップロード提供をやるといいでしょう。

----

認証をGithub連携にしてみたバージョン: [ShpinxをShinatraでホストしつつGithubのOAuth経由でチーム認証をかけたが](http://qiita.com/sawanoboly@github/items/9646c4c53d442a7c6b46)
