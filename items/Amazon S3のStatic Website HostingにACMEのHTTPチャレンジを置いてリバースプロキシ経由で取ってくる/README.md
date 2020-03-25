
もう[Let's Encrypt](https://letsencrypt.jp/)もすっかり普及したね！

さて、証明書を発行するためには、いくつか用意された手段でドメインの所有を確認してもらう(チャレンジ)必要があります。
手段の中でも最もサクサクなのはHTTPかな、と思っています。

でも環境によってはコンテンツをいちいちHTTPサーバのローカルに置くのが面倒だ。証明書ペアだけ作って、一旦手元に欲しいケースもあるのだ。

そうだプロキシしよう。


## なんでS3に置いてみるの？

このHTTPチャレンジは基本的に使い捨てなので、次のように管理できるとまあまあ都合がよいのです。

- キーバリューストア的使い勝手で、出し入れ簡単
- コンテンツ掃除が面倒なので、寿命つき
    - そもそも一回取れればいいだけ


なかでもS3は、

- オブジェクトストレージ(キーバリュー)
    - トークンはまず被らないのでキーの重複も無い
- Lyfecycleを使えば最短1dayで勝手に消せるので掃除いらず
- たいてい稼働している。
- この用途なら月あたり数円かかる？ という感じ

という、なかなかマッチした環境。


s3が禁止されたら、`ngx_mruby + redis`ででもやってみるかなあ。
POSTしたらそのままGET(1回レスポンスしたら消す)できるだけのアプリを[sinatra](http://www.sinatrarb.com/)あたり書いたりでも良いけど。

## S3の仕込み


- バケットを`your-token-bucket`などで作成。
- static web hosting有効
- Lifecycle => delete 1day

最低限でこのくらい。

## リバースプロキシの仕込み(Nginx)

単純に、`/.well-known/acme-challenge/`に来たらS3のバケットにリクエストすればよい。

これで十分、このやり取りは暗号で行なう必要すら無いし。

```
location /.well-known/acme-challenge/ {
    proxy_pass   http://your-token-bucket.s3-website-ap-northeast-1.amazonaws.com/;
}
```

AutoScaleとかでHTTPサーバが並んでしまっているとかの環境でも、同じ設定にしておけば大丈夫なのがよいね。


## HTTPチャレンジを通してみた

じゃあ手元のPry(Ruby)で。

[letsencrypt - RubyでLet's Encryptのスクリプト - Qiita](http://qiita.com/sawanoboly/items/f23e73c613e9454076b8 "letsencrypt - RubyでLet's Encryptのスクリプト - Qiita")

ドメインのチャレンジを発行してもらいます。

```ruby
(略)クライアント作成とか、Authとかをすませて。

# チャレンジ内容の取得
> challenge.filename
=> ".well-known/acme-challenge/-oiuJqnLwVNitQlAKQfoX5cw0SpOKk8Y5ErRwFtG_d0"

> challenge.file_content
=> "-oiuJqnLwVNitQlAKQfoX5cw0SpOKk8Y5ErRwFtG_d0.7HGvevoQOzBU7mTxRBHr572hhPEd_Hx-ugxK_mF13yI"

> challenge.content_type
=> "text/plain"
```

上記の内容を元に、s3にオブジェクトを作成しましょう。

そのあと`request_verification`。

```
> challenge.request_verification
=> true
```

すぐにboulderからリクエストが来た、バッチリ200で返せたね。

```
66.133.109.36 - - [05/Jan/2016:04:53:32 +0000] "GET /.well-known/acme-challenge/-oiuJqnLwVNitQlAKQfoX5cw0SpOKk8Y5ErRwFtG_d0 HTTP/1.1" 200 88 "-" "Mozilla/5.0 (compatible; Let's Encrypt validation server; +https://www.letsencrypt.org)"
```

チャレンジ`valid`、と。

```ruby
> challenge.verify_status
=> "valid"
```

これでcsrを作って送信すれば手元に証明書作成完了と。メールで担当者に送るとか、ロードバランサに適用するため、AWSにアップするとかできるね。


## 気になること

引けたレコードと応答するサーバが全然関係無くても、共通のバックエンドを指定しているサーバにさえリクエストがくれば普通にチャレンジ成功します。
なのでドメインを持っていることが前提で、とりあえず`*`なAレコードを用意してあれば結構自由につくれる。

まあ、ドメイン1つにつき、証明書上限10ヶくらいのようだし、DV認証なんてそのくらい手軽でよいよね。
