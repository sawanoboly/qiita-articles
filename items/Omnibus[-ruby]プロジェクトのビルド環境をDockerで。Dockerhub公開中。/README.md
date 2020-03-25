

以前、[Omnibus-rubyプロジェクトでツールの周辺依存をまるごとrpm,debに固める - Qiita](http://qiita.com/sawanoboly/items/a2c258a235824b91b70f "Omnibus-rubyプロジェクトでツールの周辺依存をまるごとrpm,debに固める - Qiita")を書きました。
当時のOmnibus3.xから、この記事時点では4.xになって、書式は同じですが(デフォルトの)ビルド環境ツールチェーンがVagrantからTest-Kitchenに変わるなどしてます。

うーん、要は`omnibus build $PROJECTNAME`が走ればいいんでしょー。じゃあDockerでいいやと言うわけで次のようにしました。

- https://github.com/OpsRockin/omnibus_base_ubuntu14
- https://github.com/OpsRockin/omnibus_base_centos6

使い方はOmnibusプロジェクトのルートで、次のように`ほぼカラッポのDockerfile`をつくって、

```Dockerfile
FROM opsrock/omnibus_base_centos6
MAINTAINER you
```

buildしてrunすれば`./pkg/`の下にdeb,rpmをポイッと吐いておわり。

```shell:
$ docker build -t omnibus_myproject-centos6 -f Dockerfile.centos6 .
$ docker run -e OMNIBUS_PROJECT=serverspec -v pkg:/home/omnibus/omnibus-project/pkg omnibus_myproject-centos6
```


以下はしくみやきっかけなどの話、読みたい人だけ。

## Omnibusの一生 (イニシャルから、パッケージをつくるまで)

1. VMを作成する
1. Chef(Test-Kitchenなど)でVM内に`omnibus`コマンド用の環境を構築
1. Omnibusプロジェクトの設定に従い、パッケージ用の各種プロダクトをビルドする
1. パッケージに固める

これ、一番安定する方法としては3を実行済、または開始できる状態にあるVMを保持しておくことです。
2までの処理はそれなりに時間がかかるし、3以降もライブラリがバージョンでキャッシュされるから変更がない部分は何度もビルドが走らなくておトク。

ただ個人的にそういったビルドサーバの維持は好かないので、とりあえず2までをDockerイメージですぐ使えるようにすりゃイイやと。

3以降も一工夫いるけど、キャッシュの使い回しもなんとかできそうです。CIでキャッシュ持ってリストアするかは、プロジェクトによって選びましょう。

## Dockerfile

大したことはしてないけどもDockerfileの説明。他所のイメージ使う時は、まずDockerfile見ないと気がすまないよね？

```Dockerfile
FROM centos:6
MAINTAINER sawanoboriyu@higanworks.com

RUN yum install curl tar -y

## cookbookのomnibusを実行(2のステップ)するための下準備
RUN mkdir /root/chefrepo
ADD files/Cheffile /root/chefrepo/Cheffile
WORKDIR /root/chefrepo

## Chefでレシピ omnibus::default を適用する。
## Chef(Omnibus)およびsrcが残るとイメージのサイズがでかくなるので同じ行で消す
RUN eval "$(curl chef.sh)" && \
    /opt/chef/embedded/bin/gem install librarian-chef --no-ri --no-rdoc && \
    /opt/chef/embedded/bin/librarian-chef install && \
    chef-client -z -o "omnibus::default" && \
    rm -rf /opt/chef /root/chefrepo /root/.chef /root/.ccache /usr/local/src/*

## どうせomnibus(Gem)が入るので、一旦いれちゃう
WORKDIR /root
ADD files/Gemfile /root/Gemfile
ADD files/prebundle.sh /root/prebundle.sh
RUN ./prebundle.sh

ADD files/bash_with_env.sh /home/omnibus/bash_with_env.sh
ADD files/build.sh /home/omnibus/build.sh

ENV HOME /home/omnibus

## docker build時にプロジェクトディレクトリを登録、差分bundle
ONBUILD ADD . /home/omnibus/omnibus-project

WORKDIR /home/omnibus/omnibus-project
ONBUILD RUN bash -c 'source /home/omnibus/load-omnibus-toolchain.sh ; bundle install --binstubs bundle_bin --without development test'
ONBUILD RUN echo "Usage: docker run  -it -e OMNIBUS_PROJECT=${PROJECT_NAME} -v pkg:/home/omnibus/omnibus-project/pkg builder-centos6"

CMD ["/home/omnibus/build.sh"]
```

cookbookのomnibusとbundlerのおかげで、OS毎に変更がほとんどいらないので楽だ。

大元のomnibus(Gem)とomnibus-software(Gem)が更新されたらこれらのlatestも更新しておく方針です。ただ、omnibus-softwareとかタグ切らねーのでmaster追うしかなさげ。


## Omnibus-Serverspecの継続リリース

さて、このOmnibusをつかって個人的にomnibus-serverspecというパッケージを配っていたりします。クローズドなら他にも少し。

https://github.com/OpsRockin/omnibus-serverspec

これまではCircleCIからDigitalOceanのDroplet(4コア)を2つ作ってパッケージを作っていたため、自分が用事のあるときだけ適当に更新していました。DOにかかっていた費用は月額1,000円もいかない程度。
『とりあえずServerspecでテストしたい、なんとかして！』みたいなお仕事で重宝したり。

しかしこういう記事が登場。

[packer-provisioner-serverspec というPackerプラグインを書いた - ローファイ日記](http://udzura.hatenablog.jp/entry/2015/09/25/194640 "packer-provisioner-serverspec というPackerプラグインを書いた - ローファイ日記")

> Serverspec自体のインストールは omnibus を使っています。

Omnibus-Serverspecが気まぐれ更新のままではマズイと思ったので、CircleCI内だけで完結させるべくDockerのみでパッケージを作るようにしたのが今回の発端です。

で、[mizzy](https://github.com/mizzy)さんのご厚意でServerspecおよび、Specinfraのリリース時、ついでにbuildを叩くWebHookを追加していただきました。これで自動的にパッケージがリリースされ、古いやつからローテーションで消えていきます。

![ご依頼内容](https://qiita-image-store.s3.amazonaws.com/0/7454/9e78beb2-7a9e-e922-412a-ee679c8351f7.jpeg "webhook.jpg")

呼ばれるのはこれ： [https://circleci.com/gh/OpsRockin/omnibus-serverspec](https://circleci.com/gh/OpsRockin/omnibus-serverspec)

人様のプロダクトをビルドするようなキックは、自動にした所でプロダクトによってはまともに機能しませんが、リリース当初からインストール方法が変わらないことに定評があるServerspecならその点ほとんど心配いりません。


> 注意：
> Omnibus-Serverspecのパッケージ自体は非公式であり、動作保証も特にしていません。Omnibus-Serverspecで挙動がおかしいとか動かないとかで、[mizzy](https://github.com/mizzy)さんに問い合わせたりはしないでね。
> リリースイベントをわりかし早く、ナイスなプルリクをした人の次(直後)に教えてもらっているだけという関係です。
>
> そのうち、パッケージのPush前に軽く結合テスト(これもDockerでOK)を入れとこうとは思います。

ちなみに、GithubのリリースなどのURLって、`.atom`つければフィード形式で拾えるんですよ。

- [https://github.com/mizzy/serverspec/commits/master.atom](https://github.com/mizzy/serverspec/commits/master.atom)
- [https://github.com/mizzy/serverspec/releases.atom](https://github.com/mizzy/serverspec/releases.atom)

omnibus本体などの更新は、このフィードからDockerhubのビルドをキックするようにしとけばいいかな。
