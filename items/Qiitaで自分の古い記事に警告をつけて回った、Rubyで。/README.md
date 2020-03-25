
書きやすいのでホイホイQiitaに色々記事を書いてるわけですが、たまにやたら古い記事をストックされることがあります。
古いだけで今でも通用する内容なら良いんだけど、ライブラリのバージョンとか変わりすぎてて、そのままでは全く動かない記事もある。


動かないとはっきりわかっている記事がストックされたら、手動で警告をいれたりもしています。


![スクリーンショット_4_8_15__5_05_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d421bfb1-cb8e-6677-53de-01a5dde8488e.png "スクリーンショット_4_8_15__5_05_PM.png")


ただ、毎度警告を書くわけにもいかない。とりあえず古い記事には **これ古いのよ** と適当につけてみました。

[この記事用のリポジトリ： sawanoboly/qiita-articles-best-before](https://github.com/sawanoboly/qiita-articles-best-before)

## Rakeで全部の記事に警告をつける

またRakeでベタ書きか。はい。
最終更新日を基準に、365日くらい経過している記事には先頭に固定の文字列を入れることにした。

```ruby:Rakefile
# coding: utf-8
require 'qiita'
require 'formatador'
require 'logger'

BEST_BEFORE="<!-- too_old -->\n> **この記事は最終更新から1年以上経過しています。** 気をつけてね。\n"
@client = Qiita::Client.new(access_token: ENV['QIITA_TOKEN'])
@logger = Logger.new($stdout)

desc "show all items"
task :show do
  all_items = return_all_items
  # ["rendered_body", "body", "coediting", "created_at", "id", "private", "tags", "title", "updated_at", "url", "user"]
  Formatador.display_table(all_items, ['created_at', 'updated_at', 'since_last_update', 'tagged', 'title', 'url'])
end

## PATCH
# {
#   "body": "# Example",
#   "coediting": false,
#   "private": false,
#   "tags": [
#     {
#       "name": "example",
#       "versions": [
#         "0.0.1"
#       ]
#     }
#   ],
#   "title": "hello world"
# }

task :preview do
  all_items = return_all_items
  tester = all_items[-4]
  item = set_header_to_item(tester)
  patch_header_to_item(item)
end

desc "put warning header to all items"
task :all do
  all_items = return_all_items
  all_items.each do |item|
    updater = set_header_to_item(item)
    @logger.info updater["id"] + " will be update." if updater
    patch_header_to_item(updater)
  end
end

def return_all_items
  all_items = []
  page = 1

  items = @client.list_user_items(ENV['QIITA_USER'], page: page)

  while items.next_page_url do
    all_items.concat(items.body)
    page += 1
    items = @client.list_user_items(ENV['QIITA_USER'], page: page)
  end

  all_items.concat(items.body)

  # Time.strptime('2015-04-03T15:38:50+09:00', '%Y-%m-%dT%H:%M:%S')
  all_items.map do |i|
    ## 最終更新日からの経過日数を一旦計算していれておく
    i['since_last_update']  = ( (Time.now - Time.strptime(i['updated_at'], '%Y-%m-%dT%H:%M:%S')) / 60 / 60 / 24 ).to_i
    ## 固定の文字列があるかのチェックフラグを作っておく
    i['tagged'] = i['body'].start_with?(BEST_BEFORE)
    i
  end
end

def set_header_to_item(item)
  @logger.info item["id"] + ": " + item["since_last_update"].to_s + " days."
  return false if item["tagged"]
  return false if item["since_last_update"] < 365
  item["body"] = BEST_BEFORE +  item["body"]
  item
end

def patch_header_to_item(item)
  if item
    res = @client.update_item(item["id"], {
      body:       item["body"],
      coediting:  item["coediting"],
      private:    item["private"],
      tags:       item["tags"],
      title:      item["title"]
    })
    @logger.info res
  end
end
```

### 現状を把握する

さっきのRakeファイルをでは、`rake show`で経過日数と警告をつけているかの確認(2年目以降の対策)を確認できるようにしました。

```
$ rake show
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | created_at                | updated_at                | since_last_update | tagged | title                                                                       | url                                                    |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2015-04-03T15:16:06+09:00 | 2015-04-03T15:38:50+09:00 | 5                 |        | Shellで構築、Shellでテストする Test-Kitchen                                           | http://qiita.com/sawanoboly/items/2874018f385dbe8562b1 |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2015-04-01T10:09:05+09:00 | 2015-04-01T10:25:48+09:00 | 7                 |        | Loader.ioを使ってWordPressのバックエンド構成を変えつつ性能を測ってみる、php-fpmとHHVM                   | http://qiita.com/sawanoboly/items/e422b366439f67bfb01c |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2015-03-25T23:30:15+09:00 | 2015-03-26T17:20:09+09:00 | 12                |        | Dockerで自分のサイトを手軽にHTTP/2対応にしよっか                                              | http://qiita.com/sawanoboly/items/9b6b21430781fbc48370 |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2015-03-24T21:05:12+09:00 | 2015-03-25T15:04:00+09:00 | 14                |        | サーバー管理者のためのChef超入門（2） - @IT の補足、Cookbookのディレクトリ                             | http://qiita.com/sawanoboly/items/a2cbd4c42525ae894371 |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2015-03-06T18:34:39+09:00 | 2015-04-02T15:29:32+09:00 | 6                 |        | GithubのマイルストーンからCHANGELOGを生成する                                              | http://qiita.com/sawanoboly/items/abd46dafc18c59dde368 |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+

----

  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2012-07-12T16:28:15+09:00 | 2015-04-08T16:24:45+09:00 | 0                 | true   | HAProxyのtcplogをsyslog-ngでファイルに出力してfluentdで拾う                                | http://qiita.com/sawanoboly/items/fa69ca3f122719479c29 |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
  | 2012-07-11T17:30:55+09:00 | 2015-04-08T15:47:08+09:00 | 0                 | true   | naveでインストールしたnode.jsのアプリをstart-stop-daemonとmonitで運用する                       | http://qiita.com/sawanoboly/items/e1c40a9a3f30a8ee2334 |
  +---------------------------+---------------------------+-------------------+--------+-----------------------------------------------------------------------------+--------------------------------------------------------+
```


### 記事に警告をつける

`rake all`で全記事をなめておわり。

```
$ rake all
I, [2015-04-08 #6842]  INFO -- : 2874018f385dbe8562b1: 5 days.
I, [2015-04-08 #6842]  INFO -- : e422b366439f67bfb01c: 7 days.
I, [2015-04-08 #6842]  INFO -- : 9b6b21430781fbc48370: 12 days.
I, [2015-04-08 #6842]  INFO -- : a2cbd4c42525ae894371: 14 days.
I, [2015-04-08 #6842]  INFO -- : abd46dafc18c59dde368: 6 days.
I, [2015-04-08 #6842]  INFO -- : 741fdb59c14736841b24: 34 days.
I, [2015-04-08 #6842]  INFO -- : 7cf323e65a4757321553: 34 days.
I, [2015-04-08 #6842]  INFO -- : 8c512c7a35a90c6c6bff: 48 days.
I, [2015-04-08 #6842]  INFO -- : ac559c43b9662304931a: 65 days.
I, [2015-04-08 #6842]  INFO -- : b9d3ca4c6f0ef5544c92: 68 days.
I, [2015-04-08 #6842]  INFO -- : 7dc40a156db92ac96876: 88 days.
I, [2015-04-08 #6842]  INFO -- : 05c1131753f63448350e: 102 days.
I, [2015-04-08 #6842]  INFO -- : 76abd295b32123c4568d: 104 days.
I, [2015-04-08 #6842]  INFO -- : 28e98827bc044abdc32f: 92 days.
I, [2015-04-08 #6842]  INFO -- : 80c43625a06717fa1da3: 124 days.
I, [2015-04-08 #6842]  INFO -- : 498497383319ad200e3d: 132 days.
I, [2015-04-08 #6842]  INFO -- : 0b2afedd7949cd045e82: 140 days.
I, [2015-04-08 #6842]  INFO -- : d7a77901bc7685540d4d: 167 days.
I, [2015-04-08 #6842]  INFO -- : f5c72db552113c3f19a1: 175 days.
```

どれどれ。

![bashのfunctionを外部に置いてcurlで取り込む_-_Qiita.png](https://qiita-image-store.s3.amazonaws.com/0/7454/f52e5b57-18fc-dcc9-5fef-c3cc14aeae34.png "bashのfunctionを外部に置いてcurlで取り込む_-_Qiita.png")



影響は更新履歴から確認。ミスってもここから戻せばいいしね！

![「HAProxyのtcplogをsyslog-ngでファイルに出力してfluentdで拾う」の編集履歴_-_Qiita.png](https://qiita-image-store.s3.amazonaws.com/0/7454/c109d088-f7b7-9624-1a0d-1a6e77b05e23.png "「HAProxyのtcplogをsyslog-ngでファイルに出力してfluentdで拾う」の編集履歴_-_Qiita.png")

## さいごに

タスクを作っている間(特に記事取得の)APIをガンガン叩いたけど、終始レスポンスは快適。やるなQiitaと思いました。

なお、そのうちこの記事にも警告が張られるでしょう。
