
ただ、`--squash`をしたかった。

CircleCI2.0では実行環境(Executer)を2種類から選択できる。指定イメージをつかう`docker`と1.0のような環境の`machine`。
dockerデーモンの挙動に手を入れられるのは`machine`のほうだけなので、環境に`machine`を指定。この時点でdockerは`17.03.0-ce`だったので、`--squash`が使える。おそらくもっと新しいバージョンにアップグレードして使うこともできるでしょう。


サンプルで使用したDockerfileはこちら。レイヤに差分がないと`--squash`がコケるので軽く変更します。

```Dockerfile
FROM alpine:latest

RUN apk update
ADD Dockerfile /Dockerfile
CMD ["/bin/true"]
```

## `.circleci/config.yml`のサンプル

最初のstepで`--experimental=true`を追記してリスタート。

```yaml:.circleci/config.yml
version: 2
jobs:
  build:
    machine: true
    working_directory: /home/circleci/circleci-sandbox-bb
    steps:
      - checkout
      - run:
          name: Enable experimental
          command: |
            set -x
            echo 'DOCKER_OPTS="--experimental=true"' | sudo tee -a /etc/default/docker
            # sudo DEBIAN_FRONTEND=noninteractive apt-get update && sudo apt-get install -y docker-engine ## ついでにdockerデーモンをaptリポジトリの最新にしたい時
            sudo service docker restart
      - run: |
          TAG=0.1.$CIRCLE_BUILD_NUM
          docker build -t local/circleci-sandbox-bb:$TAG --squash .
          docker image history local/circleci-sandbox-bb:$TAG
```


`image history`の実行結果で`--squash`による寄せができていることを確認。

```shell:circleci-outputs
.
.
.
Successfully built 6d611d5da4fc
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
2de9d6caee54        1 second ago                                                        1.03 MB             merge sha256:6d611d5da4fcfe515263c628140ae8a25a4b870881b8da9f88f9130caf44ae3c to sha256:02674b9cb179d57c68b526733adf38b458bd31ba0abff0c2bf5ceca5bad72cd9
<missing>           1 second ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0 B                 
<missing>           1 second ago        /bin/sh -c #(nop) ADD file:00457989c9c63fa...   0 B                 
<missing>           1 second ago        /bin/sh -c apk update                           0 B                 
<missing>           37 hours ago        /bin/sh -c #(nop)  CMD ["/bin/sh"]              0 B                 
<missing>           37 hours ago        /bin/sh -c #(nop) ADD file:63f63606d6e289e...   3.99 MB            
```

machine-executerは起動がちょっと遅いのと、ちょいちょい環境の変化がありそうなのであまり常用したくはないけど、イメージのビルド用途なら十分。
