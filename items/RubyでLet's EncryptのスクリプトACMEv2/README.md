前回ACMEv1でやったのはこちら。 [RubyでLet's Encryptのスクリプト - Qiita](https://qiita.com/sawanoboly/items/f23e73c613e9454076b8)

Let's EncryptでACMEv2が使えるようになり、rubyのクライアントもv2対応してしばらく経ちました。
一応v1で困ってはいなかったのでちゃんと見ていなかったが、いつまでもv1で置いておくのも気持ちが悪いのでv2の仕様を確認することにした。

v1との相違点なんかを確認しながら同じことをやってみよう。

-  https://github.com/unixcharles/acme-client

あくまでRubyのクライアントでv1=>v2を使う際の変更点などであることを事前にご了承で。

以下、Let's EncryptはLEと呼称します。ほか、本記事内でのv2は文脈によって次のうちのいずれかを指しますので、少々読みにくいかもしれませんが流れで判断してください。

- 仕様としてのACMEv2
- LEのACMEv2仕様API
- ruby acme-clientのv2


## 環境

- IDCFクラウドにDebian9のVM
  - aptでRuby, acme-client v2.0.1
- WebサーバとしてNginx
  - rootは /var/www/html;

pryで実行していきます。
  
## ステップ 秘密鍵

ここは同じですね。

```ruby:
> require 'openssl'
=> true
> private_key = OpenSSL::PKey::RSA.new(2048)
=> #<OpenSSL::PKey::RSA:0x00007fc6491e3540>
```

## ステップ ACMEv2のディレクトリ (※エンドポイント指定からちょっと変更)

v2用から、directoryのパス込みで指定するようになってます。
引数のキーも`directory`なので、変数名もdirectoryにしちゃおう。

```ruby:
directory = 'https://acme-staging-v02.api.letsencrypt.org/directory'
```

例によってACMEv2に対応しているAPIならなんでもよいです。

## ステップ ACMEクライアントの初期化

手順はあまり変わらず、`directory`指定となった。
オブジェクトの中身は結構変わったね。秘密鍵がjwkの一部として使われている。

```ruby:
> require 'acme-client'
=> true
> Acme::Client::VERSION
=> "2.0.1"

> client = Acme::Client.new(private_key: private_key, directory: directory)
=> #<Acme::Client:0x00007fc64a30b8e0
 @connection_options={},
 @directory=
  #<Acme::Client::Resources::Directory:0x00007fc64a30b250
   @connection_options={},
   @url=#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/directory>>,
 @jwk=#<Acme::Client::JWK::RSA:0x00007fc64a30b7f0 @private_key=#<OpenSSL::PKey::RSA:0x00007fc6491e3540>>,
 @kid=nil,
 @nonces=[]>
```

また、すでに鍵を登録済み(≒アカウントがある)であれば、後述のkidを使ってクライアントを作成することができるようだ。


## ステップ CAへの公開鍵登録(初回) 

`terms_of_service_agreed`オプションがついて、更に<del>形骸化</del>簡略化した。
のはまあいいとして、v2では登録した秘密鍵に対応するkidという識別情報がもらえるようになった。


```ruby:
> account = client.new_account(contact: 'mailto:sawanoboriyu+acme@example.com', terms_of_service_agreed: true)
=> #<Acme::Client::Resources::Account:0x00007fc64a2faea0
 @client=
  #<Acme::Client:0x00007fc64a30b8e0
   @connection_options={},
   @connections=
    ...
   @directory=
    #<Acme::Client::Resources::Directory:0x00007fc64a30b250
     @connection_options={},
     @directory=
      {:meta=>
        {"caaIdentities"=>["letsencrypt.org"],
         "termsOfService"=>"https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf",
         "website"=>"https://letsencrypt.org/docs/staging-environment/"},
       :new_nonce=>#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/acme/new-nonce>,
       :new_account=>#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/acme/new-acct>,
       :new_order=>#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/acme/new-order>,
       :revoke_certificate=>#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/acme/revoke-cert>,
       :key_change=>#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/acme/key-change>},
     @url=#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/directory>>,
   @jwk=#<Acme::Client::JWK::RSA:0x00007fc64a30b7f0 @private_key=#<OpenSSL::PKey::RSA:0x00007fc6491e3540>>,
   @kid="https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx",
   @nonces=["uX19iMR4YlWAdYiu-KMzPdlYDFzpoAB-4-ikAvJPGQg"]>,
 @contact=["mailto:sawanoboriyu+acme@example.com"],
 @status="valid",
 @term_of_service=nil,
 @url="https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx">
```

上記のレスポンスでkidが`.../acme/acct/xxxxxxxx` といった感じで含まれています。

ちなみに`mailto:`では`@example.com,@example.net`といったドメインはブラックリストで却下されます。(これもLEのACMEv2 APIからだと思う)

> invalid contact domain. Contact emails @example.com are forbidden

なにかしら有効なドメインをつかいましょう。こちらは秘密鍵と違って重複しても問題ないはず。

### kid?

登録済みのアカウントを使い回すには、kidを保存しておかないと駄目なの？というと、そうでもないです。

```ruby:
> account.kid
=> "https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx"
```

クライアント初期化時にオプションとして渡すことができて、いちおうなんとなくスムーズな感じですすめることができます。

```ruby:
> kid = account.kid
=> "https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx"

# kidを指定したクライアント作成
> client = Acme::Client.new(private_key: private_key, directory: directory, kid: kid)
=> #<Acme::Client:0x00007fc64aa9a840
 @connection_options={},
 @directory=#<Acme::Client::Resources::Directory:0x00007fc64aa9a5e8 @connection_options={}, @url=#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/directory>>,
 @jwk=#<Acme::Client::JWK::RSA:0x00007fc64aa9a7c8 @private_key=#<OpenSSL::PKey::RSA:0x00007fc6491e3540>>,
 @kid="https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx",
 @nonces=[]>
```

ただ、すでに登録済みの秘密鍵を使ったクライアントは、kidを省略してもLE側で勝手に突き合わせてくれます。

```ruby:
# 初期化直後は kid = nil
> client = Acme::Client.new(private_key: private_key, directory: directory)
=> #<Acme::Client:0x00007fc6491e9260
 @connection_options={},
 @directory=#<Acme::Client::Resources::Directory:0x00007fc6491e9008 @connection_options={}, @url=#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/directory>>,
 @jwk=#<Acme::Client::JWK::RSA:0x00007fc6491e91e8 @private_key=#<OpenSSL::PKey::RSA:0x00007fc6491e3540>>,
 @kid=nil,
 @nonces=[]>

# でも取りに行けば、値があることが確認できる。
> client.kid
# 取得にはすこし時間がかかる
=> "https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx"
```

この辺はkidを使っても良いし、v1のときのように気にせず先にすすめてよいように思います。

> kidの値が間違っている場合でも、クライアントは作成可能。このあとのnew_orderでハネられます。


##  ステップ registration(v1) .... なくなった。

アカウント作成時に約款同意を含めたので不要と。

## ステップ ドメインのチャレンジを申請する

v1ではclientからauthorizationを作っていたが、v2では一旦`Acme::Client::Resources::Order`のインスタンスを作るようになった。
※このあとの`example.com`は適当に読み替えで。

> 追記:
> v2では、 identifiersとして渡したドメインはLowercaseで処理されるようになりました。
> あとでつくるauthorizationで、domain属性の中身が ACME2.example.com => acmev2.example.com になる感じ。

```ruby:
> order = client.new_order(identifiers: ['acmev2.example.com'])
=> #<Acme::Client::Resources::Order:0x00007fc64a506a50
 @authorization_urls=["https://acme-staging-v02.api.letsencrypt.org/acme/authz/8Kd11K9K7_sw4MyyEW3q5137Vphs0OLWHToQrlaax2A"],
 @certificate_url=nil,
 @client=
  #<Acme::Client:0x00007fc6491e9260
   @connection_options={},
   @connections=
    ...
   @directory=
    #<Acme::Client::Resources::Directory:0x00007fc6491e9008
     @connection_options={},
     @directory=
      ...
     @url=#<URI::HTTPS https://acme-staging-v02.api.letsencrypt.org/directory>>,
   @jwk=#<Acme::Client::JWK::RSA:0x00007fc6491e91e8 @private_key=#<OpenSSL::PKey::RSA:0x00007fc6491e3540>>,
   @kid="https://acme-staging-v02.api.letsencrypt.org/acme/acct/xxxxxxxx",
   @nonces=["yt3TCMgt67czokGFRzoQAr5JSWvMq-APFc5Ygak6JGg"]>,
 @expires="2018-09-04T05:38:09.929580426Z",
 @finalize_url="https://acme-staging-v02.api.letsencrypt.org/acme/finalize/xxxxxxxx/6557233",
 @identifiers=[{"type"=>"dns", "value"=>"example.com"}],
 @status="pending",
 @url="https://acme-staging-v02.api.letsencrypt.org/acme/order/xxxxxxxx/6557233">
```

なんでOrderを1つかませるようにしたのかなと、`new_order`の実装をみてみる。
なるほど`not_before`, `not_after`などが使えるようになったから、一枚かますようになったのかな。

```ruby:
# Owner: Acme::Client
# Visibility: public
# Number of lines: 16

def new_order(identifiers:, not_before: nil, not_after: nil)
  payload = {}
  payload['identifiers'] = if identifiers.is_a?(Hash)
    identifiers
  else
    Array(identifiers).map do |identifier|
      { type: 'dns', value: identifier }
    end
  end
  payload['notBefore'] = not_before if not_before
  payload['notAfter'] = not_after if not_after

  response = post(endpoint_for(:new_order), payload: payload)
  arguments = attributes_from_order_response(response)
  Acme::Client::Resources::Order.new(self, **arguments)
end
```

authorizationは`order.authorizations.first`として

authorizations自体は要素が1つの配列だけど、他の方式とかへの対応のため拡張可能な仕様なんだろう。

```ruby:
> authorization = order.authorizations.first
=> #<Acme::Client::Resources::Authorization:0x00007fc64a9e28d0
 @challenges=
  [{"type"=>"tls-alpn-01",
    "status"=>"pending",
    "url"=>"https://acme-staging-v02.api.letsencrypt.org/acme/challenge/8Kd11K9K7_sw4MyyEW3q5137Vphs0OLWHToQrlaax2A/164428936",
    "token"=>"8xTyIIL4sip6nJlI_7gg9AMp_3uFByQXHoqIRP0gbqQ"},
   {"type"=>"http-01",
    "status"=>"pending",
    "url"=>"https://acme-staging-v02.api.letsencrypt.org/acme/challenge/8Kd11K9K7_sw4MyyEW3q5137Vphs0OLWHToQrlaax2A/164428937",
    "token"=>"_Nu_4S3OuYkaMG-Kq5DHrX05zp9tlx-jubnMg0-k1v4"},
   {"type"=>"dns-01",
    "status"=>"pending",
    "url"=>"https://acme-staging-v02.api.letsencrypt.org/acme/challenge/8Kd11K9K7_sw4MyyEW3q5137Vphs0OLWHToQrlaax2A/164428938",
    "token"=>"yxJQO-KqzLwG70wnV5gs6PCF7v0JrGJdhvEa5kRtzNk"}],
 @client=
   ...
 @domain="example.com",
 @expires="2018-09-04T05:38:09Z",
 @identifier={"type"=>"dns", "value"=>"acmev2.example.com"},
 @status="pending",
 @url="https://acme-staging-v02.api.letsencrypt.org/acme/authz/8Kd11K9K7_sw4MyyEW3q5137Vphs0OLWHToQrlaax2A",
 @wildcard=nil>
```

`tls-alpn-01`, `http-01`, `dns-01` の3方式が

tls-alpn-01(The TLS with Application Level Protocol Negotiation) については [https://tools.ietf.org/html/draft-ietf-acme-tls-alpn-01](https://tools.ietf.org/html/draft-ietf-acme-tls-alpn-01) で。


## ステップ http-01タイプでチャレンジする

とりあえずv1の記事と同じように`http-01`で。


```ruby:
> challenge = authorization.http
=> #<Acme::Client::Resources::Challenges::HTTP01:0x00007fc649a813c8
 @client=
  #<Acme::Client:0x00007fc6491e9260
   @connection_options={},
   @connections=
    ...
   @directory=
    ...
   @jwk=#<Acme::Client::JWK::RSA:0x00007fc6491e91e8 @private_key=#<OpenSSL::PKey::RSA:0x00007fc6491e3540>>,
   @kid="https://acme-staging-v02.api.letsencrypt.org/acme/acct/6820825",
   @nonces=[]>,
 @error=nil,
 @status="pending",
 @token="_Nu_4S3OuYkaMG-Kq5DHrX05zp9tlx-jubnMg0-k1v4",
 @url="https://acme-staging-v02.api.letsencrypt.org/acme/challenge/8Kd11K9K7_sw4MyyEW3q5137Vphs0OLWHToQrlaax2A/164428937">
```

challengeの中身を確認しよう。ここはv1と全く同じですね。

```ruby:
> challenge.content_type
=> "text/plain"

> challenge.file_content
=> "MVOFaIaalrqjB_570ZnCztuJgQcW9zxQxIu2YcNllU0.84USDpgw-Xi9j6lV8ucRAwUOVv3-Aos2bQmb-cgACkU"

> challenge.filename
=> ".well-known/acme-challenge/MVOFaIaalrqjB_570ZnCztuJgQcW9zxQxIu2YcNllU0"

> challenge.token
=> "MVOFaIaalrqjB_570ZnCztuJgQcW9zxQxIu2YcNllU0"
```

### チャレンジのレスポンスを設置しよう

ここもv1同様に。

```ruby:
> www_root = '/var/www/html'
=> "/var/www/html"

> FileUtils.mkdir_p( File.join( www_root , File.dirname( challenge.filename ) ) )
=> ["/var/www/html/.well-known/acme-challenge"]

> File.write( File.join( www_root, challenge.filename), challenge.file_content )
=> 87
```

ここで`Content-Type: text/plain`になるようにnginx.confの修正も忘れないように。 [前回記事参照](https://qiita.com/sawanoboly/items/f23e73c613e9454076b8)


## チャレンジ対応OKをLEに通知

`request_validation`はv1と一緒、でも状態の更新は`challenge.reload`に変更となってる。

```ruby:
> challenge.status
=> "pending"

> challenge.request_validation
=> true


> challenge.status
=> "pending"

> challenge.reload
=> true
> challenge.status
=> "valid"
```

HTTPのリクエストログにはこんな感じで来ています。

```
GET /.well-known/acme-challenge/MVOFaIaalrqjB_570ZnCztuJgQcW9zxQxIu2YcNllU0 HTTP/1.1" 200 87 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)"
```

> challengeが`invalid`の場合、今回は `challenge.error` にレスポンスがはいっています。
> こんな感じで
> => {"type"=>"urn:ietf:params:acme:error:unauthorized", "detail"=>"No TXT record found at _acme-challenge.test.example.com", "status"=>403}


## ステップ CSRを作成する

`Acme::Client::CertificateRequest`が自動で秘密鍵を作ってくれるのはv1同様だが、単体のときでも`common_name`として指定するようになった。。のかな？


```ruby:
> csr = Acme::Client::CertificateRequest.new(subject: { common_name: 'acmev2.example.com' })
=> #<Acme::Client::CertificateRequest:0x00007fc649ae51e8
 @common_name="example.com",
 @digest=#<OpenSSL::Digest::SHA256: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855>,
 @names=["example.com"],
 @private_key=#<OpenSSL::PKey::RSA:0x00007fc649ae5148>,
 @subject={"CN"=>"example.com"}>
```


## ステップ CSRから証明書発行を申請する (※非同期になった)

v1との違いとして、ここで`order`が再登場。

```ruby:
> order.finalize(csr: csr)
=> true

# 出来上がるまでは 'processing'
> order.status
=> "valid"
```

## ステップ 作られた証明書を確認しよう


acme-client v2では`order`にはいっている。

```ruby:
> order.certificate
=> "-----BEGIN CERTIFICATE-----\nMIIF9TCCBN2gAwIBAgITAPqK0JUd ....
```

[初版から訂正] この`order.certificate`は新規証明書＋中間証明書が含まれていて、v1でのfull_chainにあたる内容がはいっています。

先頭が今回作成した証明書、それ以降が中間証明書。

```ruby:
certs = order.certificate.split("\n\n")

> OpenSSL::X509::Certificate.new(certs[0])
=> #<OpenSSL::X509::Certificate
 subject=#<OpenSSL::X509::Name CN=acmev2.example.com>,
 issuer=#<OpenSSL::X509::Name CN=Fake LE Intermediate X1>,
 serial=#<OpenSSL::BN 21825307703253062656376784444942033656726194>,
 not_before=2018-08-28 05:23:50 UTC,
 not_after=2018-11-26 05:23:50 UTC>


> OpenSSL::X509::Certificate.new(certs[1])
=> #<OpenSSL::X509::Certificate
 subject=#<OpenSSL::X509::Name CN=Fake LE Intermediate X1>,
 issuer=#<OpenSSL::X509::Name CN=Fake LE Root X1>,
 serial=#<OpenSSL::BN 185931811205298764263482705882344673253>,
 not_before=2016-05-23 22:07:59 UTC,
 not_after=2036-05-23 22:07:59 UTC>
```

ペアの鍵は`csr`から持ってくる必要があると。

```ruby:
> csr.private_key.to_pem
=> "-----BEGIN RSA PRIVATE KEY-----\nM
```

## 通して実行

ここまでのスクリプトをまとめてみるとこんな感じ、大筋はv1と変わらず。

registrationがいらない分すこしだけ短くなったのと、cert作成が非同期になったのでwaitが一箇所増えた。

```ruby:acmev2.rb
require 'openssl'

private_key = OpenSSL::PKey::RSA.new(2048)
directory = 'https://acme-v02.api.letsencrypt.org/directory'

require 'acme-client'

client = Acme::Client.new(private_key: private_key, directory: directory)
account = client.new_account(contact: 'mailto:sawanoboriyu+acme@example.com', terms_of_service_agreed: true)
puts account.kid

order = client.new_order(identifiers: ['acmev2.example.com'])
authorization = order.authorizations.first

challenge = authorization.http

www_root = '/var/www/html'
FileUtils.mkdir_p( File.join( www_root , File.dirname( challenge.filename ) ) )
File.write( File.join( www_root, challenge.filename), challenge.file_content )

challenge.request_validation

while challenge.status == 'pending'
  print "."
  sleep 2
  challenge.reload
end
puts challenge.status
raise if challenge.status != 'valid'

csr = Acme::Client::CertificateRequest.new(subject: { common_name: 'acmev2.example.com' })
order.finalize(csr: csr)
sleep(1) while order.status == 'processing'

File.write("privkey.pem", csr.private_key.to_pem)
File.write("fullchain.pem", order.certificate)

certs = order.certificate.split("\n\n")

certs.each do |cert|
  new_cert = OpenSSL::X509::Certificate.new(cert)
  puts new_cert.subject.to_s
  puts new_cert.not_after
end
```

実行してみる。

```bash:
$ bundle exec ruby ./acmev2.rb
https://acme-v02.api.letsencrypt.org/acme/acct/xxxxxxxx
.valid
/CN=acmev2.example.com/serialNumber=xxxxxxxxxxxxxxxx
2018-11-26 06:21:39 UTC
/CN=h2ppy h2cker fake CA
2021-03-21 02:47:52 UTC
```


## Rubyのacme-client v1 -> v2 まとめ

あくまでRubyライブラリ視点なので、そのへん注意で。

- ACMEプロトコルがv1からv2になった
  - (そもそもこれに追随した各種変更でもある)
- registrationは要らなくなった
- アカウント(≒秘密鍵)の使い回しはkidもついでに保存しておく選択肢が増えた
  - 正しいkidを使えば、チャレンジの段取りが通しですこしだけ早くなるかもしれない
- 証明書申請に関する一連のやりとりは`Acme::Client::Resources::Order`で取りまとめることになった
- [訂正：そうでもなかった]<del>キーペア、中間証明書の取り出しがやや面倒になってる</del>
  - <del>これは簡単なローカルの関数でも対応できるし、せっかくだからメソッドを生やすためにPRするかもしれない</del>
- ドメインが内部的に全部lowercaseで扱われるようになった。


### 付録: wildcardもやっとく？

orderのidentifiersに`*.`で始まるドメイン名を使えばワイルドカード証明書も作れます。というかACMEv2の目玉よね。
折角だからやっときましょうか。

```ruby:
> order = client.new_order(identifiers: ['*.test.opsrockin.com'])
=> #<Acme::Client::Resources::Order:0x00007fc649b47848
...

> authorization = order.authorizations.first
=> #<Acme::Client::Resources::Authorization:0x00007fc64aa71170
...

# 例によってHTTP01はなし
> authorization.http01
=> nil


> challenge = authorization.dns01
=> #<Acme::Client::Resources::Challenges::DNS01:0x00007fc64a4fc5c8


> challenge.record_type
=> "TXT"

> challenge.record_name
=> "_acme-challenge"

> challenge.record_content
=> "yo1HkEkMu8q8mebJ4Sr5CY2Gjp6gtoybHNp6WdVKQyM"

# DNSレコードを作ってくる

> challenge.request_validation
=> true
> challenge.reload
=> true
> challenge.status
=> "valid"


> csr = Acme::Client::CertificateRequest.new(subject: { common_name: '*.test.example.com' })
=> #<Acme::Client::CertificateRequest:0x00007fc649a921a0

> order.finalize(csr: csr)
=> true


# できた
> OpenSSL::X509::Certificate.new(order.certificate).subject.to_s
=> "/CN=*.test.example.com"
```
