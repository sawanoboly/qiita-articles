kafkaを楽に扱いたいなあと、良い感じのRubygemsを探しました。でもよく紹介されているのはたいていRailsとの連携を前提にしているRubyGemsたちです。

目的はただ、topicをサブスクライブしてカジュアルにディスパッチがしたいんだー。というのにはちょっと重たいかなって。

そんななか、見つけたのがphobosです。コード数行からConsumerをつくれてとってもハッピーです。

- [phobos/phobos: Simplifying Kafka for ruby apps](https://github.com/phobos/phobos)

これなら(俺でも)スクリプト感覚でリアクションかけちゃうじゃないか。ということで紹介します。

## テスト用kafka-brokerの準備

このあたりから引用して、とりあえずローカルでkafka-brokerを動かします。

- https://docs.confluent.io/current/installation/docker/docs/config-reference.html

ポートやenvをちょっとアレンジ。

zookeeperと、

```bash:run-zookeeper
$ docker run -d --rm \
  -p 2181:2181 \
  --name=zookeeper \
  -e ZOOKEEPER_CLIENT_PORT=2181 \
  -e ZOOKEEPER_TICK_TIME=2000 \
  -e ZOOKEEPER_SYNC_LIMIT=2 \
  confluentinc/cp-zookeeper:5.1.0
```

brokerです。

```bash:run-broker
$ docker run -d --rm \
  -p 9092:9092 \
  --name=kafka \
  --link=zookeeper \
  -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092 \
  -e KAFKA_BROKER_ID=1 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  -e KAFKA_CONFLUENT_SUPPORT_METRICS_ENABLE=false \
  confluentinc/cp-kafka:5.1.0
```

docker-composeならこんな感じ。

```yaml:docker-compose.yml
---
version: "3.1"
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:5.1.0
    ports:
      - 2181:2181
    environment:
      - ZOOKEEPER_CLIENT_PORT=2181
      - ZOOKEEPER_TICK_TIME=2000
      - ZOOKEEPER_SYNC_LIMIT=2
  kafka:
    image: confluentinc/cp-kafka:5.1.0
    ports:
      - 9092:9092
    links:
      - zookeeper
    environment:
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://127.0.0.1:9092
      - KAFKA_BROKER_ID=2
      - KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1
      - KAFKA_CONFLUENT_SUPPORT_METRICS_ENABLE=false
```

さて、kafka-brokerの準備が整ったところで、phobosの話へ移りましょう。

## phobosセットアップ

Rubygemsのphobosをインストール。

```
$ bundle init
$ echo "gem 'phobos'" >> Gemfile 
$ bundle install --binstubs --path vendor/bundle
```

ヘルプが出るか確認します。

```
$ ./bin/phobos 
Commands:
  phobos help [COMMAND]  # Describe available commands or one specific command
  phobos init            # Initialize your project with Phobos
  phobos start           # Starts Phobos
  phobos version         # Outputs the version number. Can be used with: phobos -v or phobos --version
```

`phobos init`で初期ファイルを用意しましょう。

```
$ ./bin/phobos init
      create  config/phobos.yml
      create  phobos_boot.rb
```

この時点ですでになんか親切ですね。

## 軽くphobosの動作確認

`config/phobos.yml` に全体的な設定を記述します。

- 接続情報などグローバルな設定
- Producer/Consumerのデフォルト設定
- Consumerの購読先(複数可)とリアクション


ちょいとinitがこしらえた`config/phobos.yml`を覗いてみます。

```yaml:config/phobos.yml(part)
listeners:
  - handler: Phobos::EchoHandler
    topic: test
```

`test`というtopicを購読、`Phobos::EchoHandler`という組み込みのハンドラが処理する、という定義ですね。

前述の通りkafka-brokerがローカルに立ち上がっていれば変更は不要です、`phobos start`で起動してみましょう。

```bash:
$ ./bin/phobos start
______ _           _
| ___ \ |         | |
| |_/ / |__   ___ | |__   ___  ___
|  __/| '_ \ / _ \| '_ \ / _ \/ __|
| |   | | | | (_) | |_) | (_) \__ \
\_|   |_| |_|\___/|_.__/ \___/|___/

phobos_boot.rb - find this file at /Users/sawanoboriyu/develop/src/sandbox/qiita_ruby-phobos/phobos_boot.rb

[2019-02-03T18:11:59:345+0900Z] INFO  -- Phobos : <Hash> {:message=>"Phobos configured", :env=>"N/A"}
[2019-02-03T18:11:59:374+0900Z] INFO  -- Phobos : <Hash> {:message=>"Listener started", :listener_id=>"442f72", :group_id=>"test-1", :topic=>"test", :handler=>"Phobos::EchoHandler"}
```

最後の行から、`test`というtopicのConsumerとして稼働したことがわかります。なにか流し込んでみますか。

では、流し込みに[kafkacat](https://github.com/edenhill/kafkacat)をつかいましょう。homebrewなりで入れちゃいます。

- [edenhill/kafkacat: Generic command line non-JVM Apache Kafka producer and consumer](https://github.com/edenhill/kafkacat)

※ 実際のところ、`kafkacat`ですでにカジュアルなconsumerとして動かせるんですが、それは一旦置いときましょう。

```bash:kafkacat-1
$ echo aa | kafkacat -P -b localhost -t test
```

`Phobos::EchoHandler`は受け取ったメッセージをログに出すだけのシンプルなハンドラです。`:message=>"aa"`をちゃんと受け取ってますね。

```bash:
$ ./bin/phobos start
______ _           _
| ___ \ |         | |
| |_/ / |__   ___ | |__   ___  ___
|  __/| '_ \ / _ \| '_ \ / _ \/ __|

# -- snip --

[2019-02-03T18:12:14:481+0900Z] INFO  -- Phobos : <Hash> {:message=>"aa", :listener_id=>"442f72", :group_id=>"test-1", :topic=>"test", :handler=>"Phobos::EchoHandler", :key=>nil, :partition=>0, :offset=>0, :retry_count=>0}
```

### ハンドラを作ってmessageを処理する。

`phobos init`が作ったファイルはもう一つあります、`phobos_boot.rb`を見てみましょう。

putしかない。

```ruby:phobos_boot.rb(initial)
# Use this file to load your code
puts <<~ART
  ______ _           _
  | ___ \\ |         | |
  | |_/ / |__   ___ | |__   ___  ___
  |  __/| '_ \\ / _ \\| '_ \\ / _ \\/ __|
  | |   | | | | (_) | |_) | (_) \\__ \\
  \\_|   |_| |_|\\___/|_.__/ \\___/|___/
ART
puts "
phobos_boot.rb - find this file at #{File.expand_path(__FILE__)}

"
```

これはデフォルトでphobosが参照するエントリーポイントです。なにかするならこれに追記していく格好ですね。

では`Phobos::Handler`の代わりに使うハンドラを記述します、とりあえず`#consume`があればOKです。標準出力にputsするだけのハンドラ、`MyHandler`を次のように書いてみました。

```ruby:phobos_boot.rb(append-1)
class MyHandler
  include Phobos::Handler

  def consume(payload, metadata)
    puts 'your message is ' + payload
  end
end
```

このハンドラを使うように`config/phobos.yml`で設定を変更します。

```yaml:config/phobos.yml(part)
listeners:
  - handler: MyHandler
    # handler: Phobos::EchoHandler
    topic: test
```

`phobos start`して、裏でmessageを流し込んでみます。

```
$ ./bin/phobos start
______ _           _
| ___ \ |         | |
| |_/ / |__   ___ | |__   ___  ___
|  __/| '_ \ / _ \| '_ \ / _ \/ __|
| |   | | | | (_) | |_) | (_) \__ \
\_|   |_| |_|\___/|_.__/ \___/|___/

phobos_boot.rb - find this file at /Users/sawanoboriyu/develop/src/sandbox/qiita_ruby-phobos/phobos_boot.rb

[2019-02-03T18:45:52:276+0900Z] INFO  -- Phobos : <Hash> {:message=>"Phobos configured", :env=>"N/A"}
[2019-02-03T18:45:52:287+0900Z] INFO  -- Phobos : <Hash> {:message=>"Listener started", :listener_id=>"b59783", :group_id=>"test-1", :topic=>"test", :handler=>"MyHandler"}

# 裏で `echo aa | kafkacat -P -b localhost -t test`

your message is aa
```

やったぜ。

ちなみに`Phobos::Producer`をincludeすれば任意のtopicにメッセージを流し込んだりできます。こちらも簡単なのでやっちゃってみてください。

### 例外どうなんの？

Consumerの処理で例外があると、シンプルなBackoffアルゴリズムでランダム待ちのあと、リトライします。
ですが、ConsumerGroupの仕組み上、どこまで処理したか覚えてるようで、このmessageに対するリトライが繰り返してしまいます。
さっさとログに吐いて終了とするなり、通知したりリトライ用のトピックなどにペイロードを回しちゃうなりをしないと次を処理しないようなのでそのように。

## おわりに

これだけ簡単に扱えると、kafkaがぐっと身近になった感じがします。感覚的にはredisくらいまでカジュアル感UPです。
既存のなにかに組み込んでライブラリ的に使うのもOKで、そっちは`amqp + bunny`な感じの使い勝手かなと思います。

Phobos自体だけでもスレッドでの並列処理をパラメータ一つで行えたり、複数のlistnerを個別に取り扱えたりと便利な上に、`group_id`とかkafka側の仕組みがしっかりしているので雑にスケールできるのもよいですね。


