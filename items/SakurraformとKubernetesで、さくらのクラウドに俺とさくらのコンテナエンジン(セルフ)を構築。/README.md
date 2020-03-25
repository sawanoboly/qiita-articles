

どっかのイベントで[@wslash](https://twitter.com/wslash)さんにお会いした際、さくらのクラウドにCoreOSのイメージ入れてよ～、と言っったらいつの間にか入っており、さらに1年くらい経ちました。
ふと、Kubernetes環境が欲しくなったため、そのことを思い出して、ついでに[Sakurraform](https://github.com/higanworks/sakurraform)から構築できると楽そうだ、という過程ではじめてみました。

なお、Sakurraformについてはこの辺を見ればわかると思います。

- [スライド：さくらのインフラコード](http://www.slideshare.net/YukihikoSawanobori/ssi-sakurraform)
- [さくらのクラウド/オブジェクトストレージでInfrastructure as Codeを実現する「Sakurraform」レビュー - さくらのナレッジ](http://knowledge.sakura.ad.jp/tech/2972/)

## 必要なもの

- さくらのクラウド
    - SSHキーの登録
    - APIキー
    - クーポン(Optional)
- さくらのオブジェクトストレージ
    - バケット作成(Public)
    - APIキー(R/W)
- kubectl
    - homebrewで入れるか、[release.sh](https://github.com/kubernetes/kubernetes/tree/master/build)で。
- Ruby & Bundler

この記事の真似をして見る場合のリポジトリはこちら => https://github.com/higanworks/sakura-container-engine

### sakurraform initでcredentialsファイル作成とSSHキーIDの用意

以降、Gemなどを次のコマンドでインストールした前提で進めます。
`bundle install --binstubs --path vendor/bundle`


```
$ ./bin/sakurraform init
      create  .sakuracloud
Sakuracloud_api_token(required) ?  さくらのクラウドで発行したAPIのトークン
Sakuracloud_api_token_secret(required) ?  さくらのクラウドで発行したAPIのトークンシークレット
Sakura Object Storage buket name(optional) ?  オブジェクトストレージのアクセスキーID
Sakura Object Storage token(optional) ?  オブジェクトストレージのR/Wトークン
      create  .sakuracloud/credentials
```

SSHキーは環境変数`SAKURA_SSH_KEYID`から取得するようになっています。エクスポートします。


```
export SAKURA_SSH_KEYID="SSHキーのID"
```

direnvなどで設定しておくと良いでしょう。


## ルータとCoreOSのマシン群を作成する

ではKubernetesクラスタを組むため、マスターとノード(旧称Minion)を4台ほど調達しましょう。
statusを見ると、リソースがまだ作成されてない状態となっています。

```
$ ./bin/sakurraform status
  Nework resources
  +---------------+------------------+-------------+-------------+-------------+
  | name          | sakurraform_name | sakura_id   | subnet      | gateway     |
  +---------------+------------------+-------------+-------------+-------------+
  | defaultrouter | not created      | not created | not created | not created |
  +---------------+------------------+-------------+-------------+-------------+

  Server resources
  +--------+------------------+-------------+-------------+-------------+--------------------+
  | name   | sakurraform_name | sakura_id   | ipaddress   | status      | last_state_changed |
  +--------+------------------+-------------+-------------+-------------+--------------------+
  | master | not created      | not created | not created | not created | not created        |
  +--------+------------------+-------------+-------------+-------------+--------------------+
  | node-1 | not created      | not created | not created | not created | not created        |
  +--------+------------------+-------------+-------------+-------------+--------------------+
  | node-2 | not created      | not created | not created | not created | not created        |
  +--------+------------------+-------------+-------------+-------------+--------------------+
  | node-3 | not created      | not created | not created | not created | not created        |
  +--------+------------------+-------------+-------------+-------------+--------------------+
  | node-4 | not created      | not created | not created | not created | not created        |
  +--------+------------------+-------------+-------------+-------------+--------------------+

```

サーバ群のプランはこの様に、masterと4台のnodeです。

```plan/server.yml 
---
server:
  - name: master
    serverplan: 2001
    sshkey: <%= ENV['SAKURA_SSH_KEYID'] %>
    volume:
      diskplan: 4
      sourcearchive: 112600559854
    switch: network[defaultrouter]
    meta:
      network_offset: 3
<% 1.upto(3) do |index| %>
  - name: minion-<%= index %>
    serverplan: 2001
    sshkey: <%= ENV['SAKURA_SSH_KEYID'] %>
    volume:
      diskplan: 4
      sourcearchive: 112600559854
    switch: network[defaultrouter]
    meta:
      network_offset: <%= 3 + index %>
<% end %>
```

`plan apply`して、とりあえずPublicアクセスができるルータとCoreOS群を調達します。
ここで結構時間がかかります。Serverの調達は並行でできるようにするのがいいかもしれないと思いました。

```
$ ./bin/sakurraform plan apply
Create new network defaultrouter
[fog][WARNING] Create Router with public subnet
[fog][WARNING] Waiting available new router...
......      create  state/network/defaultrouter-02d99990-3838-0133-a639-38e8563c85ec.yml
Create new server master
[fog][WARNING] Create Server
[fog][WARNING] Create Volume
[fog][WARNING] Waiting disk until available
.[fog][WARNING] Modifing disk
Associate 133.xxx.xxx.137 to master
      create  state/server/master-0e7083f0-3838-0133-a639-38e8563c85ec.yml
Create new server node-1
[fog][WARNING] Create Server
[fog][WARNING] Create Volume
[fog][WARNING] Waiting disk until available
.[fog][WARNING] Modifing disk
Associate 133.xxx.xxx.138 to node-1
      create  state/server/node-1-84c92190-3838-0133-a639-38e8563c85ec.yml
Create new server node-2
[fog][WARNING] Create Server
[fog][WARNING] Create Volume
[fog][WARNING] Waiting disk until available
.[fog][WARNING] Modifing disk
Associate 133.xxx.xxx.139 to node-2
      create  state/server/node-2-d63ba180-3838-0133-a639-38e8563c85ec.yml
Create new server node-3
[fog][WARNING] Create Server
[fog][WARNING] Create Volume
[fog][WARNING] Waiting disk until available
.[fog][WARNING] Modifing disk
Associate 133.xxx.xxx.140 to node-3
      create  state/server/node-3-284b7560-3839-0133-a639-38e8563c85ec.yml
Create new server node-4
[fog][WARNING] Create Server
[fog][WARNING] Create Volume
[fog][WARNING] Waiting disk until available
.[fog][WARNING] Modifing disk
Associate 133.xxx.xxx.141 to node-4
      create  state/server/node-4-7ab202c0-3839-0133-a639-38e8563c85ec.yml
```


待つことしばし、全リソース出揃いました！

```
$ ./bin/sakurraform status
  Nework resources
  +---------------+----------------------------------------------------+--------------+--------------------+-----------------+
  | name          | sakurraform_name                                   | sakura_id    | subnet             | gateway         |
  +---------------+----------------------------------------------------+--------------+--------------------+-----------------+
  | defaultrouter | defaultrouter-02d99990-3838-0133-a639-38e8563c85ec | 112700766519 | 133.xxx.xxx.137/28 | 133.xxx.xxx.137 |
  +---------------+----------------------------------------------------+--------------+--------------------+-----------------+

  Server resources
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+
  | name   | sakurraform_name                            | sakura_id    | ipaddress       | status | last_state_changed        |
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+
  | master | master-0e7083f0-3838-0133-a639-38e8563c85ec | 112700766524 | 133.xxx.xxx.137 | up     | 2015-09-08Txx:xx:40+09:00 |
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+
  | node-1 | node-1-84c92190-3838-0133-a639-38e8563c85ec | 112700766531 | 133.xxx.xxx.138 | up     | 2015-09-08Txx:xx:57+09:00 |
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+
  | node-2 | node-2-d63ba180-3838-0133-a639-38e8563c85ec | 112700766540 | 133.xxx.xxx.139 | up     | 2015-09-08Txx:xx:15+09:00 |
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+
  | node-3 | node-3-284b7560-3839-0133-a639-38e8563c85ec | 112700766546 | 133.xxx.xxx.140 | up     | 2015-09-08Txx:xx:33+09:00 |
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+
  | node-4 | node-4-7ab202c0-3839-0133-a639-38e8563c85ec | 112700766558 | 133.xxx.xxx.141 | up     | 2015-09-08Txx:xx:51+09:00 |
  +--------+---------------------------------------------+--------------+-----------------+--------+---------------------------+

```

### CoreOSの更新

さて、用意されているアーカイブからCoreOSを起動すると、少々古めです。
このあとkubernetesのインストールでチャンネルはalphaに設定されますが、ひとまず最新のstableまで更新しておきたい所です。

stableの最新にする方針は、次の２通り。

1. しばらく放置する(たしかCoreOSが自ら更新する)
2. `update_engine_client`を手動で実行する。

ここは2の方式でいきます。とりあえず現状を確認してみましょう。
`sakurraform status`は`--json`オプションをつけるとJSONで出力するようにしたので、とりいそぎSSHでOSバージョンを取ってきます。

 ※ これ以降の`xargs`はOSXでのオプション(BSD)を使っているので、使っている端末によって適当に変えてください。

```
$ ./bin/sakurraform status --json | jq .Servers[].ipaddress | xargs -I {} ssh core@{} cat /etc/os-release | grep VERSION
VERSION=367.1.0
VERSION_ID=367.1.0
VERSION=367.1.0
VERSION_ID=367.1.0
VERSION=367.1.0
...
```

あらリリース`367`ですね。では同様にSSHで`update_engine_client`を気が済むまで叩きます。

```
$ ./bin/sakurraform status --json | jq -r .Servers[].ipaddress | xargs -I {} ssh -t core@{} sudo update_engine_client -update

...
...

```

それぞれ時間も取られるので、GNU parallelでやるほうがいいかもしれない。

```
$ ./bin/sakurraform status --json | jq -r .Servers[].ipaddress | parallel -j 5 ssh -t core@{} sudo update_engine_client -update
```

そのうち全部stableの最新版になっています。 (etcdが2になるので、リブートも)

```
$ ./bin/sakurraform status --json | jq -r .Servers[].ipaddress | xargs -I {} ssh core@{} cat /etc/os-release | grep VERSION
VERSION=723.3.0
VERSION_ID=723.3.0
VERSION=723.3.0
VERSION_ID=723.3.0
...
```


## coreos-cloudinit用のファイルを作成して、オブジェクトストレージにアップする

さくらのクラウドではOSの構成に、[スタートアップスクリプト](http://cloud-news.sakura.ad.jp/startup-script/)が使える... と見せかけてCoreOSには対応してません。

> 実はこれで使おうと思って先日[fog-sakuracloud](https://github.com/fog/fog-sakuracloud)をスタートアップスクリプトに対応した。まあ仕方ないね。
> 一応、Sakurraformでもyamlに`startup_sctipt`キーを追加すれば対応OSで使えます。

まあ、CoreOS相手で何するにしても、coreos-cloudinitを叩くだけになると思う。スクリプトよりだいぶ楽だし。

- [Using Cloud-Config](https://coreos.com/os/docs/latest/cloud-config.html)

折角なのでオブジェクトストレージにアップして、Public URLからを渡してみましょう。

とりあえずのKubernetes構築用に、[getting-started-guides/coreos](https://github.com/kubernetes/kubernetes/tree/master/docs/getting-started-guides/coreos) からmasterとnodeのymlを拝借します。

Sakurraformを複数スイッチ構成に対応させていないため、今回は少々大胆にも全部のコンポーネントをPublicIPでやりますわ。

次の２つのファイルがあるので、それぞれにMasterのIPアドレスを入れていきます。

```
$ ls cloud-configs
master.yaml	node.yaml
```

- master.ymlは`$private_ipv4`の記述を差し替えます。
- node.ymlは `<master-private-ip>`の部分を書き換えます。

編集が済んだら、`sakurraform storage put`でさくらのオブジェクトストレージにアップしましょう。`mput`も作ればよかったかな。

```
$ ./bin/sakurraform storage put cloud-configs/master.yaml 
File cloud-configs/master.yaml was put to cloud-configs/master.yaml.

$ ./bin/sakurraform storage put cloud-configs/node.yaml 
File cloud-configs/node.yaml was put to cloud-configs/node.yaml.
```

`sakurraform storage ls`でPublic_URLを確認します、アップできてますね。

```
$ ./bin/sakurraform storage ls
  +---------------------------+----------------+---------------------+---------------------------+------------------------------------------------------------------+
  | key                       | content_length | content_type        | last_modified             | public_url                                                       |
  +---------------------------+----------------+---------------------+---------------------------+------------------------------------------------------------------+
  | cloud-configs/master.yaml | 5562           | binary/octet-stream | 2015-09-08 xx:xx:25 +0900 | https://xxxxxx-test.b.sakurastorage.jp/cloud-configs/master.yaml |
  +---------------------------+----------------+---------------------+---------------------------+------------------------------------------------------------------+
  | cloud-configs/node.yaml   | 3755           | binary/octet-stream | 2015-09-08 xx:xx:31 +0900 | https://xxxxxx-test.b.sakurastorage.jp/cloud-configs/node.yaml   |
  +---------------------------+----------------+---------------------+---------------------------+------------------------------------------------------------------+
```


## KubernetesのMasterを構築

マスターのIPは手抜きしてこんな感じでとりましょう。

```
$ ./bin/sakurraform status --json | jq -r .Servers[0].ipaddress
133.xxx.xxx.137 (MasterのIPアドレス)
```

では`coreos-cloudinit`に渡します。何かと連携させようかと思いましたが、これもsshでいいか。

```
$ ssh core@133.xxx.xxx.137 sudo coreos-cloudinit --from-url https://xxxxxx-test.b.sakurastorage.jp/cloud-configs/master.yaml 

Checking availability of "url"
Fetching user-data from datasource of type "url"
2015/09/08 xx:xx:24 Fetching data from https://xxxxxx-test.b.sakurastorage.jp/cloud-configs/master.yaml. Attempt #1
Fetching meta-data from datasource of type "url"
Merging cloud-config from meta-data and user-data

-- 中略

2015/09/08 xx:xx:43 Result of "start" on "kube-scheduler.service": done
2015/09/08 xx:xx:43 Calling unit command "stop" on "locksmithd.service"'
2015/09/08 xx:xx:43 Result of "stop" on "locksmithd.service": done
2015/09/08 xx:xx:43 Calling unit command "restart" on "update-engine.service"'
2015/09/08 xx:xx:43 Result of "restart" on "update-engine.service": done
```

終わったら、ひとまず`kubectl`でAPIと通信してみます。

```
$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 version
Client Version: version.Info{Major:"1", Minor:"1+", GitVersion:"v1.1.0-alpha.1.388+bb3e20e361764c", GitCommit:"bb3e20e361764cf0af8d23671b7fe00f0951b0f3", GitTreeState:"clean"}
Server Version: version.Info{Major:"1", Minor:"0", GitVersion:"v1.0.3", GitCommit:"61c6ac5f350253a4dc002aee97b7db7ff01ee4ca", GitTreeState:"clean"}
```

Serverの情報とれてますねー。APIが動きました。

## KubernetesのNodeを構築

NodeたちのIPは並びが丁度いいので、これでよしとします。

```
$ ./bin/sakurraform status --json | jq -r .Servers[1:][].ipaddress
133.xxx.xxx.138
133.xxx.xxx.139
133.xxx.xxx.140
133.xxx.xxx.141
```

いっぺんにやりましょうか。今度は`node.yml`を渡します。

```
$ ./bin/sakurraform status --json | jq -r .Servers[1:][].ipaddress | parallel -j 4 ssh core@{} sudo coreos-cloudinit --from-url https://xxxxxx-test.b.sakurastorage.jp/cloud-configs/node.yaml


-- 省略。。。

```

終わったら、`kubectl get nodes`をしてみます。

```
$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 get nodes
NAME              LABELS                                   STATUS    AGE
133.xxx.xxx.138   kubernetes.io/hostname=133.xxx.xxx.138   Ready     5m
133.xxx.xxx.139   kubernetes.io/hostname=133.xxx.xxx.139   Ready     5m
133.xxx.xxx.140   kubernetes.io/hostname=133.xxx.xxx.140   Ready     5m
133.xxx.xxx.141   kubernetes.io/hostname=133.xxx.xxx.141   Ready     5m
```

うん、Nodeがぶら下がりましたね。


## Exampleから動作確認

では動作の様子をみるため、`examples/simple-nginx`からコマンドを実行してみよう。

```
$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 run my-nginx --image=nginx --replicas=2 --port=80
replicationcontroller "my-nginx" created
```

podsが作られた。

```
$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 get pods
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-uabgm   1/1       Running   0          1m
my-nginx-vt40g   1/1       Running   0          1m
```

replicationcontrollerも動いているね。

```
$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 get rc
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR       REPLICAS   AGE
my-nginx     my-nginx       nginx      run=my-nginx   2          1m
```

サービス用のIPについてノープランだったので、次のexposeは無理な気がします。とりあえずスケールでもしてみましょう。

```
$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 scale --replicas=20  rc my-nginx
replicationcontroller "my-nginx" scaled


$ ./k8bin/kubectl -s http://133.xxx.xxx.137:8080 get pods
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-0hs7n   1/1       Running   0          1m
my-nginx-3uoi8   1/1       Running   0          1m
my-nginx-50tal   1/1       Running   0          1m
my-nginx-7v1in   1/1       Running   0          1m
my-nginx-8uwhd   1/1       Running   0          1m
my-nginx-adkk5   1/1       Running   0          1m
my-nginx-blje1   1/1       Running   0          1m
my-nginx-coent   1/1       Running   0          1m
my-nginx-hs3wq   1/1       Running   0          1m
my-nginx-hy3rk   1/1       Running   0          1m
my-nginx-i5h4x   1/1       Running   0          1m
my-nginx-jcbkl   1/1       Running   0          1m
my-nginx-m0xd9   1/1       Running   0          1m
my-nginx-monyg   1/1       Running   0          1m
my-nginx-oqy92   1/1       Running   0          1m
my-nginx-p22xa   1/1       Running   0          1m
my-nginx-t8z8r   1/1       Running   0          1m
my-nginx-uad1i   1/1       Running   0          7m
my-nginx-uk2q5   1/1       Running   0          1m
my-nginx-w089w   1/1       Running   0          7m
```

やった、増えたね。

まあまあ楽にKubernetesの環境を作れました、さくらのコンテナエンジン(セルフタイプ)は8割がた出来たと言えましょう。


## 次と課題

認証とか管理系のIPアドレスとか常識的な範囲はおいといて、Public(Service)IPの戦略が必要だ。

- GCEでは特に気にしなくてよかったことなので、まだよく調べていない。
- Local系や、他のクラウド向け構築のドキュメントを参考によしなに設定したい。

さくらのクラウドはスイッチを作ってバーチャルサーバにプスプスできる、割と変態仕様なので結構好きな様にサービスIPを組めるかもしれない(LBは組み込めるかわからないけど)。
その辺を踏まえてKubernetesの知見をお持ちの方から意見など伺えるといいなあ。
