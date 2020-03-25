[Google Safe Browsing](https://developers.google.com/safe-browsing/)ってありまして。
APIがあるのでRubyからきゃつらのルールで不正と判断しているドメイン(または実行ファイル)をチェックしてみようとしました。

## 事前準備

- rubygems `google-api-client` をインストール
- APIKEYを環境変数にいれておく


## Lookupのサンプルコード

google-api-clientってほとんど動的生成なので、普通に使い方をソースから見ようとするとサッパリわかりません。

動的に生成されたらこうなるよというコードがリポジトリに一緒に上がっているので、そちらをドキュメント代わりにチェックします。

- https://github.com/google/google-api-ruby-client/tree/master/generated/google/apis

[safebrowsing_v4はこのへん](https://github.com/google/google-api-ruby-client/tree/master/generated/google/apis/safebrowsing_v4)ですね。

Lookupする場合、`find_threat_matches`メソッドに`Google::Apis::SafebrowsingV4::FindThreatMatchesRequest`のオブジェクトを渡すだけです。


```ruby:safebrowsing_lookup.rb
require 'google/apis/safebrowsing_v4'
sb_service = Google::Apis::SafebrowsingV4::SafebrowsingService.new
sb_service.key = ENV['APIKEY']

# タイプの詳細は https://developers.google.com/safe-browsing/v4/reference/rest/ で
tr = Google::Apis::SafebrowsingV4::ThreatInfo.new(
  threat_types: ["MALWARE", "SOCIAL_ENGINEERING"],
  platform_types:    ["ANY_PLATFORM"],
  threat_entry_types: ["URL"],
  threat_entries: [
    {"url": "http://testsafebrowsing.appspot.com/apiv4/ANY_PLATFORM/MALWARE/URL/"}
  ]
)

# client_idはなるべく一意にしてねとのことですが、サンプルコードなので適当に
opts = Google::Apis::SafebrowsingV4::FindThreatMatchesRequest.new(
  client: Google::Apis::SafebrowsingV4::ClientInfo.new(
    client_id: "google-api-client",
    client_version: "0.19.8"
  ),
  threat_info: tr
)

res = sb_service.find_threat_matches(opts)

puts res.to_json
```

レスポンスのmatchesに何かしら入っていれば引っかかったということです。
ちなみに`threat_entries`の`url`は`http://〜`のようにスキーマを含めなくてもマッチします。

```shell:
$ ruby safebrowsing_lookup.rb
{
  "matches": [
    {
      "cacheDuration": "300s",
      "platformType": "ANY_PLATFORM",
      "threat": {
        "url": "http://testsafebrowsing.appspot.com/apiv4/ANY_PLATFORM/MALWARE/URL/"
      },
      "threatEntryType": "URL",
      "threatType": "MALWARE"
    }
  ]
}
```


## 余談

Google Safe Browsingの影響として身近なところでは、Let's Encryptでドメインの申請をする際、このAPIでマッチしたら所有確認の時点でお断りされるんですよ。
