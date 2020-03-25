<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


Chef,Puppetに代表される自動構築・構成管理ツールを使うと開発したサーバを検証用などの目的で簡単に再現可能になります。

ただ、漠然としたサーバ構築をしていると何をサービス提供しているのかという定義が曖昧になるため、Cucumber等を使ったテストを軸にテスト駆動でのサーバ構築をしてみましょう。

応用すれば既存のサーバをCucumberによってモデリングし、Chefによって繰り返し再現可能な状態に持っていけます。

このコンテンツで使ったコードはGithubの [https://github.com/higanworks/test_driven_infrastructure_example](https://github.com/higanworks/test_driven_infrastructure_example) で公開しています、参考にしてみたりフィードバックしてもらえると助かります。

## ツール

- **Cucumber**: "ふるまい"を自然言語のように記述し、結果をテストする
- **ChefSpec**: ChefのCookbookのテストができるツール、目的のリソース定義がCookbookに記述されているかを確認する
- **Foodcritic**: CookbookレシピのSyntaxほか、全体的な書き方のチェックをする、矛盾なども指摘してくれる
- **Chef(Solo,Cookbook)**: CookBookを参照し、サーバの役割をコンバージェンスする


## モデル

練習用にこんなサーバを考えます。

- nginxをインストールして、HTTPをホスティングする
- "www.example.com" のバーチャルホストを名前ベースで提供する

新規サーバならこれで良し、既存のサーバなら上記のようにモデリングしてみましょう。


## Cucumber

上記モデルで"ふるまい"を定義するとはどういうことでしょう。
テストするなら - nginxのパッケージをインストールして、コンフィグを置いてあれこれパーミッションを確認してサービスをスタートして．．． と考えてしまいがちです。

ここはインフラ・サーバエンジニア脳ではなく、エンドユーザの視点で見ることが大事です。中で何が動いているかというのは、"ふるまい"の本質ではありません。

それを踏まえた"ふるまい"の定義はこうなります。

・HTTPでTCP/80番をサービスしている
・`Host: www.example.com` ヘッダをつけたHTTPアクセスなら`www.example.com`のコンテンツを返す

足りない？そんなことはありません。


### Cucumber-feature

先ほどの２つをCucumberでテストできるようにしていきます、こんなもんでしょうか。

```cucumber:features/http_server.feature
# language: ja

機能: httpサーバ
  webサーバを提供するために
  HTTPクライアントの立場から
  HTTPのレスポンスをチェックする


  シナリオ: HTTPクライアントリクエスト
    前提: HTTPで localhost にリクエストする
    ならば: レスポンスのステータスが 200 だ

  シナリオ: 名前ベースでHTTPクライアントリクエスト
    前提: Hostヘッダ に "www.example.com" をつけてHTTPで localhost にリクエストする
    ならば: レスポンスのステータスが 200 だ
    かつ: コンテンツに www.example.com が含まれている
```

サーバがこれらの機能を満たしていれば、構築(再現)は完了となるはずです。自分のサーバをレシピにしたい人はこういう形で書きだしてみましょう。



#### Cucumber実行

Cucumberはもう動きます、実行してみましょう。

```shell:Shell-Out(Cucumber)
$ cucumber 
# language: ja
機能: httpサーバ
  webサーバを提供するために
  HTTPクライアントの立場から
  HTTPのレスポンスをチェックする

  シナリオ: HTTPクライアントリクエスト          # features/http_server.feature:9
    前提: HTTPで localhost にリクエストする # features/http_server.feature:10
    ならば: レスポンスのステータスが 200 だ      # features/http_server.feature:11

  シナリオ: 名前ベースでHTTPクライアントリクエスト                                    # features/http_server.feature:13
    前提: Hostヘッダ に "www.example.com" をつけてHTTPで localhost にリクエストする # features/http_server.feature:14
    ならば: レスポンスのステータスが 200 だ                                      # features/http_server.feature:15
    かつ: コンテンツに www.example.com が含まれている                           # features/http_server.feature:16

2 scenarios (2 undefined)
5 steps (5 undefined)
0m0.006s

You can implement step definitions for undefined steps with these snippets:

前提 /^: HTTPで (.*?) にリクエストする$/ do |arg1, arg2, arg3, arg4|
  pending # express the regexp above with the code you wish you had
end

ならば /^: レスポンスのステータスが (\d+) だ$/ do |arg1|
  pending # express the regexp above with the code you wish you had
end

前提 /^: Hostヘッダ に "(.*?)" をつけてHTTPで (.*?) にリクエストする$/ do |arg1, arg2, arg3, arg4, arg5|
  pending # express the regexp above with the code you wish you had
end

ならば /^: コンテンツに www\.example\.com が含まれている$/ do
  pending # express the regexp above with the code you wish you had
end

If you want snippets in a different programming language,
just make sure a file with the appropriate file extension
exists where cucumber looks for step definitions.

```

シナリオ2つ、ステップが5つあることを検出しています。
同時にステップを実行するためのコードを定義するよう求めてきています、次はそれを元にステップを作ります。


### Cucumber-Steps

Featureに書かれている行をコードとして実行するのはStepです、Step実行結果がTrueならそのテストは成功という仕組みです。

先ほどcucumberを実行した際に出力されたStep定義を参考に、Stepファイルを作ります。

捕捉マッチを使って引数を拾い、コードを実行する仕組みなので多少変更します。

```ｒuby:features/step_definitions/http_server_step.rb
# encoding: UTF-8


前提 /^: HTTPで ([\w\.]+) にリクエストする$/ do |host|
  pending # express the regexp above with the code you wish you had
end

ならば /^: レスポンスのステータスが (\d+) だ$/ do |status|
  pending # express the regexp above with the code you wish you had
end

前提 /^: Hostヘッダ に "([\w\.]+)" をつけてHTTPで ([\w\.]+) にリクエストする$/ do |host_header, host|
  pending # express the regexp above with the code you wish you had
end

ならば /^: コンテンツに ([\w\.]+) が含まれている$/ do |expect_string|
  pending # express the regexp above with the code you wish you had
end

```

いまはすべてpendingですが、Cucumberの出力は変わります。

```Shell:Shell-Out(Cucumber)
2 scenarios (2 pending)
5 steps (3 skipped, 2 pending)
```

前提がPendingのテストは以降Skipされます、Pendingをなくしていきます。


#### Pending箇所をコードで記述

それではStepを埋めていきます、リソースの取得に手軽な`open-uri`と結果の評価にわりと便利な`RSpec`の`expectations`を使いましょう。


```ruby:features/step_definitions/http_server_step.rb
# encoding: UTF-8

require 'open-uri'
require 'rspec/expectations'
  
  
前提 /^: HTTPで ([\w\.]+) にリクエストする$/ do |host|
  @resource = open("http://#{host}")
end
    
ならば /^: レスポンスのステータスが (\d+) だ$/ do |status|
  @resource.status[0].should == "200"
end
    
前提 /^: Hostヘッダ に "([\w\.]+)" をつけてHTTPで ([\w\.]+) にリクエストする$/ do |host_header, host|
  @resource = open("http://#{host}","Host" => host_header)
end

ならば /^: コンテンツに ([\w\.]+) が含まれている$/ do |expect_string|
  @resource.read.should match(expect_string)
end
```

Cucumberの出力がまた変わります、PendingをなくしたのでStepが本当に実行されFailとなります。

```Shell:Shell-Out(Cucumber)
2 scenarios (2 failed)
5 steps (2 failed, 3 skipped)
```

ちなみにスタックトレースが出ますが、例外は `Connection refused - connect(2) (Errno::ECONNREFUSED)` です。
HTTPサーバが動いてないので当たり前ですね、これは同時に対象のサーバが**HTTPサーバという"ふるまい"をしているかどうかをCucumberによってテストできるようになった**ということです。

ではその"ふるまい"テストを満たすようにChefのレシピを書いて行きます。

## ChefSpec

早速レシピを記述する・・・にはまだ早いのです。"ふるまい"確認の次はCookbookレシピの"仕様"を定義し、満たすかどうかをテストするためChefSpecを使います。

Chef-Client(及びSolo)はコンバージェンスツールです、要求する仕様が問題なく定義してあればサーバの役割はそのように収束します。
ChefSpecはレシピに期待するリソース設定を記述して、Chef-Clientを実行した際にレシピがそのように書けているかをテストすることができます。

### spec/ の作成

`chefspec`を導入していると`knife cookbook`がちょっとだけ拡張されます。

```
knife cookbook create_specs http_server -o cookbooks/
```

既存のCookbookに対して`create_specs`とすると、レシピの数だけ`spec/`以下にspecファイルが作られます。
中身は例によってペンディングです。

```ruby:spec/default_spec.rb 
require 'chefspec'

describe 'http_server::default' do
  let (:chef_run) { ChefSpec::ChefRunner.new.converge 'http_server::default' }
  it 'should do something' do
    pending 'Your recipe examples go here.'
  end
end
```

`rspec`を実行してみます。

```shell:Shell-Out(RSpec)
$ rspec -fd

http_server::default
  should do something (PENDING: Your recipe examples go here.)

Pending:
  http_server::default should do something
    # Your recipe examples go here.
    # ./spec/default_spec.rb:5

Finished in 0.00109 seconds
1 example, 0 failures, 1 pending
```

このPendingを埋めて行きましょう。

### chefspecの記述

まずレシピに期待する事をまとめていきます。

- WebサーバにはNginxを使いましょう、パッケージからでOK
- 仮想ホスト用のコンフィグをインクルード用に設置
- 追加のコンフィグを設置したらnginxをリスタートする

ChefSpecは[Githubのドキュメント](https://github.com/acrmp/chefspec)を参考に書きます、Chefのリソースプロバイダを動的に取得することができるのでドキュメントに無いリソースと思っても名称さえ合わせればチェック出来ます。


#### nginxをインストールする仕様にする

ChefRunnerが走った際に、`パッケージ[nginx]`がインストールされているように記述するには`install_package`マッチャを使用します。

```ruby:spec/default_spec.rb 
require 'chefspec'

describe 'http_server::default' do
  let (:chef_run) { ChefSpec::ChefRunner.new.converge 'http_server::default' }
  it 'should install nginx' do
    chef_run.should install_package 'nginx'
  end
end
```

`chef_run.should install_package 'nginx'`があると`RSpec`実行結果はこうなります。


```shell:Shell-Out(RSpec)
$ rspec -fd

http_server::default
  should install nginx (FAILED - 1)

Failures:

  1) http_server::default should install nginx
     Failure/Error: chef_run.should install_package 'nginx'
       No package resource named 'nginx' with action :install found.
     # ./spec/default_spec.rb:6:in `block (2 levels) in <top (required)>'

Finished in 0.00575 seconds
1 example, 1 failure

Failed examples:

rspec ./spec/default_spec.rb:5 # http_server::default should install nginx
```

PendingはFailに変わり、内容が **`No package resource named 'nginx' with action :install found.`** となりました。

### Specに合わせてCookbookレシピを記述する

そろそろCookbookレシピの作成にかかります、Specは一つ増やして2つに。

#### Spec

nginxサービスをスタートさせる仕様を追加したら、ネームベース部分は後回しにして一旦ここまでのレシピを書いてみます。

```ruby:spec/default_spec.rb 
require 'chefspec'

describe 'http_server::default' do
  let (:chef_run) { ChefSpec::ChefRunner.new.converge 'http_server::default' }
  it 'should install nginx' do
    chef_run.should start_service 'nginx'
  end

  it 'should start nginx' do
    chef_run.should start_service 'nginx'
  end
end

```

仕様が決まってきたので、いよいよCookbookを書き始めることができます。


#### Recipe::default

先ほどのSpecを満たすためのレシピはこう。


```ruby:recipes/default.rb
package "nginx" do
  action :install
end

service "nginx" do
  action [:enable, :start]
end
```

単純ですが、Cucumberの要求、ChefSpecの仕様を決めるという段取りを経ているので余裕を持って取り組めます。

このレシピを書いたらSpecの出力も変わります。

```shell:Shell-Out(RSpec)
$ rspec -fd --color

http_server::default
  should install nginx
  should start nginx

Finished in 0.00755 seconds
2 examples, 0 failures
```

これでオールグリーンです、同時にCucumber1つ目のFeatureを満たしていることでしょう。

## 2つめのFeature

次のFeatureはヴァーチャルホストの設定です、今までの要領でSpecを追加したものがこちら。`describe 'create virtualhosts'`から以降が追加分ですね。

必要なディレクトリ、ついでに`index.html`。それとnginxに読み込ませるコンフィグを設置してからNginxをリスタートする仕様にしました。

### Spec

```ruby:spec/default_spec.rb 
require 'chefspec'

describe 'http_server::default' do
  let (:chef_run) { ChefSpec::ChefRunner.new.converge 'http_server::default' }
  it 'should install nginx' do
    chef_run.should start_service 'nginx'
  end

  it 'should start nginx' do
    chef_run.should start_service 'nginx'
  end

  it 'should create virtualhost base dir' do
    dir = chef_run.node['nginx']['vhost_dir']
    chef_run.should create_directory dir
    chef_run.directory(dir).should be_owned_by('www-data', 'www-data')
    chef_run.directory(dir).mode.should == "0750"
  end


  describe 'create virtualhosts' do
    site = 'www.example.com'
    it "create vhost directory" do
      dir = ::File.join(chef_run.node['nginx']['vhost_dir'], site)
      chef_run.should create_directory dir
      chef_run.directory(dir).should be_owned_by('www-data', 'www-data')
      chef_run.directory(dir).mode.should == "0750"
    end

    it "create vhost index.html" do
      file = ::File.join(chef_run.node['nginx']['vhost_dir'], site, "index.html")
      chef_run.should create_file_with_content file, site
      chef_run.file(file).should be_owned_by('www-data', 'www-data')
      chef_run.file(file).mode.should == "0640"
    end

    it "create vhost nginx config" do
      file = ::File.join(chef_run.node['nginx']['site_avail_dir'], "#{site}.conf",)
      chef_run.template(file).should be_owned_by("root", "root")
      chef_run.template(file).mode.should == "0640"
      chef_run.template(file).should notify("service[nginx]", :restart)
    end

    it "create vhost nginx config link" do
      file = ::File.join(chef_run.node['nginx']['site_avail_dir'], "#{site}.conf")
      link = ::File.join(chef_run.node['nginx']['site_enable_dir'], "#{site}.conf")
      chef_run.should create_link link
      chef_run.link(link).to.should == file
    end
  end
end
```


### Recipe & Attributes

Specを元に作成したレシピです、さすがにベタ書きは辛くなってきたので少しAttributesに逃がして定義しました。

```ruby:recipes/default.rb 
#
# Cookbook Name:: http_server
# Recipe:: default
#

package "nginx" do
  action :install
end

service "nginx" do
  action [:enable, :start]
end


# vhosts
directory node['nginx']['vhost_dir'] do
  action :create
  owner "www-data"
  group "www-data"
  mode  "0750"
  recursive true
end

["www.example.com"].each do |site|

  directory ::File.join(node['nginx']['vhost_dir'], site) do
    action :create
    owner "www-data"
    group "www-data"
    mode  "0750"
  end

  file ::File.join(node['nginx']['vhost_dir'], site, "index.html") do
    action :create
    content site
    owner "www-data"
    group "www-data"
    mode  "0640"
  end


  template ::File.join(node['nginx']['site_avail_dir'], "#{site}.conf") do
    source "vhost.conf.erb"
    variables({
      :server_name => site,
      :vhost_root => ::File.join(node['nginx']['vhost_dir'], site)
    })
    owner "root"
    group "root"
    mode  "0640"
    notifies :restart, "service[nginx]"   
  end

  link ::File.join(node['nginx']['site_enable_dir'], "#{site}.conf") do
    action :create
    to ::File.join(node['nginx']['site_avail_dir'], "#{site}.conf") 
  end
end
```


```ruby:attributes/default.rb 
default['nginx']['site_avail_dir'] = "/etc/nginx/sites-available"
default['nginx']['site_enable_dir'] = "/etc/nginx/sites-enabled"

default['nginx']['vhost_dir'] = "/var/www/vhosts"
```


## ChefSpec通過、そしてFoodcriticへ

7つのサンプルを全て通過しました、ChefSpecに作成した仕様を満たすレシピの完成です。

```shell::Shell-Out(RSpec)
$ rspec -fd --color

http_server::default
  should install nginx
  should start nginx
  should create virtualhost base dir
  create virtualhosts
    create vhost directory
    create vhost index.html
    create vhost nginx config
    create vhost nginx config link

Finished in 0.07299 seconds
7 examples, 0 failures
```

さあいよいよChef-Client(Solo)を実行に移しましょうか？
その前にもう一つツールを叩いてみましょう、**FoodCritic**です。

### Foodcritic

Foodcriticは実際にレシピを適用した際のチェックと読みやすいレシピにするための指摘を行うツールです。
指摘される内容はドキュメントを参照しながら直していきます。 [http://acrmp.github.com/foodcritic/](http://acrmp.github.com/foodcritic/)


#### Foodcritic実行

Cookbookのルートで実行してみます。

```shell:Shell-Out(Foodcritic)
$ foodcritic ./
FC033: Missing template: ./recipes/default.rb:45
```

`FC033: Missing template` をいただきました、テンプレート元のファイルが無いようですね。
このまま実行していてはエラーで失敗するところでした、これはうっかり。


nginxのコンフィグ元ファイルを適当に作ります、一部Chefによってリプレースするため`erb`形式にしてあります。

```text:templates/default/vhost.conf.erb 
server {
  listen 80;
  server_name <%= @server_name %>;
  root <%= @vhost_root %>;
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }
}

```

これで`Foodcritic`のチェックもクリアしました。


## Chef-Recipeの実行

お気づきとは思いますが、**ここまでChefは一度も実行されず、サーバに何も変更はありません。**(※ChefSpecのRunner等は除く)


### Solo実行ファイルの用意

ではChef-Soloで今回のレシピを適用してみましょう、実行のためのコンフィグとランリストをファイルに書きます。

```ruby:solo.rb 
file_cache_path File.expand_path('tmp_chef/cache', File.dirname(__FILE__))
file_backup_path File.expand_path('tmp_chef/backup', File.dirname(__FILE__))
cookbook_path File.expand_path('cookbooks', File.dirname(__FILE__))

log_location 'chef.log'
```

```json::dna.json 
{
  "run_list" : "recipe[http_server::default]"
}
```


### Solo実行

では満を持してChefを実行し、レシピを適用する。コマンドはコンフィグとランリストを含むJsonを渡すと実行される。 `chef-solo -c solo.rb -j dna.json`。

```shell:Shell-Out(Chef-Solo)
$ chef-solo -c solo.rb -j dna.json
INFO: *** Chef 10.18.2 ***
INFO: Setting the run_list to "recipe[http_server::default]" from JSON
INFO: Run List is [recipe[http_server::default]]
INFO: Run List expands to [http_server::default]
INFO: Starting Chef Run for localhost
INFO: Running start handlers
INFO: Start handlers complete.
INFO: Processing package[nginx] action install (http_server::default line 10)
INFO: Processing service[nginx] action enable (http_server::default line 14)
INFO: Processing service[nginx] action start (http_server::default line 14)
INFO: service[nginx] started
INFO: Processing directory[/var/www/vhosts] action create (http_server::default line 20)
INFO: directory[/var/www/vhosts] created directory /var/www/vhosts
INFO: directory[/var/www/vhosts] owner changed to 33
INFO: directory[/var/www/vhosts] group changed to 33
INFO: directory[/var/www/vhosts] mode changed to 750
INFO: Processing directory[/var/www/vhosts/www.example.com] action create (http_server::default line 30)
INFO: directory[/var/www/vhosts/www.example.com] created directory /var/www/vhosts/www.example.com
INFO: directory[/var/www/vhosts/www.example.com] owner changed to 33
INFO: directory[/var/www/vhosts/www.example.com] group changed to 33
INFO: directory[/var/www/vhosts/www.example.com] mode changed to 750
INFO: Processing file[/var/www/vhosts/www.example.com/index.html] action create (http_server::default line 37)
INFO: entered create
INFO: file[/var/www/vhosts/www.example.com/index.html] owner changed to 33
INFO: file[/var/www/vhosts/www.example.com/index.html] group changed to 33
INFO: file[/var/www/vhosts/www.example.com/index.html] mode changed to 640
INFO: file[/var/www/vhosts/www.example.com/index.html] created file /var/www/vhosts/www.example.com/index.html
INFO: Processing template[/etc/nginx/sites-available/www.example.com.conf] action create (http_server::default line 46)
INFO: template[/etc/nginx/sites-available/www.example.com.conf] updated content
INFO: template[/etc/nginx/sites-available/www.example.com.conf] owner changed to 0
INFO: template[/etc/nginx/sites-available/www.example.com.conf] group changed to 0
INFO: template[/etc/nginx/sites-available/www.example.com.conf] mode changed to 640
INFO: Processing link[/etc/nginx/sites-enabled/www.example.com.conf] action create (http_server::default line 58)
INFO: link[/etc/nginx/sites-enabled/www.example.com.conf] created
INFO: template[/etc/nginx/sites-available/www.example.com.conf] sending restart action to service[nginx] (delayed)
INFO: Processing service[nginx] action restart (http_server::default line 14)
INFO: service[nginx] restarted
INFO: Chef Run complete in 5.2259668 seconds
INFO: Running report handlers
INFO: Report handlers complete
```

特にエラーもなく終わったようだ。
つまりレシピに書いた通りにコンバージェンスが行われているので、特に確認の必要もない。


## 帰ってきたCucumber

これで冒頭で作成したCucumberをついに正常終了させることが出来る。

```shell:Shell-Out(Cucumber)
$ cucumber 
# language: ja
機能: httpサーバ
  webサーバを提供するために
  HTTPクライアントの立場から
  HTTPのレスポンスをチェックする

  シナリオ: HTTPクライアントリクエスト          # features/http_server.feature:9
    前提: HTTPで localhost にリクエストする # features/step_definitions/http_server_step.rb:7
    ならば: レスポンスのステータスが 200 だ      # features/step_definitions/http_server_step.rb:11

  シナリオ: 名前ベースでHTTPクライアントリクエスト                                    # features/http_server.feature:13
    前提: Hostヘッダ に "www.example.com" をつけてHTTPで localhost にリクエストする # features/step_definitions/http_server_step.rb:15
    ならば: レスポンスのステータスが 200 だ                                      # features/step_definitions/http_server_step.rb:11
    かつ: コンテンツに www.example.com が含まれている                           # features/step_definitions/http_server_step.rb:19

2 scenarios (2 passed)
5 steps (5 passed)
0m0.044s
```

これで **Cucumber、オールグリーン** だ！



### 終わりに

ここまでくればひととおり、`infrastructure as code`における **テスト駆動サーバ構築** に触れることが出来た。
おそらくインフラ・サーバエンジニアの大半は『面倒臭い』『難しい』と感じたと思う。

しかしCucumberからchef-solo実行の流れは、要件定義・仕様書・手順書のコード化と実行者の自動化いうことに気付くだろうか。
それらを全てコードベースに置くことで管理可能になることのもたらす恩恵は、この手法を身につける努力に対する見返りとして十分なものになるはずです。
