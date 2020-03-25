

walterはこちら。 [シンプルなパイプラインツールwalterを少しだけ試してみた - Qiita](http://qiita.com/sawanoboly/items/0b2afedd7949cd045e82 "シンプルなパイプラインツールwalterを少しだけ試してみた - Qiita")

## testやdeploymentにwalterだけ書けばいいようにできるかな

CircleCIヘルプののcustom softwareを参考に。

[Install custom software - CircleCI](https://circleci.com/docs/installing-custom-software "Install custom software - CircleCI")

全容はこちら。

https://github.com/OpsRockin/the_walter_visits_with_circleci


## circle.yml

インストールしたらキャッシュしよう。

```circle.yml 
---
dependencies:
  pre:
    - bash ./ci/install-walter-0.1.0.sh
  cache_directories:
    - "~/walter"
test:
  override:
    - walter
```

## dependencies-preの中身

`~/bin/`にリンクを張ればパスが通るのでそうしてみた。

``` ci/install-walter-0.1.0.sh 
set -x
set -e
W_VERS=0.1.0
W_DIR=walter_linux_amd64

mkdir -p ~/walter/${W_VERS}

if [ ! -e ~/walter/${W_VERS}/${W_DIR}/walter ]; then
  wget https://github.com/walter-cd/walter/releases/download/v${W_VERS}/walter_linux_amd64.tar.gz
  tar xvzf walter_linux_amd64.tar.gz -C ~/walter/${W_VERS}
fi

sudo ln -sf ${HOME}/walter/${W_VERS}/${W_DIR}/walter ${HOME}/bin/walter
```

## walter動くかな

動いた。Golangめ。

![OpsRockin_the_walter_visits_with_circleci__7_-_CircleCI.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d0953112-2b62-8eec-dfa3-8f29670fc760.png "OpsRockin_the_walter_visits_with_circleci__7_-_CircleCI.png")

今のところShellを並べるのと使い勝手や取り回しやすさは変わらないけど、少しだけおしゃれだ。
