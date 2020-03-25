
前回 [Chefのローカルモードだけでリモートサーバを運用してみようと、Knife-Zeroを作った。Nodeの構成情報もとれるよ。 - Qiita](http://qiita.com/sawanoboly/items/218a7b03ddec6be45e34 "Chefのローカルモードだけでリモートサーバを運用してみようと、Knife-Zeroを作った。Nodeの構成情報もとれるよ。 - Qiita") の続きといえば続きです。

> Knife-Zeroのページはこちら。 http://knife-zero.github.io/ja/

Chef11.xからローカルモードというのが加わりました。Chef-Client/Server環境の簡易版であり、Soloの代わりでもあります。

> ### Chef-Soloからの乗り換えとしてChef-Zero(ローカルモード)検索が多いようなので、この追記を先頭に移動
> このサンプルではSSH越しにローカルモードを実行していますが、単にサーバ側にChef-Repoを置いてローカルモードをしたい場合、
> Chefをインストール後にChef-Repoのディレクトリに移動して`chef-client -z`でOKです。
> SoloみたいにCookbookPathが無いとかそういうエラーもなく、簡単に実行できます。
> ついでに、[Knife-Solo](https://matschaffer.github.io/knife-solo/)は無くならないので、乗り換え必須ではありません。
>
> 追記1： で、Soloどうなるのよってのは[こちらに書いた](http://www.sawanoboly.net/2014/12/chef-solo-zero-knife-solo-zero/)ので合わせてどうぞ。
> 追記2: 200ストックの通知が来たので、コマンド等を現行(1.8.0時)向けに少し修正
> 追記3: Knife-Zeroのページ。

## 概要

今回はローカルモードを主眼において、そのままChef-Serverレスでインフラを管理していこうという試みのチュートリアルです。
本文はだいたいこんな感じですすめます。

- ローカルモードでのChefのインフラ構築ワークフロー
- Knife-Sakuraでサーバ調達 (※ optional)
- Knife-Zero

ちなみに作業後のディレクトリ構成等は [higanworks/chef_localmode_tutorial](https://github.com/higanworks/chef_localmode_tutorial "higanworks/chef_localmode_tutorial") にアップしているので、セットアップ代わりにこちらをクローンしてから始めてもよいです。

## セットアップ

Rubyプロジェクトぽく進めます、必要なディレクトリを用意して。

```
$ mkdir my_chefrepo
$ cd my_chefrepo/
$ bundle init
Writing new Gemfile to /Users/sawanoboriyu/my_chefrepo/Gemfile
```

Gemfileに使うGemsを記述して。

```runy:Gemfile 
source "https://rubygems.org"

gem 'chef'
gem 'knife-zero'
gem 'knife-sakura'
```

```

Bundleします。

...
Installing chef (11.14.6)
Installing knife-zero (0.1.2)
...

Installing knife-sakura (0.0.2)
Your bundle is complete!
It was installed into ./vendor/bundle
```

ちなみにknife-zero(0.1.2), knife-sakura(0.0.2)以上が必須です。

`--binstubs`しているので、カレントの`./bin`に実行ファイルが来ます。

```
$ tree . -I vendor
.
├── Gemfile
├── Gemfile.lock
└── bin
    ├── chef-apply
    ├── chef-client
    ├── chef-service-manager
    ├── chef-shell
    ├── chef-solo
    ├── chef-zero
    ├── coderay
    ├── erubis
    ├── ffi-yajl-bench
    ├── fog
    ├── htmldiff
    ├── knife
    ├── ldiff
    ├── nokogiri
    ├── ohai
    ├── pry
    ├── rackup
    ├── restclient
    └── shef
```

### ローカルモードの準備

knifeコマンドのデフォルト動作をローカルモードにするため`knife.rb`を作ります。この記述はコマンドラインの`--local-mode`オプションと同じ働きです。

```
$ mkdir .chef
$ echo 'local_mode true' > .chef/knife.rb
```


これでローカルモードで動作するようになります。
Chef-Client/Server環境のコマンド、`knife environment list`等を叩いてみましょう。

```
$ ./bin/knife environment list -V
WARN: No cookbooks directory found at or above current directory.  Assuming /Users/sawanoboriyu/Dev/worktmp/my_chefrepo.
INFO: Starting chef-zero on host localhost, port 8889 with repository at repository at /Users/sawanoboriyu/Dev/worktmp/my_chefrepo
  One version per cookbook


$ ./bin/knife node list -V
WARN: No cookbooks directory found at or above current directory.  Assuming /Users/sawanoboriyu/Dev/worktmp/my_chefrepo.
INFO: Starting chef-zero on host localhost, port 8889 with repository at repository at /Users/sawanoboriyu/Dev/worktmp/my_chefrepo
  One version per cookbook
```

この記述 `Starting chef-zero on host localhost, port 8889 with repository at repository at /Users/sawanoboriyu/Dev/worktmp/my_chefrepo` がポイントです。
ローカルのディレクトリをベースにChefZeroが起動して、ChefServerの代わりをしています。


なおWARNが出ていますが、単に`./cookbooks`ディレクトリが無いためです。
あとで勝手にできるので、これ以降`WARN: No cookbooks directory〜`は省略します。

## Environmentをつくる

新しいEnvironment、`development`を作ってみましょう。`knife environment create development`です。

```json:
$ ./bin/knife environment create development

## ---エディタが開く

{
  "name": "development",
  "description": "",
  "cookbook_versions": {
  },
  "json_class": "Chef::Environment",
  "chef_type": "environment",
  "default_attributes": {
  },
  "override_attributes": {
  }
}
## --- 保存して終了

Created development
```

リストに出るようになりました。

```
$ ./bin/knife environment list
development
```

ファイルとして保存されています。

```
$ tree environments/
environments/
└── development.json
```

`knife search`で`environment`を対象に探せばちゃんと見つかります。

```
$ ./bin/knife search environment 'name:dev*'
1 items found

chef_type:           environment
cookbook_versions:
default_attributes:
description:         
json_class:          Chef::Environment
name:                development
override_attributes:
```


> ### 余談：ChefZeroを単体で上げる`knife serve`
> 
> `knife serve`サブコマンドを使うと、ローカルのディレクトリをChef-Repoに見立てたChefZeroが起動します。
> ※ 通常のchef-zeroコマンドではすべてオンメモリで、リソース空っぽで起動して終了時に何も保存しません。
> 
> ```
> $ ./bin/knife serve
> Serving files from:
> repository at /Users/sawanoboriyu/Dev/worktmp/my_chefrepo
>   One version per cookbook
> 
> >> Starting Chef Zero (v2.2)...
> >> WEBrick (v1.3.1) on Rack (v1.5) is listening at http://localhost:8889
> >> Press CTRL+C to stop
> ```
> 
> この状態なら、Chef-Serveに問い合わせる要領でオブジェクトを取得することもできるので、色々動作確認してみたい場合は使える機能です。
> 
> ```
> $ curl -H 'Accept: application/json' localhost:8889/environments
> {
>   "development": "http://localhost:8889/environments/development"
> }
> ```
> 
> `Accept: application/json`ヘッダを忘れずに。
> 
> ```
> $ curl -H 'Accept: application/json' localhost:8889/environments/development
> {
>   "name": "development",
>   "description": "",
>   "cookbook_versions": {
>   },
>   "json_class": "Chef::Environment",
>   "chef_type": "environment",
>   "default_attributes": {
>   },
>   "override_attributes": {
>   }
> }
> ```
> 
> ついでに言うと、これらのオブジェクトはファイルベースなので`rm`でもサクッと消せます。

## Data Bagをつくる

同様にData Bagを作ってみます。

```json
$ ./bin/knife data bag create samplebag sampleitem

## ---エディタが開くので編集する
{
  "id": "sampleitem",
  "foo": "bar"
}
## --- 保存して終了

Created data_bag[samplebag]
Created data_bag_item[sampleitem]
```

リストOK。

```
$ ./bin/knife data bag list
samplebag

$ ./bin/knife data bag show samplebag sampleitem
foo: bar
id:  sampleitem
```

これもファイルとして保存されています。

```
$ tree data_bags/
data_bags/
└── samplebag
    └── sampleitem.json

1 directory, 1 file
```

> 追記:
> この段落をサーバ側で実行すれば、単体でChef-Soloを実行するのとだいたいおなじ感覚になります。


## 適当なNodeを用意する knife-sakura

ではChefの管理対象とするリソースとして、今回はLinuxサーバを用意します。

サーバはSSHが繋がるならば既存のものでも良いですし、aws cliでもterraformでも、ローカルのVagrantでも好きなように調達してください。

さて、今回はせっかくなのでなんかしらクラウドで。
Knifeのプラグインを利用して[さくらのクラウド](http://cloud.sakura.ad.jp/)から調達します。みなさんたいていクーポンが遊んでいるでしょう。

- 参考＆詳しい使い方は: [cl-lab-k/knife-sakura](https://github.com/cl-lab-k/knife-sakura "cl-lab-k/knife-sakura")

Vagrant+VirtualBoxでマシンを調達するならば [Vagrantでknife-zeroを使うためのベストプラクティス | 割り箸ポテチ](http://blog.chopschips.net/blog/2015/08/25/vagrant-knife-zero-best-practice/ "Vagrantでknife-zeroを使うためのベストプラクティス | 割り箸ポテチ") も参考になるでしょう。

### さくらのクラウド(knife-sakura)セットアップ
`.chef/knife.rb`をちょろちょろっと編集して、環境変数をエクスポートしておきます。

```ruby:.chef/knife.rb 
local_mode true

## SakuraCloud credentials
knife[:sakuracloud_api_token] = ENV['SAKURACLOUD_API_TOKEN']
knife[:sakuracloud_api_token_secret] = ENV['SAKURACLOUD_API_TOKEN_SECRET']
knife[:sakuracloud_ssh_key] = ENV['SAKURACLOUD_USER_KEY_ID']
```

### さくらのクラウド(knife-sakura)でサーバ調達

サーバを調達するには、`knife sakura server create`です。

server1,server2,server3として、3台ほど調達を試みます。
ついでに`--no-bootstrap`として、Chef-Server連携用のbootstrapをキャンセルします。 (※ この後のknife-zeroのため)

```
$ for x in {1..3} ; do echo ./bin/knife sakura server create  --server-plan 1001 --disk-plan 4 --source-archive 112500463685 --no-bootstrap -n server${x}; done

Instance ID: 112600642xxx
Server Plan: プラン/1Core-1GB
Public IP Address: 133.242.231.xxx

Instance ID: 112600642xxx
Server Plan: プラン/1Core-1GB
Public IP Address: 133.242.230.xxx

Instance ID: 112600642xxx
Server Plan: プラン/1Core-1GB
Public IP Address: 133.242.237.xxx
```

新鮮なサーバが3台とれました。

```
$ ./bin/knife sakura server list
ID                                    Name                                  Status                                Created at                          
112600642xxx                          1a966089-d6bb-42cb-a7df-08beda84cxxx  up                                    2014-08-25Txx:xx:xx+09:00           
112600642xxx                          aa897042-ecf0-49fd-9eaa-05dea1be3xxx  up                                    2014-08-25Txx:xx:xx+09:00           
112600642xxx                          9bd5969f-a682-465d-b3c0-6e86148a4xxx  up                                    2014-08-25Txx:xx:xx+09:00
```

じゃあこれらをChefローカルモードの管理下においていきます。

### knife-zeroでnodeを管理下に

`knife zero bootstrap`をすることで、[前回](http://qiita.com/sawanoboly/items/218a7b03ddec6be45e34 "Chefのローカルモードだけでリモートサーバを運用してみようと、Knife-Zeroを作った。Nodeの構成情報もとれるよ。 - Qiita")のようにします。

`-E development`をつけて、`Environent => development`のノードとして登録しましょう。

```
## server1をbootstrap
$ ./bin/knife zero bootstrap 133.242.231.xxx -x ubuntu -E development --sudo -N server1
...
133.242.231.xxx Chef Client finished, 0/0 resources updated in 3.692237549 seconds


## server2をbootstrap
$ ./bin/knife zero bootstrap 133.242.230.xxx -x ubuntu -E development --sudo -N server2
...
133.242.230.xxx Chef Client finished, 0/0 resources updated in 3.692237549 seconds


## server3をbootstrap
$ ./bin/knife zero bootstrap 133.242.237.xxx -x ubuntu -E development --sudo -N server3
...
133.242.237.xxx Chef Client finished, 0/0 resources updated in 3.692237549 seconds


```

無事Zero Bootstrapが終わったので、nodeのリストが取れることを確認しましょう。

```
$ ./bin/knife node list
server1
server2
server3
```

もちろんファイルとしてローカルにありつつ。

```
$ tree nodes/
nodes/
├── server1.json
├── server2.json
└── server3.json

0 directories, 3 files
```

`knife node show`などで情報を表示することができます。

```
$ ./bin/knife node show server1
Node Name:   server1
Environment: development
FQDN:        localhost
IP:          133.242.231.xxx
Run List:    
Roles:       
Recipes:     
Platform:    ubuntu 12.04
Tags:        

```

`Environent => development`なNodeをサーチすれば、それぞれがちゃんと引っかかりますね。

```
$ ./bin/knife search node 'chef_environment:development'
3 items found

Node Name:   server1
Environment: development
FQDN:        localhost
IP:          133.242.231.xxx
Run List:    
Roles:       
Recipes:     
Platform:    ubuntu 12.04
Tags:        

Node Name:   server2
Environment: development
FQDN:        localhost
IP:          133.242.230.xxx
Run List:    
Roles:       
Recipes:     
Platform:    ubuntu 12.04
Tags:        

Node Name:   server3
Environment: development
FQDN:        localhost
IP:          133.242.237.xxx
Run List:    
Roles:       
Recipes:     
Platform:    ubuntu 12.04
Tags:        

```

標準の`knife ssh`もローカルモードで動作させれば、このとおり。

```
$ ./bin/knife ssh 'chef_environment:development' -x ubuntu --attribute ipaddress uptime 
133.242.231.xxx  xx:40:51 up  1:41,  1 user,  load average: 0.00, 0.01, 0.05
133.242.230.xxx  xx:40:51 up  1:40,  1 user,  load average: 0.00, 0.02, 0.05
133.242.237.xxx  xx:40:51 up  1:38,  1 user,  load average: 0.00, 0.02, 0.05
```

※ 標準で接続先に使われるFQDNが全部`localhost`なので、`--attribute ipaddress`を指定してSSHの接続対象をIPにしています。

> 追記： Knife-Zero 1.8.0で、Bootstrapで指定したIPを`node.knife_zero.host`で保持するようにしました。
> IPアドレスを複数持つ一部の環境では`node.ipaddress`が手元からつながらない場合があるため、Bootstrapでの接続実績を使い回しできるようにする更新です。
> そっちを使う場合は、`--attribute ipaddress`の代わりに、`--attribute knife_zero.host`でどうぞ。

## Cookbookを作ってレシピを適用、Data Bag, SearchもOK

`knife cookbook create`でCookbookを作りましょう。

```
$ ./bin/knife cookbook create demobook
** Creating cookbook demobook
** Creating README for cookbook: demobook
** Creating CHANGELOG for cookbook: demobook
** Creating metadata for cookbook: demobook
```

`local_mode`なので`./cookbooks/`ディレクトリを自動でつくって、雛形をおいてくれます。

```
$ tree cookbooks/
cookbooks/
└── demobook
    ├── CHANGELOG.md
    ├── README.md
    ├── attributes
    ├── definitions
    ├── files
    │   └── default
    ├── libraries
    ├── metadata.rb
    ├── providers
    ├── recipes
    │   └── default.rb
    ├── resources
    └── templates
        └── default

11 directories, 4 files
```

### Data Bagを使うレシピ

C/S環境と同様にかつ、ノード側に特にプラグイン無しでData Bagが利用できることを確認するため、次のようなRecipeを作りました。

```ruby:cookbooks/demobook/recipes/data_bag.rb 
text = data_bag_item('samplebag', 'sampleitem')

file '/tmp/from_databag.txt' do
  content text['foo']
end
```


### Searchを使うレシピ

同様にSearchが利用できることを確認するため、次のようなRecipeを作りました。

```ruby:cookbooks/demobook/recipes/search.rb
nodes = search(:node, "chef_environment:#{node.chef_environment}")
list = nodes.map {|n| n.to_s}

file '/tmp/nodes_from_search.txt' do
  content list.join("\n")
end
```

### ランリストをまとめるRoleを作る

Environemntなどと同様に、`knife role create`でRoleを作ります。

```json:
$ ./bin/knife role create demorole

## ---エディタが開く
{
  "name": "demorole",
  "description": "", 
  "json_class": "Chef::Role",
  "default_attributes": {
  },  
  "override_attributes": {
  },  
  "chef_type": "role",
  "run_list": [
    "demobook::data_bag",    ## ここを追記
    "demobook::search"       ## ここを追記
  ],  
  "env_run_lists": {
  }
}
## --- 保存して終了

Created role[demorole]

```

`role show`で確認したりも可と。

```
$ ./bin/knife role show demorole
chef_type:           role
default_attributes:
description:         
env_run_lists:
json_class:          Chef::Role
name:                demorole
override_attributes:
run_list:
  recipe[demobook::data_bag]
  recipe[demobook::search]
```


`server1`に`role[demorole]`の役割を与えてみましょう。
`knife node run_list add`です。

```
$ knife node run_list add server1 'role[demorole]'
server1:
  run_list: role[demorole]
```

適用するにはまたKnife-Zeroで作った拡張の`knife zero converge`でOKです。
まあポートフォワードを付けてChef-Clientを叩いてくるだけなのですが。

> 追記: `zero chef_client`は`zero converge`に(主に気分で)変えました。`zero chef_client`も使えます。

```shell:
$ ./bin/knife zero converge 'name:server1' -x ubuntu --sudo --attribute ipaddress
133.242.231.xxx Starting Chef Client, version 11.14.6

## RunListからCookbookを取得する
133.242.231.xxx resolving cookbooks for run list: ["demobook::data_bag", "demobook::search"]
133.242.231.xxx Synchronizing Cookbooks:
133.242.231.xxx   - demobook
133.242.231.xxx Compiling Cookbooks...
133.242.231.xxx Converging 2 resources


## demobook::data_bagの実行
133.242.231.xxx Recipe: demobook::data_bag
133.242.231.xxx   * file[/tmp/from_databag.txt] action create
133.242.231.xxx     - create new file /tmp/from_databag.txt
133.242.231.xxx     - update content in file /tmp/from_databag.txt from none to fcde2b
133.242.231.xxx     --- /tmp/from_databag.txt	2014-08-25 xx:15:49.135991498 +0900
133.242.231.xxx     +++ /tmp/.from_databag.txt20140825-3969-1tnpdkw	2014-08-25 xx:15:49.135991498 +0900
133.242.231.xxx     @@ -1 +1,2 @@
133.242.231.xxx     +bar  ## Data Bagの中身をファイルに出力


## demobook::searchの実行
133.242.231.xxx Recipe: demobook::search
133.242.231.xxx   * file[/tmp/nodes_from_search.txt] action create
133.242.231.xxx     - create new file /tmp/nodes_from_search.txt
133.242.231.xxx     - update content in file /tmp/nodes_from_search.txt from none to 37ab54
133.242.231.xxx     --- /tmp/nodes_from_search.txt	2014-08-25 xx:15:49.139991498 +0900
133.242.231.xxx     +++ /tmp/.nodes_from_search.txt20140825-3969-ln7sth	2014-08-25 xx:15:49.139991498 +0900
133.242.231.xxx     @@ -1 +1,4 @@   ## Searchの結果をファイルに出力
133.242.231.xxx     +node[server1]
133.242.231.xxx     +node[server2]
133.242.231.xxx     +node[server3]
133.242.231.xxx 
133.242.231.xxx Running handlers:
133.242.231.xxx Running handlers complete
133.242.231.xxx Chef Client finished, 2/2 resources updated in 4.011358121 seconds
```

Data Bag,Searchの機能がちゃんと使えていることがわかりますね。

ちなみにローカルモードで`knife status`も有効です。
最後に`knife zero converge`を実行したのが何時なのかわかって助かりますね。

```
$ ./bin/knife status
47 minutes ago, server2, localhost, 133.242.230.xxx, ubuntu 12.04.
45 minutes ago, server3, localhost, 133.242.237.xxx, ubuntu 12.04.
6 minutes ago, server1, localhost, 133.242.231.xxx, ubuntu 12.04.
```

最後に`server1`の現状を`node show`で確認してチュートリアルをおわりましょう。

```
$ ./bin/knife node show server1
Node Name:   server1
Environment: development
FQDN:        localhost
IP:          133.242.231.xxx
Run List:    role[demorole]
Roles:       demorole
Recipes:     demobook::data_bag, demobook::search
Platform:    ubuntu 12.04
Tags:        
```



## ワークフローの補足

ここで紹介しているワークフローは先日出版された『Chef活用ガイド』の内容を踏襲しています。
Chef-Solo系では難しかったワークフローだったのですが、ローカルモードを上手く使えばChef-Serverが無くてもむしろ書籍の内容をほぼ100%実践できます。

書中でもChef-Serverについて配布用のハブであり、情報の集約をするとしていますが、ローカルモード+Knife-Zeroがその役割を全部担える以上、Chef-Server本体は特に必要ありませんね。

[![Chef活用ガイド](http://ascii.asciimw.jp/books/books/jpeg150/978-4-04-891985-2.jpg)](http://ascii.asciimw.jp/books/books/detail/978-4-04-891985-2.shtml)



## おわりに

この例ではChefなので、ローカルのディレクトリをChef-Repoとしてだけ扱いました。ローカルディレクトリで完結することにはそれなりに意味があります、他のツールと組み合わせやすいんですよね。

なにかしらCLIでs3のバケットを確保してもよし、route53やdnsimpleのCLIでDNSをいじっても良いし、kumogataでCloudFormationを作ってみたりVagrantやTerraformでDigital Oceanのホストを作ったりherokuにアプリをおいてもOK。
で、Recipeが使いたい、構成情報をとっておきたいといったノードに対してのみChefをつかったり、Chef-MetalやChef-Container(Docker)と合わせるのも良いでしょう。

Test-KitchenやServerspecのテストコードをいれて、テストが通ったらほぼシームレスにNode(Searchで抽出)に適用するようなフローも可能です。

リソースが確保できるか/それがどこにあるか、どのようにリソースを使うか/使わてれいるか、またそれらはクエリ可能か、とこういった概ね現状がわかる情報をふくめたインフラの構成内容をツールで管理しつつGit等で追跡できるようになって、Infrastructure as Codeってやつが捗るんじゃないかと思います。

