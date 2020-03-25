<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[Dokku](https://github.com/progrium/dokku)はアプリ別のNginx Configを自動作成して名前ベースのアクセスを可能にしている。

アプリによってはNginxで個別設定が行いたいこともある例えば次のようなケース。

- 静的ファイルは直接Nginxでホストしたい
- IPでアクセス制限をかけたい

アプリにカスタム設定を許すプラグインがこちら。

https://github.com/neam/dokku-nginx-vhosts-custom-configuration

導入にはちょっとDokku本体付属プラグインの`nginx-vhosts`に手を加える必要がある。
巻末付録として修正点を書いておいたので、やってみるならそちらも参照で。

## インクルード用nginx.confの追加

例えば社内IPからのアクセスのみ許可したいというNginxのコンフィグはこんな感じ。

```nginx:nginx.inc.conf
allow xxx.xxx.xxx.xxx;
allow xxx.xxx.xxxx.xxx/xx;
deny  all;
```

この`nginx.inc.conf`をリポジトリに加えよう。

## APPのConfigを作成

設定のキー、`NGINX_VHOSTS_CUSTOM_CONFIGURATION`にて、インクルードするファイル名を指定します。

```
$ dokku config:set <app_name> NGINX_VHOSTS_CUSTOM_CONFIGURATION=nginx.inc.conf
```

あとはこれでPushすればOK! インクルード先は`server`ディレクティブの直下になります。


各ホストから、curlでチェックしてみよう。

```
## 許可されているホストから
$ curl -k https://sinatra-test.example.com
Hello world!


## 許可されていないホストから
# curl -k https://sinatra-test.example.com
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.6.0</center>
</body>
</html>
````

効いてるね。


現バージョンの`dokku-nginx-vhosts-custom-configuration`プラグイン注意点を3つ
- 内容を有効にするにはアプリをデプロイする必要がある
- nginxコンフィグにエラーが有る場合、DokkuサーバのNginx全体が多分止まる
- NGINX_VHOSTS_CUSTOM_CONFIGURATIONをunsetしてもファイルは残る、無効化したいなら空ファイルで上書き

ここらをうまいこと避けたいところ。修正してプルリクをかけようと思う。


## 巻末付録：本体カスタマイズ

プラグインによって作成される`nginx-vhosts-custom-configuration.conf`ファイルはどこかでインクルードしないと動作しない。
追加は2行。`#この行追加`となっている箇所。

```shell:/var/lib/dokku/plugins/nginx-vhosts/post-deploy
#!/usr/bin/env bash
set -eo pipefail; [[ $DOKKU_TRACE ]] && set -x
APP="$1"; PORT="$2"
WILDCARD_SSL="$DOKKU_ROOT/tls"
SSL="$DOKKU_ROOT/$APP/tls"

if [[ -f "$DOKKU_ROOT/VHOST" ]]; then
  VHOST=$(< "$DOKKU_ROOT/VHOST")
  SUBDOMAIN=${APP/%\.${VHOST}/}
  if [[ "$APP" == *.* ]] && [[ "$SUBDOMAIN" == "$APP" ]]; then
    hostname="${APP/\//-}"
  else
    hostname="${APP/\//-}.$VHOST"
  fi

  if [[ -e "$SSL/server.crt" ]] && [[ -e "$SSL/server.key" ]]; then
    SSL_INUSE="$SSL"
    SSL_DIRECTIVES=$(cat <<EOF
  ssl_certificate     $SSL_INUSE/server.crt;
  ssl_certificate_key $SSL_INUSE/server.key;
EOF
)
  elif  [[ -e "$WILDCARD_SSL/server.crt" ]] && [[ -e "$WILDCARD_SSL/server.key" ]] && [[ $hostname = `openssl x509 -in $WILDCARD_SSL/server.crt -noout -subject | tr '/' '\n' | grep CN= | cut -c4-` ]]; then
    SSL_INUSE="$WILDCARD_SSL"
    SSL_DIRECTIVES=""
  fi

  # ssl based nginx.conf
  if [[ -n "$SSL_INUSE" ]]; then
  cat<<EOF > $DOKKU_ROOT/$APP/nginx.conf
upstream $APP { server 127.0.0.1:$PORT; }
server {
  listen      [::]:80;
  listen      80;
  server_name $hostname;
  return 301 https://\$host\$request_uri;
}

server {
  listen      [::]:443 ssl spdy;
  listen      443 ssl spdy;
  server_name $hostname;
$SSL_DIRECTIVES

  keepalive_timeout   70;
  add_header          Alternate-Protocol  443:npn-spdy/2;

# この行追加
  include $DOKKU_ROOT/$APP/nginx.conf.d/*-vhosts-custom-configuration.conf;

  location    / {
    proxy_pass  http://$APP;
    proxy_http_version 1.1;
    proxy_set_header Upgrade \$http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host \$http_host;
    proxy_set_header X-Forwarded-Proto \$scheme;
    proxy_set_header X-Forwarded-For \$remote_addr;
    proxy_set_header X-Forwarded-Port \$server_port;
    proxy_set_header X-Request-Start \$msec;
  }
}
EOF

  echo "https://$hostname" > "$DOKKU_ROOT/$APP/URL"
else
# default nginx.conf
  cat<<EOF > $DOKKU_ROOT/$APP/nginx.conf
upstream $APP { server 127.0.0.1:$PORT; }
server {
  listen      [::]:80;
  listen      80;
  server_name $hostname;
# この行追加
  include $DOKKU_ROOT/$APP/nginx.conf.d/*-vhosts-custom-configuration.conf;
  location    / {
    proxy_pass  http://$APP;
    proxy_http_version 1.1;
    proxy_set_header Upgrade \$http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host \$http_host;
    proxy_set_header X-Forwarded-Proto \$scheme;
    proxy_set_header X-Forwarded-For \$remote_addr;
    proxy_set_header X-Forwarded-Port \$server_port;
    proxy_set_header X-Request-Start \$msec;
  }
}
EOF

  echo "http://$hostname" > "$DOKKU_ROOT/$APP/URL"
  fi
  pluginhook nginx-pre-reload $APP
  sudo /etc/init.d/nginx reload > /dev/null
fi
```

なお、Nginxのincludeはワイルドカードが使われた時、ファイルや親ディレクトリが無くてもエラーにならないのでサーチパスで`*`を使っています。
