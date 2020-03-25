<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Amazon Linux2014.09からyumでインストールできるnginxが1.6.xになったので、SPDY対応してみる。


## Nginx 1.6をインストール

ではおもむろに`yum install nginx`してnginxをインストール。

```shell:
$ sudo yum install nginx

-- snip --

Installed:
  nginx.x86_64 1:1.6.1-2.21.amzn1                                                                                                                                                           

Dependency Installed:
  GeoIP.x86_64 0:1.4.8-1.5.amzn1    gd.x86_64 0:2.0.35-11.10.amzn1    gperftools-libs.x86_64 0:2.0-11.5.amzn1    libXpm.x86_64 0:3.5.10-2.9.amzn1    libunwind.x86_64 0:1.1-2.1.amzn1   

Complete!
```

ビルドオプションの`--with-http_spdy_module`を確認してみます。

```shell
$ nginx -V 2>&1 | tr "\ " "\n" 
nginx
version:
nginx/1.6.1
built
by
gcc
4.8.2
20131212
(Red
Hat
4.8.2-7)
(GCC)

TLS
SNI
support
enabled
configure
arguments:
--prefix=/usr/share/nginx
--sbin-path=/usr/sbin/nginx
--conf-path=/etc/nginx/nginx.conf
--error-log-path=/var/log/nginx/error.log
--http-log-path=/var/log/nginx/access.log
--http-client-body-temp-path=/var/lib/nginx/tmp/client_body
--http-proxy-temp-path=/var/lib/nginx/tmp/proxy
--http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi
--http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi
--http-scgi-temp-path=/var/lib/nginx/tmp/scgi
--pid-path=/var/run/nginx.pid
--lock-path=/var/lock/subsys/nginx
--user=nginx
--group=nginx
--with-file-aio
--with-ipv6
--with-http_ssl_module
--with-http_spdy_module        ## これ
--with-http_realip_module
--with-http_addition_module
--with-http_xslt_module
--with-http_image_filter_module
--with-http_geoip_module
--with-http_sub_module
--with-http_dav_module
--with-http_flv_module
--with-http_mp4_module
--with-http_gunzip_module
--with-http_gzip_static_module
--with-http_random_index_module
--with-http_secure_link_module
--with-http_degradation_module
--with-http_stub_status_module
--with-http_perl_module
--with-mail
--with-mail_ssl_module
--with-pcre
--with-google_perftools_module
--with-debug
--with-cc-opt='-O2
-g
-pipe
-Wall
-Wp,-D_FORTIFY_SOURCE=2
-fexceptions
-fstack-protector
--param=ssp-buffer-size=4
-m64
-mtune=generic'
--with-ld-opt='
-Wl,-E'
```


## 自己署名のサーバ証明書セットを作成

取り急ぎ、`/etc/nginx/cert.key`と`/etc/nginx/cert.pem`を、PublicHostnameを証明書のCNにして作ります。

```shell:
$ openssl genrsa 2048 | sudo tee /etc/nginx/cert.key
$ sudo chmod 0400 /etc/nginx/cert.key

$ sudo sh -c 'openssl req -subj "/C=JP/ST=Kobe-shi/L=Chuo-ku/O=OpsRock LLC/OU=opsrock.com/CN=`/opt/aws/bin/ec2-metadata | grep public-hostname | cut -d \" \" -f 2`" \
-new -x509 -nodes -sha1 -days 3650 -key /etc/nginx/cert.key > /etc/nginx/cert.pem'
```


## NginxをSPDYに

デフォルト設定ファイル`/etc/nginx/nginx.conf`の109行目以降にHTTPS設定の雛形があるので、軽くコメントアウトします。
で、`listen`に **spdy** を追記。

Vimで編集するのが面倒だったらsedでやってもいいです。
`sudo sed -e '109,126s/#//' -e '110s/443/443\ spdy/'  /etc/nginx/nginx.conf -i`

```
    # HTTPS server
    #
    server {
        listen       443 spdy; ## ここだけ追記
        server_name  localhost;
        root         html;

        ssl                  on;
        ssl_certificate      cert.pem;
        ssl_certificate_key  cert.key;

        ssl_session_timeout  5m;

        ssl_protocols  SSLv2 SSLv3 TLSv1;
        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers   on;

        location / {
        }
    }

```


そしてNginxを起動。

```
$ sudo service nginx start
Starting nginx:                                            [  OK  ]
```

確認にはGoogle Chromeの`SPDY indicator`を使いました。

![Test_Page_for_the_Nginx_HTTP_Server_on_the_Amazon_Linux_AMI.png](https://qiita-image-store.s3.amazonaws.com/0/7454/54e9991a-25e9-ed42-089e-667863baa6d0.png "Test_Page_for_the_Nginx_HTTP_Server_on_the_Amazon_Linux_AMI.png")

SPDYっすね。

自己署名では出だしがこうなりますけどね。。


![プライバシー_エラー.png](https://qiita-image-store.s3.amazonaws.com/0/7454/fb2674d7-0559-bfdf-d611-8f8daec71993.png "プライバシー_エラー.png")
