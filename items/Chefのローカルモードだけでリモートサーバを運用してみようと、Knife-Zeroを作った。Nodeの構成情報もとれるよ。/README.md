
Chef([ChefInc](http://www.getchef.com/))の管理ツールKnifeのプラグインで、[Knife-Zero](https://github.com/higanworks/knife-zero)というのを作りました。

[https://github.com/higanworks/knife-zero](https://github.com/higanworks/knife-zero)

> 追記： バージョンアップして、knife zero chef_client/convergeサブコマンドを追加しました。

> 追記： ひと通りの機能を実装したので、knife-zeroのことをまとめるドキュメントをゆるやかに作成しています。
>  https://knife-zero.github.io 

端的にいうとAnsibleのやり方をパクりつつ、Chef-Serverから構成管理を含む機能全部を頂戴しながら本体の管理を捨てました。
Knife-ZeroとChefのローカルモードを使うと、手元のChef-Repoだけで状態込みのサーバインフラ管理が完結するので色々と楽ができそうです。

参考: [About the chef-repo](http://docs.getchef.com/essentials_repository.html)

## Knife-Zeroの特徴

ほとんど最近追加されたChefのローカルモードの恩恵ですけど、だいたいこんな感じです。

- リモートサーバにKitchen(CookbookやRole等のChefリソース)をSCPやRsyncなど個別に転送する必要がない
    - 転送は普通のC/S構成と同じくHTTP、ただしover SSH tcp-forward.
    - run-listにあるCookbookのみキャッシュはさせます。(files,templatesは使う時点で転送)
- サーバの状態(Node Object)を手元のファイルに書き出して保存
    - 不要なAttributeは保存対象外にできる  
    - RecipeやKnifeからサーチも使える
- ChefServer構成で使える機能を認証を除いてサポート


> ### ChefZeroとChefのローカルモードについて
> ChefZeroはオンメモリ軽量ChefServer(※認証はダミー)。
> 標準のオンメモリだけでなくバックエンドが色々選択でき、今回関係するChef-Clientのローカルモードはファイルシステムをそのままバックエンドとして使用する。
> 
> ローカルモードはChefZero(※実行中のみ起動)をChefServerに見立ててknifeによるCRUD、Node情報の更新やサーチが行えるモード。

ちなみに[Chef-Metal](https://github.com/opscode/chef-metal)も似たような思想で作られていますが、knife-zeroは従来のワークフローを維持する簡単な拡張で済ませています。

## インストールと実行

GemなのでBundleで。既存のChef-Repoか、Chef-Repoにしていく予定のディレクトリで。

```Gemfile
source "https://rubygems.org"

gem 'chef'
gem 'knife-zero'
```

```
$ bundle install --path vendor/bundle --binstubs

...

Installing chef (11.14.6)
Installing knife-zero (0.0.1)
Your bundle is complete!
It was installed into ./vendor/bundle
```

実行はChefの入っていないサーバに対して、`knife zero bootstrap FQDN(or IPaddress)`。

(※Chef導入済みならインストールだけスキップされます)

```
$ ./bin/knife zero bootstrap zero.example.com
WARN: No cookbooks directory found at or above current directory.  Assuming /Users/sawanoboriyu/Dev/worktmp/knife-zero01.  ## Soloと違い、Cookbooksディレクトリは無くてもよい

INFO: Starting chef-zero on host localhost, port 8889 with repository at repository at /Users/sawanoboriyu/Dev/worktmp/knife-zero01 ## ChefZeroが一時的に起動
  One version per cookbook

Connecting to zero.example.com  ## リモートへ

zero.example.com Installing Chef Client...  ## Chefが入ってなかったのでインストール

...

zero.example.com Thank you for installing Chef!


zero.example.com Starting first Chef Client run...  ## 次にChefがClient/Serverモードで実行される
zero.example.com Starting Chef Client, version 11.14.2
zero.example.com resolving cookbooks for run list: []
zero.example.com Synchronizing Cookbooks:
zero.example.com Compiling Cookbooks...
zero.example.com [2014-08-21T09:24:31+00:00] WARN: Node zero.example.com has an empty run list.
zero.example.com Converging 0 resources
zero.example.com 
zero.example.com Running handlers:
zero.example.com Running handlers complete
zero.example.com Chef Client finished, 0/0 resources updated in 4.538956678 seconds
```

ローカルのChef-Repoがカラッポで、ランリストを特に指定していないのでChef-Clientが実行されただけです。

ただNodeObjectはローカルのWorkStationに保存されました。
ファイルとしても見れるし、Server環境と同様のKnifeコマンド(LocalMode)で内容表示もOK.

```
$ ls nodes/
zero.example.com.json

$ knife node show zero.example.com --local-mode
WARN: No cookbooks directory found at or above current directory.  Assuming /Users/sawanoboriyu/Dev/worktmp/knife-zero01.
Node Name:   zero.example.com
Environment: _default
FQDN:        zero.example.com
IP:          210.xxx.xxx.xxx
Run List:    
Roles:       
Recipes:     
Platform:    ubuntu 14.04
Tags:        
```

以降毎回`--local-mode`なので、コンフィグに書いてしまうとちょっと楽です。

```.chef/knife.rb
local_mode true
```

CookbookなどもSoloのように分岐せずにちゃんと使えます。


## Knife-Zero bootstrapとLocalModeのChefZero挙動

このへんで裏がどんなふうになっているのか図解。
流れを追っていくと、次のように動作しています。

![knfe-zero.004.png](https://qiita-image-store.s3.amazonaws.com/0/7454/3f265e4f-f30f-3d3c-224c-4601b8b40de2.png "knfe-zero.004.png")

`knife zero bootstrap`を実行します。

![knfe-zero.005.png](https://qiita-image-store.s3.amazonaws.com/0/7454/979fc157-ef53-4dc8-5841-0c05c10f1255.png "knfe-zero.005.png")

ローカル側にChefZeroサーバ。

![knfe-zero.006.png](https://qiita-image-store.s3.amazonaws.com/0/7454/dd049b55-8e25-b7db-9d67-406f5f05c3ac.png "knfe-zero.006.png")

Bootstrapなので、リモートサーバにSSHセッションを確立します。s

![knfe-zero.007.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d7acda4c-8c85-1f06-3062-d3b1f5876b33.png "knfe-zero.007.png")

このSSHセッションは`-R 8889:127.0.0.1:8889`オプション相当ということですね。

![knfe-zero.008.png](https://qiita-image-store.s3.amazonaws.com/0/7454/84777892-18be-1de5-39b6-10dea8cbc224.png "knfe-zero.008.png")

以降は通常のbootstrap処理です。

![knfe-zero.009.png](https://qiita-image-store.s3.amazonaws.com/0/7454/50099ea2-a02e-ea45-e8ba-d13fb949963e.png "knfe-zero.009.png")

ただし、chef-clientの実行オプションで、ChefServerを`http://127.0.0.1:8889`と指定(`-S http://127.0.0.1:8889`)して実行。

![knfe-zero.010.png](https://qiita-image-store.s3.amazonaws.com/0/7454/7a4507c4-9255-21cc-41f9-f9c881264f45.png "knfe-zero.010.png")

Chef-Clientが終了処理としてChef-Server(Zero)にNode情報をPOST。

![knfe-zero.011.png](https://qiita-image-store.s3.amazonaws.com/0/7454/009ce5af-850f-d090-1fb9-7492528edaf6.png "knfe-zero.011.png")

POSTされたデータはChefZeroがバックエンドのファイルシステムに保存します。

![knfe-zero.012.png](https://qiita-image-store.s3.amazonaws.com/0/7454/b7de6df7-193f-d417-ade3-95c1957a9d67.png "knfe-zero.012.png")

Knifeコマンド終了時には、SSHセッションを閉じ、ChefZeroもいなくなります。

![knfe-zero.013.png](https://qiita-image-store.s3.amazonaws.com/0/7454/618f7071-a06f-ae40-9f22-16b7524f8c34.png "knfe-zero.013.png")

単純な仕組みですね。

## 適当なレシピを適用してみる

Cookbook管理ライブラリを利用して適当なCookbookを調達しましょう。

今回はlibrarian-chefで、`fluentd_bundle`(`/opt/fluentd`にfluentdをインストール)のCookbookでも。

```ruby:Cheffile
#!/usr/bin/env ruby
#^syntax detection

site 'https://supermarket.getchef.com/api/v1'

cookbook 'fluentd_bundle'
```

`librarian-chef install`で調達。

```
$ ./bin/librarian-chef install
Installing apt (2.5.2)
Installing build-essential (2.0.6)
Installing chef_handler (1.1.6)
Installing dmg (2.2.0)
Installing yum (3.2.4)
Installing yum-epel (0.4.0)
Installing runit (1.5.10)
Installing windows (1.34.2)
Installing git (4.0.2)
Installing ohai (2.0.1)
Installing rbenv (1.7.1)
Installing fluentd_bundle (0.2.0)


$ ./bin/knife recipe list fluentd  --local-mode
fluentd_bundle
fluentd_bundle::default
fluentd_bundle::depends_smartos
fluentd_bundle::depends_ubuntu
fluentd_bundle::rbenv
```

依存を含めてCookbookが揃ったら、適用してみます。

現在のバージョンだと、もう一度`knife zero bootstrap`コマンドです。更新はもう少し丁度いい名前を付けてサブコマンドを独立させたいとこですが。

ランリストに`fluentd_bundle::default`を付けて実行。

```
$ ./bin/knife zero bootstrap zero.example.com -r fluentd_bundle::default
Connecting to zero.example.com
zero.example.com Starting first Chef Client run...

...

zero.example.com Starting Chef Client, version 11.14.2
zero.example.com resolving cookbooks for run list: ["fluentd_bundle::default"]  ## HTTP over SSH経由で取得
zero.example.com Synchronizing Cookbooks:
zero.example.com   - fluentd_bundle
zero.example.com   - rbenv
zero.example.com   - build-essential
zero.example.com   - git
zero.example.com   - apt
zero.example.com   - ohai
zero.example.com   - dmg
zero.example.com   - windows
zero.example.com   - runit
zero.example.com   - yum
zero.example.com   - yum-epel
zero.example.com   - chef_handler
zero.example.com Compiling Cookbooks...


zero.example.com Running handlers:
zero.example.com Running handlers complete
zero.example.com Chef Client finished, 46/59 resources updated in 934.061999439 seconds
```

ちょっと時間のかかるサンプルを選んでしまった気がしますが。。 きっちり動く事の確認になりました。

もう一度`knife node show`で、ローカルのNode Objectに状態が反映されていることが確認できます。

```
$ knife node show zero.example.com --local-mode
Node Name:   zero.example.com
Environment: _default
FQDN:        zero.example.com
IP:          210.xxx.xxx.xxx
Run List:    recipe[fluentd_bundle::default]  ## さっき実行したランリストが反映されている
Roles:       
Recipes:     fluentd_bundle::default, fluentd_bundle::rbenv, rbenv::default, apt::default, build-essential::default, build-essential::_debian, git::default, rbenv::ruby_build, fluentd_bundle::depends_ubuntu
Platform:    ubuntu 14.04
Tags:        
```




## 他のKnifeコマンドもローカルモードで違和感なし

ちなみにKnife+LocalModeならChef-Server必須だったコマンドも違和感なく使えます。

### `knife ssh`をする例

nodeは１つ追加してます。

```shell:
$ knife ssh 'hostname:*' --local-mode uptime --attribute ipaddress 
210.xxx.xxx.xxx  08:44:09 up  1:06,  1 user,  load average: 0.00, 0.01, 0.01
210.xxx.xxx.xxx  08:44:10 up 143 days,  2:34,  4 users,  load average: 0.02, 0.02, 0.05
```

普通にできるね。


### `node edit`などでランリストを編集後、適用したい時。

> ※追記も参照のこと

この運用はプラグインにそのうち実装しておきたいけど、`knife exec`でちょっと工夫すればKnife-Zero不要で最初から可能だったりする。

```shell:
$ knife exec --local-mode -E 'nodes.all {
  |n| system "ssh -R8889:127.0.0.1:8889 #{n.ipaddress} chef-client -S http://127.0.0.1:8889" 
}'
```

`nodes`で拾ったノード全部に対して、tcpフォワード付きのSSHを繋ぎつつChef-Clientを実行すればOK。

ちなみにどうやらChefIncの事例でよくあげられるFacebookでも[同じような運用を目指しているくさい](https://github.com/opscode/chef-zero/issues/86)。

----

#### 追記：v0.1.0で`knife zero chef_client`実装しまして、1.4くらいでconvergeと別名をつけました。

`knife zero converge QUERY (options)`

サーチに引っかかったノードに対して、ChefZero+SSH tcp-forwardつきでChef-Clientを実行してきます。

GithubのREADMEから使い方を転載。

```
## add recipe to run_list of host.example.com
$ knife node run_list add host.example.com hogehoge::default --local-mode
host.example.com:
  run_list: recipe[hogehoge::default]


$ knife zero converge 'name:*' --attribute ipaddress 

## host.example.com was converged by run_list.
host.example.com Starting Chef Client, version 11.14.6
host.example.com resolving cookbooks for run list: ["hogehoge::default"]
host.example.com Synchronizing Cookbooks:
host.example.com   - hogehoge
host.example.com Compiling Cookbooks...
host.example.com Converging 0 resources

(以下略)
```

サーチ結果が複数あった場合は直列実行です。


## 現状をふまえたその他

- 結局リモートサーバにChefが入る
    - パッケージだし多めに見てもらえると。
    - 徹底リモートでやるよりは、構成情報の収集が早く済むという利点も
- 実行中はリモートサーバでlocalhost:8889がListen状態になるのってどうなん
    - 認証がダミーなので、誰でもリソース取り放題といえばそう。実行が一瞬で終わらない場合に共有サーバだとちょっと嫌かも。
    - ポート変えるとか、ちょっと認証つけとくとか([プルリク中](https://github.com/opscode/chef-zero/pull/88))まあなんとかなるしょ
- Cookbookのキャッシュはリモートに残る
    - Chef C/S環境の差し替えなんで。
    - ディレクトリ丸ごとアップよりは誤動作も無いはず



## まとめ

他のインフラ構成用ツールや、Knifeのクラウド調達系プラグインなどとあわせればインフラの現状もほぼコードで管理できていい感じかも。
