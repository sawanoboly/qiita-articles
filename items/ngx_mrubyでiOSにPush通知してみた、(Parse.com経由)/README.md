
Parse.comにはREST APIから各種デバイスへのPushをキックする仕組みがあるので、じゃあNginxが押してもいいじゃんと。


## iOS Pushの仕込み

Pushを受ける側についてはドキュメントを追っていっただけなので箇条書き。

- Apple Dev Centerで一式そろえる。
    - pushアリのアプリ用証明書作成＆Push用証明書作成＆登録
    - デバイス登録
    - アプリ＆デバイスを含むプロビジョニングプロファイル作成(Developer)
- Parse.comでアプリ登録
    - アプリのPushに使う証明書を登録
- 通知用の空アプリをデプロイ
    - ParseStarterProject-SwiftをXCodeでビルド
    - 端末でアプリを起動し、Pushを許可
- Parse.comで端末登録の確認
    - ついでにチャンネル`Nginx`をSubscribe(手書き)


## ngx_mrubyで外部HTTPSリクエスト用のmgemを追加

ngx_mrubyに最初から用意されているサンプルの`build_config.rb`に対して、追加したのはこれら。

- mruby-httprequest
- mruby-polarssl

※ 今回はちょっとした都合(たまたま？)で `s/mruby-onig-regexp/mruby-pcre-regexp/` という変更もした。


## ではサンプルコード

使いどころとしては、何かしら不都合や攻撃があったタイミングで使えないかなーと考えていました。
しかし、まずはシンプルにこの方針。

1. 誰かがサイトを訪れたら
2. IPをiOSに通報する

やってみよう。

```
location / {
  mruby_content_handler_code '
    Server = Nginx

    c = Server::Connection.new

    http = HttpRequest.new()
    res = http.post(
           "https://api.parse.com/1/push",
           JSON.generate({
              "channels" => ["Nginx"],
              "data"  => {"alert" => "Your site was visited from #{c.remote_ip} ."}
            }),{
            "Content-Type"           => "application/json",
            "X-Parse-Application-Id" => "Parseのアプリケーションキー",
            "X-Parse-REST-API-Key"   => "ParseのREST APIキー"
            })

    Nginx.errlogger Nginx::LOG_WARN, res["status"]

    Server.echo "reported your access."
  ';
}
```


### ブラウザでリクエストした

ちちょっとDockerで立ち上げたサーバにブラウザでリクエスト。通報された。

![192_168_99_100_8080_と_192_168_99_100_8080_と_sawanoboriyu_—_bash_—_157×41.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/b12b8515-e043-5fa0-9bae-1caa5a59da10.jpeg "192_168_99_100_8080_と_192_168_99_100_8080_と_sawanoboriyu_—_bash_—_157×41.jpg")

リクエスト処理中に同期処理でParseを叩いているので、それなりに待つのは(今回は)仕方ないとして。。。
Nginx側のログに、Parseからの戻りが`200 OK`であることが記録された。

```
2015/09/02 xx:xx:xx [warn] 1#0: *1 200 OK, client: 192.168.99.1, server: _, request: "GET / HTTP/1.1", host: "192.168.99.100:8080"
```

### Push通知確認

そしてPushは来た。

![IMG_0573.JPG](https://qiita-image-store.s3.amazonaws.com/0/7454/66490988-854a-1100-55e9-04d134147e61.jpeg "IMG_0573.JPG")

何かに外部へのリクエストを使うなら、うまいことワーカープロセスが丸ごと待ちにならないよう書きたいね。

----

追記:

メモリが無尽蔵で外部APIのレスポンスがクライアントに対して不要ならば、ダブルforkで囲んじゃうという手がある。
