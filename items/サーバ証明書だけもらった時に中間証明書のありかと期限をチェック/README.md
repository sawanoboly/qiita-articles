<!-- permanent -->

『サーバの証明書更新してください！ファイルはこれ』と、鍵セットだけ貰った時。いやそれどこ発行でいつまで有効やねんと。

とりあえずベンダをチェック。

```shell:
$ openssl x509 -in ssl.example.com -text | grep -i Issuer
        Issuer: C=JP, O=Cybertrust Japan Co., Ltd., CN=Cybertrust Japan Public CA G3
                CA Issuers - URI:http://sureseries-crl.cybertrust.ne.jp/SureServer/ctjpubcag3/ctjpubcag3.crt
```

発行者が`Cybertrust Japan Public CA G3`だ。
ほな中間証明書いるので、 https://www.cybertrust.ne.jp/ssl/support/download_ca.html#02 から取ってくるか。。。

ついでに期限をチェック。

```shell:
 $ openssl x509 -in ssl.example.com.cert.pem -text | grep -i after
            Not After : Sep 30 14:59:00 2015 GMT
```

リマインダにでも登録しとくか。
