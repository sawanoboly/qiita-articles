<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

名前ベースのSSLを。

## nginxでsniが有効になっているか確認
コンパイル時に`--with-http_ssl_module` をつけつつ、OpenSSLのライブラリが対応していればよい。
パッケージ版でも有効なので大体の環境では問題ないでしょう。

```shell:CLI_output
$ nginx -V
nginx version: nginx/1.2.1
TLS SNI support enabled
```

`TLS SNI support enabled`となってればOK。

これでSSLのClientHelloにくっついてきたサーバ名を拾ってくれる。


## nginxの設定

nginxにインクルードするコンフィグの例、[本家ドキュメント](http://nginx.org/en/docs/http/configuring_https_servers.html)案内通り、普通に設定してよい。
ファイルを用意するのが面倒なので[HttpLuaModule](http://wiki.nginx.org/HttpLuaModule)でコンテンツを返却しています。

```nginx:nginx_sni.conf
server {
        listen 443;
        server_name cert1.example.com;

        root html;
        index index.html index.htm;

        ssl on;
        ssl_certificate certs/cert1.example.com.cert;
        ssl_certificate_key certs/cert1.example.com.key;

        location / {
                default_type 'text/plain';
                content_by_lua 'ngx.say(ngx.var.server_name)';
        }
}

server {
        listen 443;
        server_name cert2.example.com;

        root html;
        index index.html index.htm;

        ssl on;
        ssl_certificate certs/cert2.example.com.cert;
        ssl_certificate_key certs/cert2.example.com.key;

        location / {
                default_type 'text/plain';
                content_by_lua 'ngx.say(ngx.var.server_name)';
        }
}

server {
        listen 443 default;
        server_name cert3.example.com;

        root html;
        index index.html index.htm;

        ssl on;
        ssl_certificate certs/cert3.example.com.cert;
        ssl_certificate_key certs/cert3.example.com.key;
   
        location / {
                default_type 'text/plain';
                content_by_lua 'ngx.say(ngx.var.server_name)';
        }
}
                                                       
```

### OpenSSLのクライアントでテスト
`-servername cert*.example.com` を渡して証明書の使い分けをテスト。

```shell:CLI_output
# cert1 が使われている
$ openssl s_client -quiet -connect localhost:443 -servername cert1.example.com
depth=0 /C=JP/ST=Osaka/L=Osaka-shi/O=HiganWorks LLC./CN=cert1.example.com

# cert2 が使われている
$ openssl s_client -quiet -connect localhost:443 -servername cert2.example.com
depth=0 /C=JP/ST=Osaka/L=Osaka-shi/O=HiganWorks LLC./CN=cert2.example.com

# cert3 が使われている
$ openssl s_client -quiet -connect localhost:443 -servername cert3.example.com
depth=0 /C=JP/ST=Osaka/L=Osaka-shi/O=HiganWorks LLC./CN=cert3.example.com

# 未設定ならデフォルトが使われる
$ openssl s_client -quiet -connect localhost:443 -servername cert4.example.com
depth=0 /C=JP/ST=Osaka/L=Osaka-shi/O=HiganWorks LLC./CN=cert3.example.com
```

それぞれのリクエストに対して、期待通りのサーバ証明書が選択されている。

### Hostを渡してコンテンツのチェック
名前ベースで別コンテンツを返している。

```shell:CLI_output
$ curl -k https://localhost:443/ -H 'Host: cert1.example.com'
cert1.example.com

$ curl -k https://localhost:443/ -H 'Host: cert2.example.com'
cert2.example.com

$ curl -k https://localhost:443/ -H 'Host: cert3.example.com'
cert3.example.com
```
※これらcurlで使われているサーバ証明書はすべてデフォルトの`CN: cert3.example.com`のものになる、理由は後述。


## ClientHello中サーバ名の確認
SNIではTCPセッション確立後の1発目、ClientHelloにサーバ名を混ぜることで暗号化前にサーバ名を設定できるようにしている。
というよりこのタイミングしか無いので仕方ない。

```
$ openssl s_client -connect localhost:443 -servername cert3.example.com -debug
CONNECTED(00000003)
write to 0x1461ff0 [0x1463700] (121 bytes => 121 (0x79))
-- snip --
0060 - 14 00 00 11 63 65 72 74-33 2e 65 78 61 6d 70 6c   ....cert3.exampl
0070 - 65 2e 63 6f 6d 00 23                              e.com.#
-- snip --
```

デバッグ出力から一つ目のWrite(ClientHello)のダンプを見るとサーバ名が含まれていた。


### curlのSNI対応
curlもsniに対応しているが、ClientHelloで渡す文字列が、

    curl https://{server_name}/

の`{server_name}`部分になっていた。

とういことで、下記のパターンでは`localhost`が使われるのでコンテンツは名前ベースのものが返ってくるけど証明書はデフォルトで指定しているものだけが使われてしまう。

    curl https://localhost/ -H "Host: cert1.example.com"

curlで動作確認するならhostsに書いてコモンネームをURL部分に含めます。

