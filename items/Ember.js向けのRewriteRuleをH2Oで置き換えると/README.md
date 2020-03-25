
Ember.jsのWebアプリケーションをH2Oでホストできるかな。


- [Ember.js - A framework for creating ambitious web applications.](http://emberjs.com/ "Ember.js - A framework for creating ambitious web applications.")


## Ember.jsのルーターと挙動について簡単に

Ember.jsのバージョンは1.13くらい系。
アプリケーションをビルドすると、ざっくり下のようなツリーができます。

```
dist/
├── assets
│   ├── vendor-******.css
│   └── vendor-******.js
├── crossdomain.xml
├── index.html
└── robots.txt
```

Ember.jsに限らずこの手のWebアプリケーションは、サーバサイドではJavascriptのファイルを配布にとどめ、クライアントで全部動かします。

ブラウザでの見た目は次のようになりますが、

- /login => ログインページ
- /users => ユーザのページ

すべてのリクエストは`index.html`に仕込まれたルータが対応しています。


### mod_rewriteなどでブラウザのリロードに対応する

通常使っている分には特に違和感無く使えますが、例えば`/users`の表示中にブラウザでリロードすると、`index.html`のルータを通りません。
普通に`/users`へのリクエストとなるため、対策を取っていないと404になります。

この対策に、Apache httpdでmod_rewriteを使用する場合の例が下記です。

```
RewriteEngine On
RewriteRule ^index\.html$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule (.*) index.html [L]
```

おおざっぱにいうと、リクエストが存在するファイル/ディレクトリでなければ、`index.html`に処理を投げます。


## H2OでEmber.js向けに設定を書いてみる

まずはこちらを参考に。

- [Kazuho's Weblog: H2OとPHPを組み合わせるの、超簡単です（もしくはmod_rewriteが不要な理由）](http://blog.kazuhooku.com/2015/06/h2ophpmodrewrite.html "Kazuho's Weblog: H2OとPHPを組み合わせるの、超簡単です（もしくはmod_rewriteが不要な理由）")

なるほど超簡単。

で、うまくいったのが次の記述。H2Oは1.6.0を使用、Webアプリケーションは`/public`に置いてます。


```
hosts:
  default:
    listen:
      port: 8000
    paths:
      /index.html:
        redirect: /
      /:
        file.dir: /public
        redirect:
          url: /index.html?
          internal: YES
          status: 307
```


いくらか試して、redirect.urlを`/index.html?`とするのが今のところ挙動がしっくりくる。


### ほか、redirect.urlで試した記述。

- NG: `/index.html` => リクエストが`/index.htmlusers`になり、さらにネストされたリダイレクトへ。
- NG: `/index.html/` => リクエストが`/index.html/users`になり、`/users`になり、リダイレクトループへ。
- OK?: `"/#"` => `internal=NO`でなら動作可能。2回リダイレクトされて元に戻る。旧Ember風な挙動。イマイチ。
- OK?: `/index.html#` => `internal=NO`でなら動作可能。2回リダイレクト後に"/"にたどり着けるのでなんとかなる模様。合計でリダイレクト3回。イマイチ。


H2OとJavaScriptフレームワークを組み合わせるのも超簡単。。かな。。？
