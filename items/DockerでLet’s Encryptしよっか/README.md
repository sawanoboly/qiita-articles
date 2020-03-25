この記事ちょいちょい参考にする人いますが相当古いです、普通に下記のcertbotつかってください。

https://certbot.eff.org



----

Let's EncryptのCAがじわじわと信頼の手を広げている。

> 12/3にパブリックベータになったので、記述に追記をいれました。

- [Our First Certificate Is Now Live](https://letsencrypt.org/2015/09/14/our-first-cert.html)
- [Let's Encrypt is Trusted](https://letsencrypt.org/2015/10/19/lets-encrypt-is-trusted.html)


じゃあそろそろ試そうと、CLIインストールのドキュメント [Using the Let’s Encrypt client](https://letsencrypt.readthedocs.org/en/latest/using.html)を見たら`Running with Docker`というセクションがあった。


## t2.microのAmazonLinuxを上げてやってみる。

準備はこのくらい。

- ポート80を開けておこう
- PublicIPをなにかしらDNSで引けるようにしておこう

Dockerデーモンを上げよう。

```
$ sudo yum -y install docker
$ sudo service docker start
```


## Dockerでletsencryptコマンド実行

```
$ sudo docker run -it --rm -p 443:443 --name letsencrypt \
            -v "/etc/letsencrypt:/etc/letsencrypt" \
            -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
            quay.io/letsencrypt/letsencrypt:latest certonly
```

> 12/9修正: 
> authはcertonlyに変わりました。
> ついでに、-p 80:80 を追加して、 コマンドラインを`certonly --standalone` とすれば後のPythonWebサーバの手順は不要です。

> もうひとつ追記： 
> Webrootプラグインによる証明書取得は、Webサーバを停止せずに、自動取得ができる。 => [Webroot](https://http2.try-and-test.net/letsencrypt.html)
> せやね。

少し待つ。

ぬお、[紹介のビデオ](https://www.youtube.com/watch?v=Gas_sSB-5SU)で見たやつだ。

![opsrockin_—_ec2-user_ip-10-0-1-85___—_ssh_-A_ec2-user_52_69_130_56_-i____ssh_opsworks_amimoto_v1_—_177×48.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/d7730c57-0b8a-b4a4-efad-81523e96307d.jpeg "opsrockin_—_ec2-user_ip-10-0-1-85___—_ssh_-A_ec2-user_52_69_130_56_-i____ssh_opsworks_amimoto_v1_—_177×48.jpg")

ホスト名を`letsen.opsrockin.com`などと入れると、JSONをWebサーバで返してくれと言われる。 (IPだと駄目なので)

```
Make sure your web server displays the following content at
http://letsen.opsrockin.com/.well-known/acme-challenge/j0KEGQ8U6_KZBicqFxazIyEo_4FPAC0RvPadG_0SMLE before continuing:


Content-Type header MUST be set to application/jose+json.

If you don't have HTTP server configured, you can run the following
command on the target server (as root):

mkdir -p /tmp/letsencrypt/public_html/.well-known/acme-challenge
cd /tmp/letsencrypt/public_html
echo -n '{JSONの中身}' > .well-known/acme-challenge/j0KEGQ8U6_KZBicqFxazIyEo_4FPAC0RvPadG_0SMLE
# run only once per server:
$(command -v python2 || command -v python2.7 || command -v python2.6) -c \
"import BaseHTTPServer, SimpleHTTPServer; \
SimpleHTTPServer.SimpleHTTPRequestHandler.extensions_map = {'': 'application/jose+json'}; \
s = BaseHTTPServer.HTTPServer(('', 80), SimpleHTTPServer.SimpleHTTPRequestHandler); \
s.serve_forever()" 
```

作ったばかりのインスタンスでまだWebサーバもいないことだし、SimpleHTTPServerさんを使おう。

別のターミナルから、80をlistenするのでrootで起動。

```
mkdir -p /tmp/letsencrypt/public_html/.well-known/acme-challenge
cd /tmp/letsencrypt/public_html
echo -n '{"JSONの中身"}' > .well-known/acme-challenge/j0KEGQ8U6_KZBicqFxazIyEo_4FPAC0RvPadG_0SMLE
# run only once per server:
$(command -v python2 || command -v python2.7 || command -v python2.6) -c \
"import BaseHTTPServer, SimpleHTTPServer; \
SimpleHTTPServer.SimpleHTTPRequestHandler.extensions_map = {'': 'application/jose+json'}; \
s = BaseHTTPServer.HTTPServer(('', 80), SimpleHTTPServer.SimpleHTTPRequestHandler); \
s.serve_forever()" 
```

Docker側を進めると、Webサーバにリクエストが送られてきていた。

そしてちょっとまつと、サーバ証明書が作られた。

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/letsen.opsrockin.com/fullchain.pem. Your cert
   will expire on 2016-01-21. To obtain a new version of the
   certificate in the future, simply run Let's Encrypt again.
```

JSONをつついて、ドメインの所有を確認しているのね。


## 出来たサーバ証明書を試してみる

使うファイルはこれらかね？ `fullchain`は`cert + chain`だった。nginxとかで使う形式。

```
/etc/letsencrypt/live/letsen.opsrockin.com/fullchain.pem
/etc/letsencrypt/live/letsen.opsrockin.com/cert.pem
/etc/letsencrypt/live/letsen.opsrockin.com/chain.pem
/etc/letsencrypt/live/letsen.opsrockin.com/privkey.pem
```

鍵ペアが使えるかどうか、`openssl`コマンドでサーバを立てる。


```
$ sudo openssl s_server -cert /etc/letsencrypt/live/letsen.opsrockin.com/cert.pem -key /etc/letsencrypt/live/letsen.opsrockin.com/privkey.pem -CAfile /etc/letsencrypt/live/letsen.opsrockin.com/chain.pem
Using default temp DH parameters
Using default temp ECDH parameters
ACCEPT
```

で、同じくクライアントで。

```
$ openssl s_client
CONNECTED(00000003)
depth=1 CN = happy hacker fake CA
verify error:num=19:self signed certificate in certificate chain
verify return:0
---
Certificate chain
 0 s:/CN=letsen.opsrockin.com
   i:/CN=happy hacker fake CA
 1 s:/CN=happy hacker fake CA
   i:/CN=happy hacker fake CA
---
Server certificate
-----BEGIN CERTIFICATE-----
MIIE7TCCA9WgAwIBAgITAPqv3jjriYXOmiuQpfWx6J4vvzANBgkqhkiG9w0BAQsF
ADAfMR0wGwYDVQQDDBRoYXBweSBoYWNrZXIgZmFrZSBDQTAeFw0xNTEwMjMwOTI2
MDBaFw0xNjAxMjEwOTI2MDBaMB8xHTAbBgNVBAMTFGxldHNlbi5vcHNyb2NraW4u

...

---
New, TLSv1/SSLv3, Cipher is ECDHE-RSA-AES256-GCM-SHA384
Server public key is 2048 bit
Secure Renegotiation IS supported
Compression: NONE
Expansion: NONE
SSL-Session:
    Protocol  : TLSv1.2

...
```

CA証明書のチェーンが、`happy hacker fake CA`となっているが、ちゃんとTLSで接続を確立できた。

さすがにこの状態では使っても普通の自己署名扱いだけど、そのうち`fake CA`では無く、次のようなチェインになるのだろう。これで済むならまったくカンタンだね。

> 12/9 追記: ベータでもチェーンは下記の信頼できるものになりました。

```
---
Certificate chain
 0 s:/CN=letsencrypt.org/O=INTERNET SECURITY RESEARCH GROUP/L=Mountain View/ST=California/C=US
   i:/C=US/O=IdenTrust/OU=TrustID Server/CN=TrustID Server CA A52
 1 s:/C=US/O=IdenTrust/OU=TrustID Server/CN=TrustID Server CA A52
   i:/C=US/O=IdenTrust/CN=IdenTrust Commercial Root CA 1
 2 s:/C=US/O=IdenTrust/CN=IdenTrust Commercial Root CA 1
   i:/O=Digital Signature Trust Co./CN=DST Root CA X3
```

## ついでに、CLIのヘルプもでる。

エントリーポイントがletsencryptコマンドなので、ヘルプやらもちゃんとでる。

> 12/9修正: ヘルプ表示内容を更新

```
$ sudo docker run -it --rm -p 443:443 --name letsencrypt             -v "/etc/letsencrypt:/etc/letsencrypt"             -v "/var/lib/letsencrypt:/var/lib/letsencrypt"             quay.io/letsencrypt/letsencrypt:latest --help

  letsencrypt [SUBCOMMAND] [options] [-d domain] [-d domain] ...

The Let's Encrypt agent can obtain and install HTTPS/TLS/SSL certificates.  By
default, it will attempt to use a webserver both for obtaining and installing
the cert. Major SUBCOMMANDS are:

  (default) run        Obtain & install a cert in your current webserver
  certonly             Obtain cert, but do not install it (aka "auth")
  install              Install a previously obtained cert in a server
  revoke               Revoke a previously obtained certificate
  rollback             Rollback server configuration changes made during install
  config_changes       Show changes made to server config during installation
  plugins              Display information about installed plugins

Choice of server plugins for obtaining and installing cert:

  --apache          Use the Apache plugin for authentication & installation
  --standalone      Run a standalone webserver for authentication
  --nginx           Use the Nginx plugin for authentication & installation
  --webroot         Place files in a server's webroot folder for authentication

OR use different plugins to obtain (authenticate) the cert and then install it:

  --authenticator standalone --installer apache

More detailed help:

  -h, --help [topic]    print this message, or detailed help on a topic;
                        the available topics are:

   all, automation, paths, security, testing, or any of the subcommands or
   plugins (certonly, install, nginx, apache, standalone, webroot, etc)

```

