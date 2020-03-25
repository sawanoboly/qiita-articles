なにかしらアプリケーションを開発しており、公式のDockerイメージを配布する。最近よく見かけますね。
利用者側の意見として、アプリケーションの設定には環境変数が使えると助かったりします。

なんで？

- ADD/COPYがいらない
  - 変更のたびビルドしなくていい
- Mountつかわなくていい
  - Mount用のファイルを管理しなくていい
- docker-compose.ymlなどで完結

で、この環境変数からアプリケーションの設定というやり方、一部界隈のDockerイメージで命名規則が次のように、

- `{アプリケーション名}_{ディレクティブ}_{サブディレクティブ}`
- `{アプリケーション名}_{設定項目名}`

という感じの環境変数をUppercaseでアプリケーションに渡せば、設定ファイルの同じ項目を指定・または上書きするというルールがあります。

別にどんなイメージでもこうすればよいという主張ではなく、なんかしらの公式イメージとかを作って不特定多数に使ってもらう際にはこのようにしておくやり方もありますという話です。


例えばこれらがそんなアプローチを取っています。

- kafka(Confluent)とその周りのモジュール [confluentinc/cp-docker-images at v5.1.2](https://github.com/confluentinc/cp-docker-images/tree/v5.1.2)
- graylog [Graylog2/graylog-docker at 3.0.0-1](https://github.com/Graylog2/graylog-docker/tree/3.0.0-1)

> 追記: 後でもすこし触れているけど、設定ファイルのリファレンスがそのまま環境変数に置き換えられるのがこの規則の良いとこです。

## 環境変数から、Dockerコンテナ起動時に設定ファイルに追記するタイプ

例であげたkafka(Confluent)は、Dockerコンテナが起動する際に環境変数から設定ファイルへの追記を行います。

[confluentinc/cp-docker-images at v5.1.2](https://github.com/confluentinc/cp-docker-images/tree/v5.1.2)

設定ファイルに追記したい設定項目があれば、下記のような対応で環境変数をセットしてあげます。

- `default.replication.factor` <= `KAFKA_DEFAULT_REPLICATION_FACTOR`
- `kafka.num.partitions` <= `KAFKA_NUM_PARTITIONS`

このようにすることで、あらゆるパラメータを環境変数経由で設定できるようにしています。

実際どのようにやっているかは、下記のエントリーポイントファイル群を見るとなんとなくわかります。

- https://github.com/confluentinc/cp-docker-images/tree/v5.1.2/debian/kafka/include/etc/confluent/docker

自作？のラッパーで追記してるようですね。(流石につらくはあるので、冒頭でアプリケーション配布側であればできるようにしたほうがよい、程度の表現になってます。)


## 環境変数があったら、優先して使うタイプ

多分Graylog(未確定)がこのタイプなんだと思うのです。エントリーポイントをざっと見てもファイル変更してるように思えなかった。(Java力もないのでそれ以上追いかけませんでした。)

- [Graylog2/graylog-docker at 3.0.0-1](https://github.com/Graylog2/graylog-docker/tree/3.0.0-1)

graylogはkafkaと違い、設定ファイル上の区切りが`_`なので、``GRAYLOG_*`の環境変数の後半部分がlowercaseとアンダースコアの設定項目に対応してます。

- `is_master` <= `GRAYLOG_IS_MASTER`
- `password_secret` <= `GRAYLOG_PASSWORD_SECRET`

設定ファイルのリファレンスを読みながら、環境変数に起こしていきやすいですね。
そのためか、次のような質問も、法則があるからそれに従おうぜという回答で締められています。

- [Please document GRAYLOG_ELASTICSEARCH_HOSTS · Issue #8 · Graylog2/graylog-docker](https://github.com/Graylog2/graylog-docker/issues/8)

### おまけ、Rubygems例

ちなみに、Rubygemsの[thekompanee/chamber](https://github.com/thekompanee/chamber) なんかもそのパターンで、設定用のyamlファイル上にあるキーと環境変数が対になるような使い方ができるのでたまに使っています。

- [thekompanee/chamber: A surprisingly configurable convention-based approach to managing your application's custom configuration settings.](https://github.com/thekompanee/chamber)

こっちの環境変数があったら、優先して使うタイプはDockerコンテナに限らず利用できる点もよいですね。

## おわりに

このアプローチはわりと浸透している気もしますが、はっきりこうだという話もあまり出てないなと思ったので書いときました。

テンプレートからコンフィグファイルを作る、とかは項目が多くなってくると分岐で肥大化しちゃうことと、管理対象はなるべく少なくするのがやっぱりよいです。
使う側としても嬉しいし、自分で何か配布するときはそういうように作りたいな。
