<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Elasticsearch+Kibanaといえば、ログなどの時系列データ(logstash形式)を可視化する利用法が広く知られています。

今回は構成管理の用途にElasticsearchを使うため、[ohai](https://github.com/opscode/ohai)のデータを入れて、Kibanaで検索できるようにしてみます。


## 何故やるの

OhaiはChefがNode情報を収集する時に使用するライブラリですが、個別に利用する事ができます。
通常はChef-Serverに集約しますが、それ以外の環境でOhaiのデータ(Json)を使い捨てするのは勿体無いかもしれない。

Jsonをそのまま投げて良いElasticsearchに突っ込んでみたらどうなるか。


## OhaiからElasticsearchに投げる時の方針

手っ取り早さを優先で、次の方針でElasticsearchにドキュメントを登録します。

- Indexは`ohai`
- Typeは`node`
- IDはホスト名(もしくは一意に判別)
- ドキュメントは上書き更新

要は各ホストで`curl`を使い、こんな感じでOhaiデータをPUTします。

```shell:
ohai | curl -XPUT http://localhost:9200/ohai/node/`hostname -f` -d @-'
#=> ohai | curl -XPUT http://localhost:9200/ohai/node/santiago.example.com -d @-'
```

これだけでNodeのアトリビュートがサーチ可能になるって楽ちんですね。


## サンプル環境

Githubにサンプルを用意しました。

https://github.com/higanworks/ohai_and_elasticsearch_example

必要環境はこちらです

- java
- ruby, bundler
- python


Rakeタスクで次のことをやります。

1. elasticsearch 1.1.1のダウンロードとサブディレクトリへの展開
2. kibana 3.1のダウンロードとサブディレクトリへの展開
3. elasticsearchスタート(port: 9200)
4. サンプルOhaiデータをelasticsearchにポスト
5. kibana用Python SimpleHTTPServerスタート(port: 8000)

### セットアップ

Ohaiのモック作成に、[customink/fauxhai](https://github.com/customink/fauxhai "customink/fauxhai")を使います。
ダミーのOhaiデータを作ってくれます。これも単体で使用可能っちゃ可能。


```
$ bundle
Using excon 0.33.0
Using net-ssh 2.9.1
Using ipaddress 0.8.0
Using mime-types 1.25.1
Using mixlib-cli 1.5.0
Using mixlib-config 2.1.0
Using mixlib-log 1.6.0
Using rake 10.1.0
Using mixlib-shellout 1.4.0
Using coderay 1.1.0
Using systemu 2.5.2
Using yajl-ruby 1.2.0
Using method_source 0.8.2
Using slop 3.5.0
Using yard 0.8.7.4
Using webster 0.5.0
Using bundler 1.6.2
Using ohai 7.0.4
Using pry 0.9.12.6
Using pry-doc 0.6.0
Using fauxhai 2.1.2
Your bundle is complete!
Use `bundle show [gemname]` to see where a bundled gem is installed.
```


後は`rake up`。 

※`9200`ポートが既に使用中の場合、そちらを一旦止めるか、RakefileのPut部分の修正が必要です。


```
$ rake up
Prepare elasticsearch and kibana..
--2014-05-18 17:01:33--  https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.1.1.tar.gz

2014-05-18 17:01:54 (1.01 MB/s) - `elasticsearch-1.1.1.tar.gz' saved [19872305/19872305]

x elasticsearch-1.1.1/lib/lucene-codecs-4.7.2.jar
-- snip 
x elasticsearch-1.1.1/config/elasticsearch.yml
--2014-05-18 17:01:54--  https://download.elasticsearch.org/kibana/kibana/kibana-3.1.0.tar.gz

2014-05-18 17:01:58 (302 KB/s) - `kibana-3.1.0.tar.gz' saved [1073033/1073033]

x kibana-3.1.0/app/app.js
-- snip --
x kibana-3.1.0/README.md

Start elasticsearch !!
Waiting es available....
put 30 ohai mocks to elasticsearch
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/gaditan.example.com -d @-' => omnios
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/spiderly.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/martlet.example.com -d @-' => omnios
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/pyemia.example.com -d @-' => fedora
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/discantus.example.com -d @-' => fedora
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/yogasana.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/araneae.example.com -d @-' => amazon
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/tongueman.example.com -d @-' => 
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/sluice.example.com -d @-' => freebsd
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/cupping.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/bayou.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/suterbery.example.com -d @-' => redhat
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/muirfowl.example.com -d @-' => fedora
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/unreached.example.com -d @-' => ubuntu
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/bombastic.example.com -d @-' => 
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/sproutful.example.com -d @-' => windows
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/drubbly.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/proaction.example.com -d @-' => debian
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/langite.example.com -d @-' => amazon
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/arrowless.example.com -d @-' => ubuntu
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/cloisonne.example.com -d @-' => 
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/thiasoi.example.com -d @-' => ubuntu
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/lingy.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/ductor.example.com -d @-' => ubuntu
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/sandproof.example.com -d @-' => oracle
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/tasteless.example.com -d @-' => windows
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/uglifier.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/tsumebite.example.com -d @-' => suse
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/scienced.example.com -d @-' => redhat
Example: 'ohai | curl -XPUT http://localhost:9200/ohai/node/workship.example.com -d @-' => gentoo
Serving HTTP on 0.0.0.0 port 8000 ...
```

サンプルデータを登録して、PythonのWebサーバがあがりました。

### サンプルデータの閲覧

`http://localhost:8000`を開くとKibanaが上がっています。
既存データがあるので、Sample Dashboardをそのまま利用することができますね。

![kibana_01.png](https://qiita-image-store.s3.amazonaws.com/0/7454/aa829a29-3322-b611-cd69-417412fb7dd8.png "kibana_01.png")

レコード一覧が出てきたら、`_id`フィールドのみ表示してみます。

![kibana_02.png](https://qiita-image-store.s3.amazonaws.com/0/7454/ca96aa0b-9292-6f83-34e7-e4285091c4b7.png "kibana_02.png")

サンプル登録されたホストがそのまま一覧で確認できますね。

![kibana_03.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d81925da-70f0-4f36-18f3-e3e66b900cb3.png "kibana_03.png")


次は知りたいフィールドをチェクして、カラムを追加してみましょう。
今回はプラットフォームとそのバージョン、ついでにOhai実行に使ったRubyのバージョン(全てfauxhaiによるダミーデータ)を表示してみます。

![kibana_04.png](https://qiita-image-store.s3.amazonaws.com/0/7454/b52001c8-ffa5-9dfb-55ab-5b6d12421df6.png "kibana_04.png")


定期的に`Ohai + curl`を仕込むだけで、Nodeの現在の構成情報が取れるようになりますね。

クエリをちょいといじればこのようなグラフも。

![kibana_05.png](https://qiita-image-store.s3.amazonaws.com/0/7454/abfd61da-b0c0-a544-00a5-2ab8b9d492cc.png "kibana_05.png")


### 追記:Elasticsearchクエリの出力

グラフを作ったら`inspect`ボタンでクエリ用Jsonを出力できます。
これはNodeのプラットフォームを集計するクエリです。

```
curl -XGET 'http://localhost:9200/_all/_search?pretty' -d '{
  "facets": {
    "terms": {
      "terms": {
        "field": "platform",
        "size": 100,
        "order": "count",
        "exclude": []
      },
      "facet_filter": {
        "fquery": {
          "query": {
            "filtered": {
              "query": {
                "bool": {
                  "should": [
                    {
                      "query_string": {
                        "query": "*"
                      }
                    }
                  ]
                }
              },
              "filter": {
                "bool": {
                  "must": [
                    {
                      "match_all": {}
                    }
                  ]
                }
              }
            }
          }
        }
      }
    }
  },
  "size": 0
}'
```

サンプルNodeを少々多めに突っ込んでからcurlでクエリしてみましょう。

```shell:curl
$ curl -XGET 'http://localhost:9200/_all/_search?pretty' -d '{
>   "facets": {

-- snip --

> }'

{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 111,
    "max_score" : 0.0,
    "hits" : [ ]
  },
  "facets" : {
    "terms" : {
      "_type" : "terms",
      "missing" : 7,
      "total" : 104,
      "other" : 0,
      "terms" : [ {
        "term" : "suse",
        "count" : 17
      }, {
        "term" : "fedora",
        "count" : 11
      }, {
        "term" : "amazon",
        "count" : 11
      }, {
        "term" : "smartos",
        "count" : 9
      }, {
        "term" : "debian",
        "count" : 8
      }, {
        "term" : "omnios",
        "count" : 7
      }, {
        "term" : "centos",
        "count" : 7
      }, {
        "term" : "windows",
        "count" : 6
      }, {
        "term" : "openbsd",
        "count" : 6
      }, {
        "term" : "ubuntu",
        "count" : 5
      }, {
        "term" : "oracle",
        "count" : 5
      }, {
        "term" : "freebsd",
        "count" : 5
      }, {
        "term" : "gentoo",
        "count" : 4
      }, {
        "term" : "mac_os_x",
        "count" : 2
      }, {
        "term" : "redhat",
        "count" : 1
      } ]
    }
  }
}
```

使いやすいといえば使いやすい。


### サンプル停止

WebサーバはCtrl+Cで止めて、ElasticSearchを停止しておきます。

```
$ rake down
Stop elasticsearch...
```

これでOK。

## おわりに

Ohaiの詳しい解説は拙著[Chef活用ガイド](http://ascii.asciimw.jp/books/books/detail/978-4-04-891985-2.shtml "Chef活用ガイド")でもやっているので、収集したいデータを追加したいと思ったら参考になると思います。

次は時系列っぽいデータ、Chef-HandlerからReportをElasticsearchに突っ込む予定。
