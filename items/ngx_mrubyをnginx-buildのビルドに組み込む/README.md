
[nginx-build](https://github.com/cubicdaiya/nginx-build)っちゅう、nginxのビルドをラップしてくれるツールがあります。
広めの想定範囲内で柔軟に対応できて、環境依存度が比較的高めのライブラリを自動で取ってくるなど、ノウハウが詰まっててよいものです。

で、導入に少しコツがいる、[ngx_mruby](https://github.com/matsumoto-r/ngx_mruby)を組み込む場合について書いておきます。

これを書いた時点の各々のバージョンはこちら。

- nginx-build: v0.7.1
- ngx_mruby: v1.17.0

本記事の内容は、このリポジトリで使ってます。 https://github.com/giraffi/docker-nginx-mruby-base


> 余談。`ngx_mruby x nginx-build`はわりと背徳の香りがする組み合わせです。理解できる方はほんの一部でしょうが。。。

## nginx-buildの3rdパーティーモジュール定義

nginx-buildはiniファイルに3rdパーティモジュールを定義します。
ngx_mrubyはこんな感じに、shprov指定による前処理がほぼ必須です。


```modules3rd.ini
[ngx_mruby]
form=git
url=https://github.com/matsumoto-r/ngx_mruby.git
rev=v1.17.0
shprov=/config/mruby/wrap_build.sh
```

ここのformがどうやらformatのことで、fromと誤認してたまにハマる。


## ngx_mrubyの(フックされる)前処理

前処理の内訳例です。nginxをビルドする前にこれらが終わっていないといけません。

- mgemの塩梅を調節
- mrubyをビルドする

ざっくりshellにして、shprov指定の内容がこちら。元の`build.sh`を参考にしています。

```wrap_build.sh
#!/bin/sh

set -xe

## 自前のbuild_config.rbに差し替えたい時
/bin/cp -f /config/mruby/build_config.rb ./
./configure --with-ngx-src-root=../nginx-$NGINX_VER

make build_mruby -j 2
make generate_gems_config -j 2 # これがほんとに必要かはちゃんと調べてない。
```

## nginx-buildが使うconfigureへの仕込み

さて、`ngx_mruby`では指定バージョンの`ngx_devel_kit`も`--add-module`する必要がある(っぽい)。
これをiniで賄うのは面倒なので、configureのオプションに加えておきます。幸いパスは固定です。

```configure.sh
./configure --with-debug \
 --prefix=/usr/local/nginx \
 --with-pcre-jit \

... (中略)

 --add-module=../ngx_mruby/dependence/ngx_devel_kit
```

`ngx_mruby`本体の`add-module`は`nginx-build`が自動でつけてくれます。


## nginx-build!

準備ができたら、ビルドします。

```
$ export NGINX_VER=1.9.12
$ ./nginx-build -verbose -v $NGINX_VER -d work -pcre -zlib -m modules3rd.ini -c configure.sh --clear
```

> (nginx-build 0.7.2以前) しばらく眺めていればおわります。今のところshprov(provideShell)はexitが0以外でも容赦なく進んでいくようなので、`-verbose`をつけて経過を眺めておくのが無難でした。

v0.7.3以降のnginx-buildを使えば、shprovのexit_statusが0出ない場合Failするので安心です。(コメント参照)

あとは`make install`するなりなんなりと。
