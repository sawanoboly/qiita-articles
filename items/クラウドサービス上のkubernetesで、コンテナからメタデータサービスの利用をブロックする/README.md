
kubernetes(以下k8s)は自前のサービスを展開するインフラストラクチャとして便利です。特にクラウドサービスで動いてもらうと色々とやりたい放題。

一方、管理下のコンテナ内にユーザが自由にコンテンツを配置・変更できるようなサービス(※)を展開しようと思うと、ユーザに提供するコンテナからメタデータサービスにアクセスできると大変困ります。

(※)CIとか、ホスティングとかですね。

この心配事はk8sに限った話ではないので、前にも単品のDockerホストで似たようなことをやりました。

[#d4aのmobyカスタマイズ例 / Docekr for AWS/Azure のベース、Moby Linuxを素の状態で起動してSSHログインする。 - Qiita](https://qiita.com/sawanoboly/items/d626c01d86586f79c83a#d4aのmobyカスタマイズ例)

デフォルトでブロックしてしまおうか？という話もあったようですが、今の所(k8s 1.8.x)メタデータアクセスは許可しています。

- [Filter/restrict access to http://metadata from containers · Issue #8867 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/issues/8867)

議論の中で、『(意訳)特定のプロバイダのメタデータサービスに依存するのはダメよね』ともあります。可搬性を考えるとそうなんですよね。


## k8sメタデータブロックの概要

しくみは単純です。

- `network=host`, `privileged=true `のDaemonSetを使う
- iptablesのPREROUTINGで169.254.169.254宛てのルールを適用
- (オプション)間に入るHTTPサーバを用意してダミーのレスポンスを返却または普通のメタデータサービスにプロキシする

実際はこのiptablesはホストOSで直接適用してもよいのですが、k8sらしくDaemonSetを使って汎用的にしましょうねーという感じです。


## k8sメタデータブロックの実装



- 汎用的に使えるnginxプロキシ https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/metadata-proxy
- GCP専用のgo言語サーバ https://github.com/GoogleCloudPlatform/k8s-metadata-proxy


汎用nginxのほうで使われるコンテナイメージ`gcr.io/google-containers/metadata-proxy`から、CMDに適用される内容をみてみます。



```shell:/start-proxy.sh

#!/bin/dash

# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

_term() {
  iptables -D -t filter -I KUBE-METADATA-SERVER -j ACCEPT
  iptables -D -t nat -I PREROUTING -p tcp -d 169.254.169.254 --dport 80 -j DNAT --to-destination 127.0.0.1:988
  exit
}

# Forward traffic to nginx.
iptables -t nat -I PREROUTING -p tcp -d 169.254.169.254 --dport 80 -j DNAT --to-destination 127.0.0.1:988
iptables -t filter -I KUBE-METADATA-SERVER -j ACCEPT

# Clean up the iptables rule if we're exiting gracefully.
trap _term TERM

# Run nginx in the foreground.
nginx -g 'daemon off;'
```

これでnginxにTERMちゃんと渡るんだっけ。。とかPREROUTINGの削除ってこれでいいんだっけ。。などと思いつつ、PREROUTINGでメタデータへのリクエストをダミーのnginxに向けている様子がわかります。

KUBE-METADATA-SERVERは環境次第であったりなかったりです。
ここらを見ると、GCPを使う場合には環境変数でメタデータブロックだけならできそうなので、プロキシ有効時にはそれを開くためでしょう。

https://github.com/kubernetes/kubernetes/blob/master/cluster/gce/gci/configure-helper.sh#L55

KUBE-METADATA-SERVERがない環境でもコケはしないので、その場合は気にしないでよさそう。

次は間に挟むnginxのコンフィグ。ブロックしたり条件次第でproxy_passで本来のメタデータサービスにプロキシしたりです。

```nginx./conf
# ... 省略

    http {
      access_log /dev/stdout;
      server {
        listen 127.0.0.1:988;
        # When serving 301s, don't redirect to port 988.
        port_in_redirect off;
        # By default, return 403. This protects us from new API versions.
        location / {
            return 403 "This metadata API is not allowed by the metadata proxy.";
        }


        # Allow for REST discovery.
        location = / {
            if ($args ~* "^(.+&)?recursive=") {
                    return 403 "?recursive calls are not allowed by the metadata proxy.";
            }
            proxy_pass http://169.254.169.254;
        }

# ... 省略
```

ここのルールを色々変更すれば、任意の情報だけコンテナから参照を許可する、なども可能ですね。認可だけは横取りされないようにすれば大丈夫でしょうきっと。

> 追記:
> 仕様変更があって、制限したいノードに `beta.kubernetes.io/metadata-proxy-ready=true` のラベルが必要になりました。

## k8sメタデータブロックの様子

実験はAzure Container Service (AKS)で。

プロキシコンテナ稼働前、Nodeで`docker run`からメタデータを取得。

```
$ docker run alpine:latest /bin/sh -c 'apk add curl --no-cache >/dev/null; curl -s -H Metadata:true "http://169.254.169.254/metadata/instance?api-version=2017-04-02&format=text"'
location
name
offer
osType
platformFaultDomain
platformUpdateDomain
publisher
sku
version
vmId
vmSize
```


プロキシコンテナ稼働後、Nodeで`docker run`からメタデータ取得をnginxで横取り。

```
$ docker run alpine:latest /bin/sh -c 'apk add curl --no-cache >/dev/null; curl -s -H Metadata:true "http://169.254.169.254/metadata/instance?api-version=2017-04-02"'

This metadata API is not allowed by the metadata proxy.

```

なおこの仕組みでは、ネットワークモードが`host`のコンテナは対象外です。

```
# プロキシコンテナ稼働後でも--net=hostはメタデータが取れる
$ docker run --net=host alpine:latest /bin/sh -c 'apk add curl --no-cache >/dev/null; curl -s -H Metadata:true "http://169.254.169.254/metadata/instance?api-version=2017-04-02&format=text"'
compute/
network/
```

当初の目的は達成したので一旦これでよしとします。


## ほか、関連情報

クラウドサービス上のVMをDockerなどコンテナのホストに使うと、コンテナからもVMへの認可情報を使えちゃう懸念は結局よくある話で、そもそもメタデータをブロックしたいのもそれが使えると困るからというのが大半です。(他にはuser-dataが見れるとマズい、など。)

この辺りは次のようなアプローチもあるので、利用できるならこういうのも。

- [タスク用の IAM ロール - Amazon EC2 Container Service](http://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-iam-roles.html)
  - コンテナ専用のメタデータサービスからコンテナの用途にあわせた認可情報を取得
  - 通常のメタデータはブロック
- https://github.com/dump247/ec2metaproxy
  - コンテナからのメタデータアクセスを横取りして、事前に用意した認可情報に差し替える

ただブロックするよりは、うまいこと利用しながら権限をコントロールする方法が用意されているなら、特定のプロバイダ依存にはなるかもしれませんが利用してもよいかなと思います。
