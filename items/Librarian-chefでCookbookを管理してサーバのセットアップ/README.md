<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[Librarian-chef](https://github.com/applicationsonline/librarian) を使い、bundlerのように必要なcookbookを管理する。



Chefのセットアップ
----
毎度つかうものなので、`git clone`一発でchef-client系をセットアップ出来るようにした。

Ubuntu: https://github.com/higanworks/chef-with-ruby_precise-x86_64
CentOS: https://github.com/higanworks/chef-with-ruby_CentOS-x86_64

セットアップはREADMEにあるようにコマンド2つ。

```
$ apt-get install git
$ git clone https://github.com/higanworks/chef-with-ruby_`lsb_release -cs`-`uname -i`.git /opt/ruby-chef
```

以降はとりあえずUbuntuで上記を使った場合の`chef-solo`発動までを紹介。


Cheffile作業ディレクトリへ
----
リポジトリをクローンしたら /opt/ruby-chef で作業する。

`cd /opt/ruby-chef`

サンプルではcookbooksもこの下に置かれるようにしている。

`librarian-chef`は必要なCookbookの名前と所在、依存を**Cheffile**に書き、`install`サブコマンドでCookbooksディレクトリを形成する。
集めた結果を`Cheffile.lock`に書く、lockを元に作る`deployment`モードこそ無さそうだがとてもbundlerのようだ。


Cheffile.sample をそのまま使う場合はCheffileとしてコピー。
`cp Cheffile.sample Cheffile`

空のCheffileを作る場合は
`./bin/librarian-chef init`


Cheffileからcookbooksの準備
----

chefファイルにcookbookの名前と所在を書く、とりあえずsoloなのでこのサーバに必要なものだけ記述。

`nginx`と`redis`と`monit`を導入したい、ついでにruby(rvm)。

```ruby:Cheffile(sample)
#!/usr/bin/env ruby
#^syntax detection

site 'http://community.opscode.com/api/v1'

cookbook 'apt'

# external
cookbook 'rvm',
  :git => 'https://github.com/fnichol/chef-rvm'

# my cookbooks
cookbook 'nginx_ppa',
  :git => 'https://github.com/higanworks-cookbooks/nginx_ppa.git'

cookbook 'monit_binaries',
  :git => 'https://github.com/higanworks-cookbooks/monit_binaries.git'

cookbook 'redis_src',
  :git => 'https://github.com/higanworks-cookbooks/redis_src.git'
```

### Cookbookインストール
`./bin/librarian-chef install` と叩き、`./cookbooks`の下にCookbookを集めてくる。

```実行結果
# ./bin/librarian-chef install
# ./bin/librarian-chef show
apt (1.4.8)
monit_binaries (0.0.3)
nginx_ppa (0.0.1)
redis_src (0.0.1)
rvm (0.9.1)
```

もちろんこれを`knife cookbook upload`するような運用もあり。


レシピの適用
----

### 単一のレシピ適用
solo用のサンプルコンフィグがついているので叩く。
適用したいレシピは`-o`で指定できる。

```実行結果
# ./bin/chef-solo -c solo.rb -o nginx_ppa::default
-- snip --
[2012-09-19T20:29:28-07:00] INFO: Processing package[nginx] action install (nginx_ppa::default line 34)
-- snip --

# nginx -v
nginx version: nginx/1.2.3
```

### Jsonの構成ファイルを用いた複数のレシピ適用
`-o`の直接指定ではなく、`chef-solo`に`-j`でjsonによる構成ファイルを渡す。
色々入れて、rvmでruby1.9.3を使えるようにしておいてねというコース。



```json:sample.json
{
   "rvm" : {
      "default_ruby" : "system",
      "rubies" : [
         "ruby-1.9.3-p194"
      ]
   },
   "run_list" : [
      "recipe[apt]",
      "recipe[nginx_ppa]",
      "recipe[monit_binaries]",
      "recipe[redis_src]",
      "recipe[rvm::system]"
   ]
}
```

Roleも書ける。

```実行結果
# ./bin/chef-solo -c solo.rb -j ./sample.json
[2012-09-19T20:28:35-07:00] INFO: *** Chef 10.14.2 ***
[2012-09-19T20:28:35-07:00] INFO: Setting the run_list to ["recipe[apt]", "recipe[nginx_ppa]", "recipe[monit_binaries]", "recipe[redis_src]", "recipe[rvm::system]"] from JSON
[2012-09-19T20:28:35-07:00] INFO: Run List is [recipe[apt], recipe[nginx_ppa], recipe[monit_binaries], recipe[redis_src], recipe[rvm::system]]

-- snip --

# redis-server -v
Redis server version 2.4.16 (04a08723:1)

# monit -V
This is Monit version 5.5
Copyright (C) 2001-2012 Tildeslash Ltd. All Rights Reserved.

# nginx -v
nginx version: nginx/1.2.3
```

あとは設定ファイルをテンプレートにレシピを書いて、構成をバージョン管理していく。
