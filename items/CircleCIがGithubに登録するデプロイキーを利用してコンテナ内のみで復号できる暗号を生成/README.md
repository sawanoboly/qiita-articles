
例えばCircleCIの環境変数に何かの認証情報を入れたいという時。プライベートとわかっちゃいるが、文字列を直接貼ることにどうしても気持ち悪さを覚える際の選択肢。
屋上屋を架す。

## CircleCIのデプロイキー

GithubのリポジトリをCircleCIと関連付けると、public/private問わず、cloneするためのデプロイキーが登録されます。

こんな感じですね。

![Deploy_keys.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/6557716c-bb56-49ca-c509-764dc0858e56.jpeg "Deploy_keys.jpg")

この公開鍵に対する秘密鍵は、CirlcleCIでコンテナが起動するときにセットされています。

具体的にはココ。 `~/.ssh/id_circleci_github`。フィンガープリントで比較してみます。あってますね。

```
$ ssh-keygen -yf ~/.ssh/id_circleci_github > pub ; ssh-keygen -lf pub
1024 fc:85:2a:xxx:9f:c4:b9 pub (RSA)
```

この秘密鍵の属性をちょっと考える。

- CircleCIだけが保持
    - ※プロジェクトメンバーは取り出し可っちゃあ可
- プロジェクト単位で一意
    - すべてのビルドで、に同じパス・同じ文字列で必ず存在
- 文字列としての強度はそこそこ

なら、この鍵に関連付けて何かしら文字列を暗号化すれば、コンテナ内部でだけ複合可能な暗号を作れるはず。


## 暗号・復号してみる。

秘密鍵を共通文字列としてシンプルに扱い、共通鍵暗号の方式を使ってみる。コンテナログイン時にあった`openssl.patch`を暗号にしてみよう。

```
$ openssl enc -e -aes256 -in openssl.patch -kfile ~/.ssh/id_circleci_github | base64 -w0
U2FsdGVkX1+IV5qpDaATVIjHcEg16XXMprhCSln5s9uJjWz4GudmS3xxxxxx
```

できた文字列を環境変数に。

![Project_settings_-_giraffi_circleci-sandbox_-_CircleCI.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/ec6c981f-dc18-b85d-4a02-2c531ed73616.jpeg "Project_settings_-_giraffi_circleci-sandbox_-_CircleCI.jpg")

環境変数が有効になったコンテナを新たに起動して、復号してみよう。
base64をデコードしてから、opensslコマンドへ。

```
$ echo -n $OPENSSLPATCH | base64 -d | openssl enc -d -aes256 -kfile ~/.ssh/id_circleci_github
--- ruby-1.9.2-p180/ext/openssl/ossl_ssl.c	2010-12-23 22:24:00.000000000 -0500
+++ ruby-1.9.2-p180/ext/openssl/ossl_ssl_fixed.c	2011-10-28 11:39:30.265970001 -0400
@@ -107,9 +107,9 @@
     OSSL_SSL_METHOD_ENTRY(TLSv1),
     OSSL_SSL_METHOD_ENTRY(TLSv1_server),
     OSSL_SSL_METHOD_ENTRY(TLSv1_client),
```

CircleCIの環境変数ではそこそこの文字列長が許されているので、少々ならばコンフィグ一式をtarにしてから暗号にした文字列、とかでも登録可能だった。

もっとこだわる(というか秘密鍵らしく使う)なら`S/MIME`? より手っ取り早くやるなら秘密鍵の文字列でパスワードをつけたZIPをつかう、なんて手もありますね。


## まとめや他の選択肢など

この手法、下記条件で解読可能になることを割り切るならつかえなくもない。

- CircleCIがガッツリクラックされて
    - デプロイキーがバレる (A)
    - プロジェクトの環境変数が抜かれる (B)
    - その環境変数がデプロイキーで複合できる暗号だと見抜かれる

A,Bが揃わないと、内容を確認することは無理にはなる。Bに至っては環境変数に入れるまでもなく、リポジトリに含まれていたり適当に散布したって大丈夫だろう。
まあ『デプロイキーがバレる』の時点でプライベートリポジトリがクローンできてしまうので、その場合は暗号うんぬん以前の問題ではあるね。

CircleCIは元々AWSのクレデンシャルが利用できるので、例えばAWS Key Management Serviceなどが使えるならそっちを使う方がむしろ汎用的だと思います。
