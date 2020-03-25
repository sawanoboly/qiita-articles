<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Dokkuをインストールした環境ではNginxチームが保守するNginx1.6.x(stable)のリポジトリが導入される。
このリポジトリを含め、debian系が提供する`nginx-extras`はちょっといいモジュールが追加してビルドしたNginxのパッケージだ。

特に注目は`Embedded Lua`。Spdyも目立つがこちらは標準でも含まれているので。

```
# apt-cache show nginx-extras
Package: nginx-extras
Source: nginx
Priority: extra
Section: httpd
Installed-Size: 1465
Maintainer: Kartik Mistry <kartik@debian.org>
Architecture: amd64
Version: 1.6.0-1+trusty0

-- snip --

Description: nginx web/proxy server (extended version)
 Nginx ("engine X") is a high-performance web and reverse proxy server
 created by Igor Sysoev. It can be used both as a standalone web server
 and as a proxy to reduce the load on back-end HTTP or mail servers.
 .
 This package provides a version of nginx with the standard modules, plus
 extra features and modules such as the Perl module, which allows the
 addition of Perl in configuration files.
 .
 STANDARD HTTP MODULES: Core, Access, Auth Basic, Auto Index, Browser,
 Charset, Empty GIF, FastCGI, Geo, Gzip, Headers, Index, Limit Requests,
 Limit Zone, Log, Map, Memcached, Proxy, Referer, Rewrite, SCGI,
 Split Clients, SSI, Upstream, User ID, UWSGI.
 .
 OPTIONAL HTTP MODULES: Addition, Auth Request, Debug, Embedded Perl, FLV,
 GeoIP, Gzip Precompression, Image Filter, IPv6, MP4, Random Index, Real IP,
 Secure Link, Spdy, SSL, Stub Status, Substitution, WebDAV, XSLT.
 .
 MAIL MODULES: Mail Core, IMAP, POP3, SMTP, SSL.
 .
 THIRD PARTY MODULES: Auth PAM, Chunkin, DAV Ext, Echo, Embedded Lua,
 Fancy Index, HttpHeadersMore, HTTP Substitution Filter, http push,
 Nginx Development Kit, Upload Progress, Upstream Fair Queue.
Description-md5: a1852af8d0ef61a51df8e6cb9581d6a6
```


ちなみにアプリケーションファイアーウォールが使いたい場合、nginx-naxsi(WAF for NGINX)というのもある。

## nginx-extras 1.6.xの導入

`apt-get`で入れよう。

```
# apt-get install nginx-extras
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following extra packages will be installed:
  liblua5.1-0 libperl5.18
Suggested packages:
  nginx-doc
The following packages will be REMOVED:
  nginx-full
The following NEW packages will be installed:
  liblua5.1-0 libperl5.18 nginx-extras
0 upgraded, 3 newly installed, 1 to remove and 0 not upgraded.
Need to get 699 kB of archives.
After this operation, 922 kB of additional disk space will be used.
```

これだけでOK。

## サンプル

で、前回の[Dokkuでアプリ別のカスタムNginx設定 - Qiita](http://qiita.com/sawanoboly/items/614f81c13ceadb9a3cff "Dokkuでアプリ別のカスタムNginx設定 - Qiita") と組み合わせて。

`nginx.inc.conf`を次のようにluaでコンテンツを返すように書いてみます。


```nginx.inc.conf
location = /lua {
    default_type 'text/plain';
    
    content_by_lua "ngx.say('Hello,lua module!')";
}
```

Dokkuにアプリをデプロイして、`/lua`にアクセス。

```
$ curl -k https://sinatra-test.example.com/lua
Hello,lua module!
```

できた。


`$APP/ENV`から値を取るようにnginx内で仕込めればフロントエンドとして使いやすくなりそう。

追記：
とりあえずのお勧めは、`nginx.conf`に`env DOKKU_ROOT=/home/dokku;`あたりを書いておくとよさそう。
