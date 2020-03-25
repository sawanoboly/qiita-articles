CircleCIの関係者っぽい方からなんか書いてとDMがきたので。そういえば今年はチャットサポートで問題さくっと解決してもらったりしたなあと思い、特に素性も確認することなく記事を書くことにしました。

さて、CircleCIでは毎日なにかを実行してるくらい使ってはいますが、ほぼドキュメントに普通にある程度のことしかやっていません。
とりあえず、すこし変わった使い方かな？程度の小ネタのようなのを2つほど。

## ネタ1: 親リポジトリの更新でCircleCIでワークフロー実行

私はプロビジョニングツール[Chef](https://www.chef.io)のプラグイン[knife-zero](https://knife-zero.github.io)というRubygemのメンテナをしています。

このような、元プロジェクトが別にある場合、その変更によってプラグインの互換性が崩れることがありますね。実際何度かあり、ユーザから『アップデートしたら止まった』という報告で修正したこともあります。

この修正作業、まず前バージョンとの差分を確認してからどのようにプラグインを変更するかなどを考える必要があり、非互換が複数にわたっている場合などなかなか骨の折れる作業になることも。

とりあえず結合テストを書いて、1日1回でスケジュールを回すことにしましたが、Chef側の規模が規模だけに、1日単位でもどの変更か調べるのが大変な時もありました。


### 互換性維持のために

ならもう親プロジェクトのmasterのHEAD更新のたび、結合テストしたらいいんじゃないかな。Githubはブランチのコミットログに対してフィードを生成しているので、それをつかいます。

ということで、IFTTTでこんな感じのAppletを作りました。

![My Applets - IFTTT 2019-12-13 18-34-25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/7454/c02a7081-20a0-561c-bbf4-9cecd745725f.png)


- Feed URL: `https://github.com/chef/chef/commits/master.atom`
- URL: `https://circleci.com/api/v1.1/project/github/higanworks/knife-zero/build?circle-token=(プロジェクトトークン)`
- Method: `POST`
- body: `{"branch": "integration_testedge"}` (application/json)

これで、Chef側のすべてのmaster変更に対して結合テストを実施できるようになります。そのため、非互換の変更が加えられたのがどのコミットなのかがバッチリ特定できます。

この手法をとるようになってからは、非互換が発生した変更に対応するようにプラグイン側を修正し、結合テストをグリーンに復旧させて、可能なら新旧両方サポートした分岐なりでプラグインの新バージョンを先にリリースしてあります。
実際にバージョンが付けられてリリースされるまでに対応すればよいので、事後にするより余裕を持って見られるのもよいですね。

実際にこの結合テストをやっているのは下記。

[https://circleci.com/gh/higanworks/knife-zero](https://circleci.com/gh/higanworks/knife-zero)

masterが活発だと、結構なペースでテストが走っていますね。

![CircleCI 2019-12-13 18-38-04.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/7454/46c5510b-15b0-3b26-9963-34ca4ff094d6.png)


ビルド番号はもう13,000以上。。

ともあれ、プラグインを作るなら、親プロジェクトが新バージョンをリリースする際、その前段階から互換性維持対応は終わらせておく、という状況を維持したいもんですね。

----

> 追記: せっかくなので機能の追加要望としては、よそのリポジトリをsubscribeするような形でビルドのトリガーにできる機能があったらうれしいな。

## ネタ2: systemdサービスで起動するrpmパッケージのテスト

私はAmazon Linux1/2用に、Nginxのmainline版のRPMをビルドしています。

- [Nginx mainline for Amazon Linux](https://github.com/OpsRockin/nginx_mainline_for_amazon_linux)

主な用途はWordPressの[Amimoto AMI](https://ja.amimoto-ami.com/)ですが、普通にそのへん気にしなくても使えるようにしています。

で、このRPMは`ngx_mruby`や`ngx_pagespeed`などダイナミックモジュールを入れているので、ちょっとした結合テストをしてからリリースすることにしています。

Amazon Linux 2はサービスの管理がsystemdになりましたが、パッケージを作る際にちょっと手抜きして、Sysv initで作っていたAmazon Linux 1から流用しました。(結合テストめんどくさそうだったし。。)

しかし、Amazon Linux 2に対応してる企業一覧、みたいなのがありまして。

[テクノロジーパートナー | Amazon Linux 2](https://aws.amazon.com/jp/amazon-linux-2/)

ここには『サービス全部`systemd.unit`で管理できるようにしてるぜ！』などの規定があったりするのです。

ああ、Amazon Linux 2用のほうは変更して、テストしなきゃなあと。

> 追記： 無事に載ってましたわ。

### machine executorでsystemd on docker container

dockerコンテナでsystemd配下としてサービスを起動するには `--privileged` で `/sbin/init` からエントリーすればよいと色んなとこに書いてあるのでそうするとしましょう。
CircleCIでこれをやるにはおそらく現状`machine executor`しかないのかな。

1. systemdつきでコンテナ起動
2. RPMインストール
3. curl

というテストを作るためのstepがこんな感じに。

```yaml:.circleci/config.yml(抜粋)
    steps:
      - run:
          name: Set up host system
          command: |
            sudo service apparmor teardown
      - run: docker run --name << parameters.pkg >> -d -e CIRCLE_JOB=$CIRCLE_JOB -it --privileged -p 8080:80 -v `pwd`:/mnt/host --workdir /mnt/host opsrock/amzn_linux_with:2-with-sources /sbin/init
      - run: sleep 10
      - run: docker exec << parameters.pkg >> ./ci/build_deps.sh
      - run: docker exec << parameters.pkg >> rpm -ivh ./RPMS/x86_64/nginx-*.<< parameters.pkg >>.amimoto.x86_64.rpm
      - run: docker exec << parameters.pkg >> cp -f ./ci/nginx.conf /etc/nginx/nginx.conf
      - run: docker exec << parameters.pkg >> nginx -V
      - run: docker exec << parameters.pkg >> nginx -t
      - run: docker exec << parameters.pkg >> systemctl start nginx
      - run: curl -vf 127.0.0.1:8080/hello
```

RPMファイル自体はひとつ前のワークフローでビルドしてるので、実際は `persist_to_workspace` => `attach_workspace` で運んでいます。
machine上のdockerでやるときの注意は、`CIRCLE_*`の環境変数があるつもりでやりがち、くらいでしょうか。
sleepのとこは妥協なので、どこか確認しやすい場所があれば`docker exec`で untilのループを回すほうが良いと思います。

余談ですが、CircleCIのdocker imageに`amazonlinux:2*`を選択する場合、`tar`と`gzip`が入ってないので下記メッセージでコケてしまいます。rh系8のminimumもそうだったかも。

```
> tar utility is not present in this image but it is required. Please install it to have workflow workspace capability.

> gzip utility is not present in this image but it is required. Please install it to have workflow workspace capability.
```

なのでそれをカバーしただけのイメージを作っておく必要がありますね。

[OpsRockin/amzn_linux_with](https://github.com/OpsRockin/amzn_linux_with)

こちらも確かなるべく最新を維持するため、IFTTTから毎日イメージ作成をキックしています。

[https://hub.docker.com/r/opsrock/amzn_linux_with/builds](https://hub.docker.com/r/opsrock/amzn_linux_with/builds)

## おわりに: サポート良い対応

今年からかな？CircleCIのサポート(Appの右下Chat枠)が日本語で対応してくるようになりました。
英語のみだったときから普通に対応は良かったので、先日質問して反応が返ってきたときに、『えっ日本語サポートかいな』と若干不安になったのを覚えています。

そんな心配は杞憂におわり、サポートの方は普通に各機能やAPIについても理解しているようで、プレビュー版APIの挙動について聞いたり、APIv2について教えてくれたりと頼りになる対応でした。

FAQなんかはすぐ解決できるでしょうし、トークンの権限とかも普通に素早く調べてくれたりしていたので多少の固有問題にもエスカレーション体制ができているんだろうなと思います。

なにか迷ったらチャットで聞いちゃいましょう。
