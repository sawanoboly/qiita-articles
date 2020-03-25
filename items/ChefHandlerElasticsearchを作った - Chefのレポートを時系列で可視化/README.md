<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

前回 [OhaiのデータをElasticsearchに入れてKibanaで見る構成管理 - Qiita](http://qiita.com/sawanoboly/items/a8f9357f0f6044e7d7ff "OhaiのデータをElasticsearchに入れてKibanaで見る構成管理 - Qiita") に続いて、Chefの活動レポートをlogstashよろしく時系列で可視化してみます。

## ログでなくレポート？

Chefもログファイルを出すんだから、そもそもfluentdやlogstashにtailで拾わせればいいんじゃないかと思うかも。
しかし、Chefのログはだら流し系で、apache httpd アクセスログなどのように一行ごとに完結しないため、そのまま投げても役立てるのは難しいです。
その点レポートはイベント単位でひとまとまりなので、収集する意味は十分にあるでしょう。



## Chef::Handler::Elasticsearch

Cookbookはこちら。

[higanworks-cookbooks/chef_handler_elasticsearch](https://github.com/higanworks-cookbooks/chef_handler_elasticsearch "higanworks-cookbooks/chef_handler_elasticsearch")


以下デフォルトの挙動。

- index template `chef_handler_template`が無かったら作成する、対象は`chef_handler-*`
- logstash風にするため`@timestamp`を付与
- Chefの成功レポートは`/chef_handler-YYYY.MM.DD/success/ID` にPUT
- Chefの例外レポートは`/chef_handler-YYYY.MM.DD/failure/ID` にPUT
- PUTのタイムアウトは3秒

IDはEnterprise Chefのため用意されているっぽい、Chef実行毎に生成されるユニークIDを使っています(DEBUGログで確認可能)。
このハンドラでは通常のログにも出力しているので、イベントと紐付けて参照できます。

ハンドラ部分のソース(v1.0.0時点)はこんな感じです。

```ruby:libraries/default.rb
# Cookbook Name:: chef_handler_elasticsearch
# Library:: default
#
# Copyright 2014, HiganWorks LLC.
#

class Chef::Handler::Elasticsearch < ::Chef::Handler
  require 'timeout'
  attr_reader :opts, :config

  def initialize(opts = {})
    @config = {}
    @default = {
      url: 'http://localhost:9200',
      timeout: 3,
      prefix: 'chef_handler',
      prepare_template: true,
      template_order: 10,
      index_use_utc: true,
      index_date_format: "%Y.%m.%d",
      mappings: default_mapping
    }
    @opts = opts
    @opts
  end

  def default_mapping
'{
"_default_" : {
"numeric_detection" : true,
"dynamic_date_formats" : ["yyyy-MM-dd HH:mm:ss Z", "date_optional_time"]
}
}'
  end

  def report
    @default.merge!(node[:chef_handler_elasticsearch].symbolize_keys) if node[:chef_handler_elasticsearch]
    @config= @default.merge(@opts)
    Chef::Log.debug @config.to_s

    client = Chef::HTTP.new(@config[:url])
    index = "/#{@config[:prefix]}-#{build_logstash_date(data)}"

    if exception
      type = 'failure'
    else
      type = 'success'
    end

    prepare_template(client) if @config[:prepare_template]

    body = data.merge({'@timestamp' => Time.at(data[:end_time]).to_datetime.to_s})

    Chef::Log.debug "===== Puts to es following..."
    Chef::Log.debug body.to_s

    begin
      res = timeout(@config[:timeout]) {
        client.put([index, type, Chef::RequestID.instance.request_id].join('/'), body.to_json)
      }
      Chef::Log.debug "===== Response from es following..."
      Chef::Log.debug res.to_s
      Chef::Log.info "== Chef::Handler::Elasticsearch request_id: #{JSON.parse(res)['_id']}"
    rescue => e
      Chef::Log.error "== #{e.class}: Status report could not put to Elasticsearch."
    end
  end

  def build_logstash_date(data)
    if config[:index_use_utc]
      Time.at(data[:end_time]).getutc.strftime(@config[:index_date_format])
    else
      Time.at(data[:end_time]).strftime(@config[:index_date_format])
    end
  end

  def prepare_template(client)
    begin
      res = timeout(@config[:timeout]) {
        client.get("/_template/#{@config[:prefix]}_template")
      }
    rescue Net::HTTPServerException
      put_template(client)
      return
    rescue => e
      Chef::Log.error "== #{e.class}: Status report could not put to Elasticsearch."
      raise e.class, e.message
    end

    unless JSON.parse(@config[:mappings]) == JSON.parse(res)["#{@config[:prefix]}_template"]["mappings"]
      put_template(client)
    end
  end

  def put_template(client)
    begin
      Chef::Log.info "== create mapping template to Elasticsearch."
      res = timeout(@config[:timeout]) {
        client.put("/_template/#{@config[:prefix]}_template", build_template_body)
      }
    rescue => e
      Chef::Log.warn "== #{e.class}: mapping template could not put to Elasticsearch. Exiting..."
      raise e.class, e.message
    end
  end

  def build_template_body
    body = Hash.new
    body["template"] = "#{@config[:prefix]}-*"
    body["order"] = @config[:template_order]
    body["mappings"] = JSON.parse(@config[:mappings])
    Chef::Log.debug "===== Template for index following..."
    Chef::Log.debug body.to_json
    body.to_json
  end
end
```


### 使い方

次のいずれかで使えます。設定変更はattributeまたは初期化時にどうぞ。

####  run_listに`recipe[chef_handler_elasticsearch::default]`を入れる

この場合、attributeで設定をoverrideします。


#### `Chef::Config[:report_handlers]`,`Chef::Config[:exception_handlers]`に自前で追加する

自前のCookbookで、recipeかLibrariyで次のようにします。
metadataに`depends 'chef_handler_elasticsearch'` を入れる必要があります。

`Chef::Config[:report_handlers] << Chef::Handler::Elasticsearch.new`

デフォルトの設定を上書きするには、初期化時にオプションで渡すか、

``````ruby:libraries/default.rb
Chef::Config[:report_handlers] << Chef::Handler::Elasticsearch.new(
  url: 'http://test.example.com:9200',
  timeout: 10,
)
```

自前のCookbookやRoleなどで優先度の高いattributeを使いましょう。

```ruby:attributes/default.rb
override[:chef_handler_elasticsearch][:url] = 'http://test.example.com:9200'
override[:chef_handler_elasticsearch][:timeout] = 10
```

#### solo.rbやclient.rbで指定する。

Cookbookごと環境に入れたり、余分なAttributeが増えるのを嫌う場合はハンドラのソースを適当な場所にコピーしてrequireします。


```ruby:
require 'path_to_lib/chef_handler_elasticsearch.rb'

Chef::Config[:report_handlers] << Chef::Handler::Elasticsearch.new(
  url: 'http://test.example.com:9200',
  timeout: 10,
)
```

### ハンドラ実行

ハンドラの追加が上手く言っていれば、Elasticsearchにレポートを投げるようになっています。
初回のみテンプレートを作成します。

```
[2014-05-22T03:30:15+00:00] INFO: Chef Run complete in 0.060295361 seconds  

Running handlers:       
[2014-05-22T03:30:15+00:00] INFO: Running report handlers       
[2014-05-22T03:30:15+00:00] INFO: HTTP Request Returned 404 Not Found:        
[2014-05-22T03:30:15+00:00] INFO: == create mapping template to Elasticsearch.       
[2014-05-22T03:30:19+00:00] INFO: == Chef::Handler::Elasticsearch request_id: 9368d16f-1f09-487e-9701-8acb001737fc       
  - Chef::Handler::Elasticsearch       
Running handlers complete       

```


## Kibanaでの表示調整

とりあえず標準のLogstash Dashboardで表示してみましょう。

Index Settingsから、`logstash`の文字列を`chef_handler`に変更します。`_all`でもOK。
インデックス名はハンドラの初期化時に指定できるので任意のものがつけられます。


![スクリーンショット_5_22_14__11_48_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/807d79eb-c07a-2d2f-ec8e-6dd8ddb3a5ec.png "スクリーンショット_5_22_14__11_48_AM.png")

さて、よくみる時系列データになりましたね。
ただ、デフォルトではsuccessもfailureも一緒くたなので、ひとつ調製例を。

![スクリーンショット_5_22_14__12_34_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/5f7e7443-92d5-555d-d66f-61a729f3b1ca.png "スクリーンショット_5_22_14__12_34_PM.png")


試しにクエリを次の2つにわけてみました。

- `_type:success`
- `_type:failure`


![スクリーンショット_5_22_14__12_36_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d9d6b5f4-418c-e4ef-028d-c59a898b6e90.png "スクリーンショット_5_22_14__12_36_PM.png")

色分けでスッキリ。


## 課題

index templateには改良の余地が結構ありそう。
