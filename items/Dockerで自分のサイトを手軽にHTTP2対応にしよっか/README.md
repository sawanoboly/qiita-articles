
[HTTP/2](https://http2.github.io/ "HTTP/2")もほぼ仕様ができました。
代表的なWEBブラウザのうち、Firefox, Google ChromeならHTTP2が利用可能です。

HTTP/2は大雑把にいうとデータの転送周りでいろいろ効率がよくなって嬉しい感じ。
真面目に導入するにはまだ大変だけど、普通のサイトでもとりあえずDockerがあればHTTP/2に対応させて試すことができる。

## 環境

普通のWeb環境のモデルとして、Amazon EC2上にWordPressをHTTP(TCP/80)のみでホストしてるAMIMOTOさんを使ってみた。

[WordPress AMI 網元](http://ja.amimoto-ami.com/ "AWS + Nginx + WordPress | WordPress AMI 網元")

AMIからEC2インスタンスを起動すれば、平文のHTTPでWordPressが動いている。
※ 起動時にセキュリティグループでHTTPS(TCP/443)も許可しておこう。

Dockerで起動するプロセスは、HTTP2で受けてupstream(reverse proxy)できる何かで。


## Dockerの準備

Amazon LinuxなのでDockerホストはパッケージですぐに開始できる。

```
      ___         _            __
     / _ | __ _  (_)_ _  ___  / /____
    / __ |/  ' \/ /  ' \/ _ \/ __/ _ \
   /_/ |_/_/_/_/_/_/_/_/\___/\__/\___/

  
Amazon Linux AMI release 2014.09
    Nginx 1.6 + PHP 5.5 + Percona 5.6

  amimoto     http://ja.amimoto-ami.com/
  digitalcube https://www.digitalcube.jp/

[ec2-user@ip-xxx-xxx-xxx-xxx ~]$
```

yumでOK、起動しておこう。

```
$ sudo yum install docker -y
$ sudo service docker start
```


## サンプルのDockerファイルを取得

真似する用にDocerfileをGithubにおいてます。 #=> [higanworks/http2_by_docker](https://github.com/higanworks/http2_by_docker)

適当にクローンしましょう。

```
$ mkdir github.com && cd $_
$ git clone https://github.com/higanworks/http2_by_docker.git
```


## WordPress側の仕込み

このやり方ではリバースプロキシになるため、WordPressに到達したリクエストはHTTPになります。
とりあえず何も考えずにHTTPSを使うようにコンフィグで指定します。

`/var/www/vhosts/i-xxxx/wp-config.php`

```
$_SERVER['HTTPS'] = 'on';
```

注： この設定を入れるとスタイルなどの部品がすべてHTTPSのリンクで埋め込まれるようになります。

真面目に使う場合は次のいずれかでもうすこし細かく調整すればいいとおもいます。

- `wp-config.php`で条件を書いて`HTTPS`ヘッダを操作する
- nginxで条件`x-forwarded-***`を付与したりする


## h2oをつかってHTTP/2

ではまずh2oを使ってみましょう。

[h2o/h2o](https://github.com/h2o/h2o "h2o/h2o")

h2oのDockerfileは [tcnksm/dockerfiles](https://github.com/tcnksm/dockerfiles "tcnksm/dockerfiles") をほぼ拝借しました。

Docerfileを確認したら、イメージをビルドします。

```
$ cd h2o
$ sudo docker build -t local/h2o .

...

Successfully built 21fee39e3a54
```


h2oのコンフィグはこんな感じで。`proxy.reverse.url`がDocker用バーチャルネットワークを指しています。

```YAML:conf/h2o.conf
listen:
  port: 8080
  ssl:
    certificate-file: examples/h2o/server.crt
    key-file: examples/h2o/server.key
hosts:
   default:
    paths:
      /:
        proxy.reverse.url: http://172.17.42.1:80/
    access-log: /dev/stdout
```


ではDockerでh2oプロセスを上げましょう。だいたい次のような指定です。

- ポートをフォワード
- コンフィグのディレクトリをマウント
- 引数をつかってマウントしたコンフィグを渡す。

```
$ sudo docker run -t --rm -p 443:8080 -v `pwd`/conf:/conf local/h2o /conf/h2o.conf
```

これでWEBブラウザからのリクエストをTLS(TCP/443)で受けつけて、h2oの8080に流し、さらにnginxへと渡していきます。

早速ブラウザで` https://${ec2_public_ip}`を開いてみましょう。

![HTTP2___Just_another_WordPress_site.png](https://qiita-image-store.s3.amazonaws.com/0/7454/3e17fb72-36f8-4343-dc4c-8117a2cfe0cc.png "HTTP2___Just_another_WordPress_site.png")


開発ツールをみると、serverがh2oで、HTTP2を使っていることがわかります。

![ネットワーク_-_https___ec2-176-34-3-108_ap-northeast-1_compute_amazonaws_com_wp-login_php_と_HTTP2___Just_another_WordPress_site.png](https://qiita-image-store.s3.amazonaws.com/0/7454/828cc4c6-f30e-8aef-1095-898cdba6d88a.png "ネットワーク_-_https___ec2-176-34-3-108_ap-northeast-1_compute_amazonaws_com_wp-login_php_と_HTTP2___Just_another_WordPress_site.png")


h2oはクライアントの対応状況によってHTTP2ではなくHTTP1.xを自動で切り替えるので、大抵のブラウザ、クライアントは普通にサイトを表示できます。


## TrusterdをつかってHTTP/2

同じようなことを、今度は[Trusterd](http://trusterd.github.io/ "Trusterd - HTTP/2 Web Server scripting with mruby by trusterd")でやってみます。

こちらもイメージのビルドから。

```
$ cd ../trusterd
$ sudo docker build -t local/trusterd .

...

```

Trusterdの設定ファイルはこんな感じ。Ruby!

```ruby:conf/trusterd.conf
SERVER_NAME = "Trusterd"
SERVER_VERSION = "0.0.1"
SERVER_DESCRIPTION = "#{SERVER_NAME}/#{SERVER_VERSION}"
 
root_dir = "/usr/local/trusterd"
 
s = HTTP2::Server.new({
  :port           => 8080,
  :document_root  => "#{root_dir}/htdocs",
  :server_name    => SERVER_DESCRIPTION,
 
  :worker         => "auto",
  :server_host  => "0.0.0.0",
 
  :debug  =>  true,
  :key    => "/usr/local/src/trusterd/docker/conf/server.key",
  :crt    => "/usr/local/src/trusterd/docker/conf/server.crt",
  :tls => true,
 
  :callback => true,
 
  :run_user => "nobody",
 
  :server_status => true,
  :upstream => true,
 
})
 
s.set_map_to_storage_cb {
  s.upstream_uri = s.percent_encode_uri
  s.upstream_host = "172.17.42.1"
  s.upstream_port = 80
}
 
s.run
```

DockerでTrusterdプロセスを起動するときはこう。h2oの時と基本同じやり方です。

```
$ sudo docker run -t --rm -p 443:8080 -v `pwd`/conf:/conf local/trusterd /conf/trusterd.conf
```


こちらもブラウザで開発ツールから確認します、`HTTP/2.0`が使われていますね。

![ネットワーク_-_https___ec2-176-34-3-108_ap-northeast-1_compute_amazonaws_com_.png](https://qiita-image-store.s3.amazonaws.com/0/7454/2883fa3c-6486-5c6d-de6e-174195e24bb5.png "ネットワーク_-_https___ec2-176-34-3-108_ap-northeast-1_compute_amazonaws_com_.png")


Trusterdは`via`ヘッダをつかうなど、プロキシらしい振る舞いが見られます。要所のコールバックでヘッダはじめ色々と通信内容に介入できるのも面白い。

ただ本記事公開時バージョンのTrusterdはHTTP/2に対応していないWEBブラウザ・クライアントには無慈悲です。`bad connection`で接続を切っておしまい。

```trusterd_log(stdout)
xxx.xxx.xxx.xxx connected
session_recv: datalen = 611
Fatal error: Received bad connection preface114.172.124.198 disconnected
```

主に切られるのは<del>Safari</del>(<=対応した)とIE。。？ で考えると別にいいのかもよ。


## h2oとtrusterd

使った2つのHTTP/2サーバをざっくり比較。

| - |h2o|trusterd|
|---|----|----|
|HTTPプロトコル| 1.0/1.1/2.0| 2.0 |
|設定ファイル|yaml (ビルド時にWITH_MRUBYでmRubyも。)|mRuby |
|プロキシらしいふるまい|特になし|プロキシ的ヘッダを付ける(任意で追加も可)|
|安定度|ほぼ安定 |よーわからん |
|即戦力度|実績もあるっぽいし、十分 |これから。ユーザフィードバック次第？ |
|拡張性|Cが書けるなら拡張できそう。trusterd 作者により[h2o_mrubyが追加され](http://hb.matsumoto-r.jp/entry/2015/06/30/144717)、基本的なクラスの実装もすすんでいる。 |フルmRubyで結構やりたい放題。Cが読み書きできるとなおよし。 |
|ロゴ|ない |かっこいい |

とりあえずh2oで、リクエスト・レスポンスの動作に(手軽に)凝ってみたければTrustedで。君のサイトもHTTP/2対応にしてみよう。


> 表の更新
> - 2015.8 h2oにmruby拡張を追記
