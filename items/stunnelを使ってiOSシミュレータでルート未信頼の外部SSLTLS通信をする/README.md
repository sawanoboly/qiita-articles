<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

iOSで外部APIとSSL/TLS通信をするアプリを開発する場合、相手方がルートの信頼が無い証明書を使っていると何らかの方法でVerifyを回避する必要がある。

iPhone実機でテストするには適当なCA証明書を入れ、サーバ側にはそれで署名したサーバ証明書を使えばいい。

iOSシミュレータで未信頼CAの証明書を使うのは手間らしいのでSSL/TLSのレイヤを`stunnel`を使って下記のような設定をする。
シミュレータから見ればただの平文通信ということになる。

```stunnel.conf
-- snip --
[remote_https]
client = yes
accept = 127.0.0.1:10443
connect = your_domain:443
-- snip --
```

これで`localhost:10443`がListenされ、通信の転送先がリモートHTTPSサーバ宛になる。


ちなみに

```
"~/Library/Application Support/iPhone Simulator/5.0/Library/Keychains/TrustStore.sqlite3"
```

がiOSシミュレータの証明書ストアのようなので、ここにCA証明を入れれば良いのかもしれないがうまくいかない。
