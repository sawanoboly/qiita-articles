<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

連携するWEBAPIを開発する際、SSL/TLS部分には自己署名か自己CAで署名した証明書ペアを使う。

外部リソースの取得にfaradayを使う場合、証明書ストアを操作してVerifyを通過させる。

```
> require 'faraday'
> con = Faraday.new(url: "https://example.com")
> con.get

Faraday::Error::ConnectionFailed: SSL_connect returned=1 errno=0 state=SSLv3 read server certificate B: certificate verify failed
from /Users/sawanoboriyu/.rvm/rubies/ruby-1.9.3-p194/lib/ruby/1.9.1/net/http.rb:799:in `connect'
```

無対策ではOSのストア(OSXのキーチェーン等)が使われて、`certificate verify failed`となる。

コネクション初期化時にCAのファイルを下記要領で指定すれば問題なく通信可能だ。

```
> con = Faraday.new(url: "https://ssl.example.com", ssl: {ca_file: "your_cacert.pem"})
> con.get
=> #<Faraday::Response:0x007ff66acd71d0
 @env=
  {:method=>:get,
   :body=>
    "<!DOCTYPE html>\n<html>…
-- snip --
```

HTTP関連のオプションはラップされているnet::httpにそのまま渡している。


証明書が複数必要なら一つのファイル内にすべて入れるか、ストアを使う。

```
> cert_store = OpenSSL::X509::Store.new
> cert_store.add_file("your_cacert.pem")
> con = Faraday.new(url: "https://ssl.example.com", ssl: {cert_store: cert_store})
> con.get
=> #<Faraday::Response:0x007ff66acd71d0
 @env=
  {:method=>:get,
   :body=>
    "<!DOCTYPE html>\n<html>…
-- snip --
```



クライアントの開発では、証明書の検証ができないという理由でVerifyを無効にすることがあるが、適当に構築したCAで発行した証明書をVerifyありで使っていく方がいい。
