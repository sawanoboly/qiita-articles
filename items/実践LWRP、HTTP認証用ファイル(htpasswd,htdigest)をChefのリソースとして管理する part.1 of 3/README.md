<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


実践Opscode Chefシリーズ。

LWRPを使うと任意のリソースをプリセットのChefリソース群と同じようにレシピに記述することができます。
それにより、他のリソースと同じようにChefによる収束管理がしやすくなります。つまりは冪等性・バックアップ・ログ出力、そしてWhy-Runに対応できると。

最初に補足しておくと、何でもかんでもChefでやれば良いというものではありません、ただ使うならChefらしく。そうでなければSSHでコマンド叩いて来たほうがよほど手堅く安全です。

>  記事中のCookbookとChefSolo設定一式はこちらに公開しています。
> https://github.com/OpsRockin/lwrp_http_userdb

## まずレシピに書く内容を想像する

今回はHTTPサーバが認証に使う、`.htpasswd`や`.htdigest`を取り上げてLWRPの作り方をフルコースで紹介してみます、ファイルをユーザDBとして扱うケースですね。

まずベーシック認証、必要なのはこのへんですかね。

- ユーザDBに使うファイル
- ユーザ名
- パスワード

じゃあ、こんな風に書けると良いんじゃないかな？

```ruby:recipe_sample_basic.rb
httpsv_auth_basic '/var/www/site1' do
  action :create
  user 'hoge'
  password 'password'
end
```

ほらリソースっぽい、これをChef Clientを実行すればユーザDBが作成/更新されるとレシピもスッキリです。

そして何よりも、レシピが何をするかをぱっと見で分かるようになることで可読性が向上し、作成後の保守性も良好になります。

続いてDigest、こちらは`realm`が必要になるので指定できるとうれしい、と。

```ruby:recipe_sample_digest.rb
httpsv_auth_digest '/var/www/site1' do
  action :delete
  user 'hoge'
  realm 'realm1'
end
```

actionを`:delete`にしてみました、`password`が必要ないので省略されていますね。

ちなみに最終形はこれより少しだけ複雑になります。

## Cookbookを用意し、LWRPで作るリソースの名称を決定する

LWRPはcookbookに格納し、レシピに記述するリソース名は`{cookbook名}_{リソース名}` に自動で変換されます。

先程の妄想レシピでは`httpsv_*`というリソースだったので、`cookbook::httpsv`を作成しました。


```Tree_of_cookbooks
cookbooks/httpsv/
├── CHANGELOG.md
├── README.md
├── attributes
├── metadata.rb
├── providers
├── recipes
│   └── default.rb
└── resources
```

ということは、`resources/`の下に作成するファイルは？ そう、`auth_basic.rb`です。

作りましょう。

```Tree_of_cookbooks
cookbooks/httpsv/
├── CHANGELOG.md
├── README.md
├── attributes
├── metadata.rb
├── providers
├── recipes
│   └── default.rb
└── resources
    └── auth_basic.rb
```

## LWRP/auth_basic を作成する

ファイルを作成したことにより、リソースの定義は出来ました。
手始めにこのリソースは…

- `action :create`を持ち、action省略時のデフォルトは`:create`
- パラメータ、`user`をとる、必須
- パラメータ、`password`をとる

と考えた際の`resources/auth_basic.rb`を作ります。

### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-initial
actions :create
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
```

大体見たとおりですね、attributeに若干ヴァリデーションが入っているのも分かるでしょう。このようにattributeはある程度のチェックを実装してます。

ちなみに使えるオプションはChefのドキュメントを見て下さい。

> [Chef Docs - Lightweight Resources](http://docs.opscode.com/lwrp_custom_resource.html#validation-parameters)



では続いて、`action :create`に対応するアクションを`providers/auth_basic.rb`に実装します。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-initial
action :create do
end
```

カラですか？カラですね。
まあ慌てずに一歩ずつ進めていきましょう、actionに対応するブロックさえあれば、LWRPは動作します。

まず最小限の記述で動作を確認して、手応えを掴んでみましょう。

### レシピに記述し、実行する

リソースには名前が必須です、ここでは名前をユーザDBファイルのパスとしてレシピを書いてみました。

サンプルを各環境で実行できるようにするため、作業中のディレクトリの下に`var/www/site1/.htpasswd` ファイルを作るっぽい名前にしています。

### `recipes/sample1.rb`

```ruby:cookbooks/httpsv/recipes/sample1.rb
base_dir = ENV['PWD']

httpsv_auth_basic File.join(base_dir, 'var/www/site1/.htpasswd') do
  user 'hoge'
  password 'password'
end
```
初めに思い描いたレシピのとおりです！


#### chef-solo run

実行できます。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample01' 
Starting Chef Client, version 11.6.0
[2013-09-06T18:05:26+09:00] WARN: Run List override has been provided.
[2013-09-06T18:05:26+09:00] WARN: Original Run List: []
[2013-09-06T18:05:26+09:00] WARN: Overridden Run List: [recipe[httpsv::sample01]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample01
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create (up to date)
Chef Client finished, 0 resources updated
```

正常に終了しました、`Converging 1 resources` からの `0 resources updated`!!

もちろんリソースの内訳はレシピの記述通りです。
- `httpsv_auth_basic[(中略)/var/www/site1/.htpasswd]`

しかし更新したリソースが0個と言われています、実行＝更新では無いのでしょうか？

### updated_by_last_actionでリソースの更新フラグを管理

実は更新カウントの仕組みは`Provider`のactionに委ねられています。

`:create`アクションに一行追加して、もう一度実行してみましょう。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-last_action
action :create do
  @new_resource.updated_by_last_action(true)
end
```

2行目が追加行です、メソッド名からなんとなく分かりますかね。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample01' 
Starting Chef Client, version 11.6.0
[2013-09-06T18:20:31+09:00] WARN: Run List override has been provided.
[2013-09-06T18:20:31+09:00] WARN: Original Run List: []
[2013-09-06T18:20:31+09:00] WARN: Overridden Run List: [recipe[httpsv::sample01]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample01
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create

Chef Client finished, 1 resources updated
```

やった！ **`1 resources updated`** として認識されました。
ProviderがConvergenceを実施したら、そのことを通知してあげる必用があるんですね。


## createアクションの中身を作る。

ではそろそろ`:create`アクションに何かさせてみましょう。
呼ばれたことをログに出力しつつ、.htpasswdを実際に作っちゃいます。


### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-create_htpasswd
require 'webrick'

action :create do
  Chef::Log.warn "====== Create http_auth user #{@new_resource.user}"

  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.name)
  htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
  htpasswd.flush
  @new_resource.updated_by_last_action(true)
end
```

ユーザDBを読み込み、操作してFlushするライブラリを使って簡単に。`WEBrick::HTTPAuth::Htpasswd`についてはドキュメントで確認しましょう。
[Class: WEBrick::HTTPAuth::Htpasswd (Ruby 1.9.3)](http://ruby-doc.org/stdlib-1.9.3/libdoc/webrick/rdoc/WEBrick/HTTPAuth/Htpasswd.html)

`Htdigest`クラスもあるので、Digestの時はそちらを使いましょう。

#### chef-solo run

先程のレシピでまたchef-soloを実行してみます。
※CreateのログをWARNにしているのは練習だからです、通常はINFOなりで。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample01' 
Starting Chef Client, version 11.6.0
[2013-09-06T18:46:35+09:00] WARN: Run List override has been provided.
[2013-09-06T18:46:35+09:00] WARN: Original Run List: []
[2013-09-06T18:46:35+09:00] WARN: Overridden Run List: [recipe[httpsv::sample01]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample01
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
[2013-09-06T18:46:35+09:00] WARN: ====== Create http_auth user hoge


Chef Client finished, 1 resources updated
```

出来上がったファイルを見てみましょう。

```var/www/site1/.htpasswd 
hoge:iLWIQVkBPUswQ
```

はい、できていますね。

## Why-Runをサポートしよう

Providerが小さい今のうちに、Why-Runサポートに行きましょう。

Why-Runをサポートするためには、実際に処理を行う部分を`converge_by`以下のブロックにすればOKです。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-support_why-run
require 'webrick'

def whyrun_supported?
  true
end

action :create do
  converge_by("====== Create http_auth user #{@new_resource.user}") do
    htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.name)
    htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
    htpasswd.flush
  end
end
```

出だしの`def whyrun_supported?`はそのままなので気にしないとして`converge_by`によって2つ消えたものがあります、気づきましたか？

- Chef::Logの記述
- updated_by_last_action

が無くなっています。ログはWhy-run演出のためちょっと消しただけなのですが、先程紹介したばかりの重要なフラグはどこへ？

実は処理を`converge_by`のブロックにすると、updated_by_last_actionのフラグを自動で立ててくれるので不要になります。助かりますね。

#### chef-solo run

まずWhy-run(`-W`オプション)から。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample01' -W
Starting Chef Client, version 11.6.0
[2013-09-06T19:06:49+09:00] WARN: Run List override has been provided.
[2013-09-06T19:06:49+09:00] WARN: Original Run List: []
[2013-09-06T19:06:49+09:00] WARN: Overridden Run List: [recipe[httpsv::sample01]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample01
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - Would ====== Create http_auth user hoge

Chef Client finished, 1 resources would have been updated
```

表示予定のメッセージに`Would `がついており、レポートも`1 resources would have been updated`と表記が変わっています。

次は`-W`を外してみましょう。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample01' 
Starting Chef Client, version 11.6.0
[2013-09-06T19:10:39+09:00] WARN: Run List override has been provided.
[2013-09-06T19:10:39+09:00] WARN: Original Run List: []
[2013-09-06T19:10:39+09:00] WARN: Overridden Run List: [recipe[httpsv::sample01]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample01
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - ====== Create http_auth user hoge

Chef Client finished, 1 resources updated
```

`1 resources updated`と、Why-Runサポート前と同じく、ちゃんと実行されていますね。

これでWhy-Runがサポートされました、他のアクション定義の処理部分でも`converge_by`を使うのをお忘れなく。


## actionの中でいつものリソースを使おう

LWRPおよびLibrariesでは、普通のレシピのようにいつものリソースを使うことも出来ます。処理のされ方が少し違う点に注意すればtemplate等は特に活用できるのではないでしょうか。

例えば作ったファイルのパーミッションを特定の状態に保ちたいとき、次のように`file`リソースを記述するとちゃんと処理されます。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-use_normal_resource
require 'webrick'

def whyrun_supported?
  true
end

action :create do
  converge_by("====== Create http_auth user #{@new_resource.user}") do
    htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.name)
    htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
    htpasswd.flush

    file @new_resource.name do
      owner 'guest'
      mode  0644
    end
  end
end
```

`WEBrick::HTTPAuth::Htpasswd`は実行したユーザをオーナに、かつパーミッション`600`で毎回ファイルを修正します。
それはそれで困るなあ、という時に上記のようにレシピをいれこむ事が可能です。

※ 理由は後述しますが、このLWRPではfileリソースの使用は向いていません。

#### chef-solo run

```shell:run_chef-solo
$ rvmsudo PWD=`pwd` chef-solo -c solo.rb -o 'httpsv::sample01' 
Starting Chef Client, version 11.6.0
[2013-09-06T19:49:13+09:00] WARN: Run List override has been provided.
[2013-09-06T19:49:13+09:00] WARN: Original Run List: []
[2013-09-06T19:49:13+09:00] WARN: Overridden Run List: [recipe[httpsv::sample01]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample01
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - ====== Create http_auth user hoge

Recipe: <Dynamically Defined Resource>
  * file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - change mode from '0600' to '0644'
    - change owner from 'root' to 'guest'

Chef Client finished, 2 resources updated
```

間違いなくレシピDSLが実行されていることが分かります。

※ rootでファイルを作ってもらうため、少々強引ですが`rvmsudo`から実行しています。

## リソースの重複を避けよう

今の状態のLWRPで一度にでユーザを複数作成してみましょう。

### `recipes/sample2.rb`

```ruby:cookbooks/httpsv/recipes/sample2.rb
base_dir = ENV['PWD']

httpsv_auth_basic File.join(base_dir, 'var/www/site1/.htpasswd') do
  user 'hoge1'
  password 'password1'
end

httpsv_auth_basic File.join(base_dir, 'var/www/site1/.htpasswd') do
  user 'hoge2'
  password 'password2'
end
```

providerはちょっとfileリソースを追加する前の状態に戻しています。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-support_why-run
require 'webrick'

def whyrun_supported?
  true
end

action :create do
  converge_by("====== Create http_auth user #{@new_resource.user}") do
    htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.name)
    htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
    htpasswd.flush
  end
end
```


#### chef-solo run

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample02' 
Starting Chef Client, version 11.6.0
[2013-09-06T20:14:45+09:00] WARN: Run List override has been provided.
[2013-09-06T20:14:45+09:00] WARN: Original Run List: []
[2013-09-06T20:14:45+09:00] WARN: Overridden Run List: [recipe[httpsv::sample02]]
Compiling Cookbooks...
[2013-09-06T20:14:45+09:00] WARN: Cloning resource attributes for httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] from prior resource (CHEF-3694)
[2013-09-06T20:14:45+09:00] WARN: Previous httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample02.rb:3:in `from_file'
[2013-09-06T20:14:45+09:00] WARN: Current  httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample02.rb:8:in `from_file'
Converging 2 resources
Recipe: httpsv::sample02
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - ====== Create http_auth user hoge1

  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - ====== Create http_auth user hoge2

Chef Client finished, 2 resources updated
```

リソース定義、`httpsv_auth_basic[/(略)/var/www/site1/.htpasswd]` が重複している旨がWARNINGで出てしまいました。

### リソース名重複で何が困るの？

少し脱線しますが、Chefの実行単位で同一のリソース名を使うと警告が出ます。
一応そのまま実行できるため特に悪いことは起こりませんが、将来的にChefの挙動が変更される可能性もあります。

他、現状でも下記のような弊害があることにはあります。

- 定義済みリソース情報の再利用で困る
- Chefの実行レポートで、ぱっと見の区別がつかない

サンプルレシピで簡単に説明してみます。

### `recipes/sample04.rb`

```ruby:cookbooks/httpsv/recipes/sample04.rb
# coding: utf-8
base_dir = ENV['PWD']
file_path = File.join(base_dir, 'var/www/site1/.htpasswd')

def mode_to_log(file_path)
  Chef::Log.warn "========= new resource mode is " + resources("file[#{file_path}]").mode
end

## >> レシピ部分
file file_path do ; mode '0660' ; end ; mode_to_log(file_path)
file file_path do ; mode '0666' ; end ; mode_to_log(file_path)
file file_path do ; mode '0644' ; end ; mode_to_log(file_path)
file file_path do ; mode '0640' ; end ; mode_to_log(file_path)
## << レシピ部分

require 'chef/handler'

class Chef::Handler::LogReport < ::Chef::Handler
  def report
    Chef::Log.warn '======= Update Resources are following...'
    data[:updated_resources].each.with_index do |r,idx|
      Chef::Log.warn [idx, r.to_s].join(':')
    end
  end
end

Chef::Config[:report_handlers] << Chef::Handler::LogReport.new
```

まず`resources("file[#{file_path}]")`の情報を再利用した場合、レシピ中のタイミング次第で内容が変わってしまうことをログに出力しています。
次にChefのレポートハンドラを追加して、今回のChef実行で更新したリソースのリストをログに出力してみます。

#### chef-solo run

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample04' 
Starting Chef Client, version 11.6.0
[2013-09-07T18:33:29+09:00] WARN: Run List override has been provided.
[2013-09-07T18:33:29+09:00] WARN: Original Run List: []
[2013-09-07T18:33:29+09:00] WARN: Overridden Run List: [recipe[httpsv::sample04]]
Compiling Cookbooks...
[2013-09-07T18:33:30+09:00] WARN: ========= new resource mode is 0660
[2013-09-07T18:33:30+09:00] WARN: Cloning resource attributes for file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] from prior resource (CHEF-3694)
[2013-09-07T18:33:30+09:00] WARN: Previous file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample04.rb:12:in `from_file'
[2013-09-07T18:33:30+09:00] WARN: Current  file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample04.rb:13:in `from_file'
[2013-09-07T18:33:30+09:00] WARN: ========= new resource mode is 0666
[2013-09-07T18:33:30+09:00] WARN: Cloning resource attributes for file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] from prior resource (CHEF-3694)
[2013-09-07T18:33:30+09:00] WARN: Previous file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample04.rb:13:in `from_file'
[2013-09-07T18:33:30+09:00] WARN: Current  file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample04.rb:14:in `from_file'
[2013-09-07T18:33:30+09:00] WARN: ========= new resource mode is 0644
[2013-09-07T18:33:30+09:00] WARN: Cloning resource attributes for file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] from prior resource (CHEF-3694)
[2013-09-07T18:33:30+09:00] WARN: Previous file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample04.rb:14:in `from_file'
[2013-09-07T18:33:30+09:00] WARN: Current  file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]: /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/cookbooks/httpsv/recipes/sample04.rb:15:in `from_file'
[2013-09-07T18:33:30+09:00] WARN: ========= new resource mode is 0640
Converging 4 resources
Recipe: httpsv::sample04
  * file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - change mode from '0640' to '0660'

  * file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - change mode from '0660' to '0666'

  * file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - change mode from '0666' to '0644'

  * file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] action create
    - change mode from '0644' to '0640'

[2013-09-07T18:33:30+09:00] WARN: ======= Update Resources are following...
[2013-09-07T18:33:30+09:00] WARN: 0:file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]
[2013-09-07T18:33:30+09:00] WARN: 1:file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]
[2013-09-07T18:33:30+09:00] WARN: 2:file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]
[2013-09-07T18:33:30+09:00] WARN: 3:file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd]
Chef Client finished, 4 resources updated
```

`(CHEF-3694)`のWARNが出ているのはさっきのとおりですね。
そして特定のファイルに設定されるべきmodeが、Chefの実行中に次々と変化している様子が見て取れます。これは好ましくない。
最後にレポートハンドラが今回の実行で更新したリソースを並べていますが、全て一緒に見えます。これでは折角のレポートが見づらいものになってしまいます。

このようにChefの実行単位内でリソースの名前はなるべく一意にしておいたほうが良いのです。
そろそろ本題に戻りましょう。


### リソース名の重複を解決する

name属性をそのままファイルのパスに使ってしまうと、リソース名が重複してしまうことが分かりました。
LWRPの特性上設定自体は出来ましたが、一度のChef実行では同一名リソース定義に対しての複数回更新は推奨できません。

以下の方針で対応してみましょう。

- 新しい属性`:path`を作り、ユーザDBをそちらで指定する
- nameは一意にするためユーザ名(+レルム)を加える

まずリソース定義に`:path`を追加します。

### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-add_path
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
attribute :path, :kind_of => String, :required => true

続いてproviderも変更、`:name`部分を`:path`に変えるだけですね。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-add_path
require 'webrick'

def whyrun_supported?
  true
end

action :create do
  converge_by("====== Create http_auth user #{@new_resource.user}") do
    htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
    htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
    htpasswd.flush
  end
end
```

次はLWRPの変更に合わせ、レシピも修正しましょう。

### 内部DSLのチカラ

さてレシピの修正ですが、修正箇所が少し多くなりそうなので少しインチキしてみます。


### `recipes/sample3.rb`

```ruby:cookbooks/httpsv/recipes/sample3.rb
base_dir = ENV['PWD']

httpsv_auth_basic File.join(base_dir, 'var/www/site1/.htpasswd') do
  user 'hoge1'
  path self.name.to_s
  name [self.path, self.user].join(':')
  password 'password1'
end

httpsv_auth_basic File.join(base_dir, 'var/www/site1/.htpasswd') do
  user 'hoge2'
  path self.name.to_s
  name [self.path, self.user].join(':')
  password 'password2'
end
```

既存部分を一切修正すること無く、2行ずつの追加で済ませてみました。

元々nameに使われている部分を`self.name`でpathに指定し、その後で`name`そのものをユーザ名と結合して上書きしてしまいます。
物は試し、実行してみましょう。


#### chef-solo run

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample03' 
Starting Chef Client, version 11.6.0
[2013-09-07T16:05:37+09:00] WARN: Run List override has been provided.
[2013-09-07T16:05:37+09:00] WARN: Original Run List: []
[2013-09-07T16:05:37+09:00] WARN: Overridden Run List: [recipe[httpsv::sample03]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample03
  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd:hoge1] action create
    - ====== Create http_auth user hoge1

  * httpsv_auth_basic[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd:hoge2] action create
    - ====== Create http_auth user hoge2

Chef Client finished, 2 resources updated

$ cat var/www/site1/.htpasswd 
hoge1:LfhaERgNIfVDo
hoge2:UsNzKe0MKEkaU
```

狙い通りにリソースがそれぞれ一意になり、かつ対象のユーザDBも作成できました。
Chefはレシピでのリソース定義を分かりやすく保ちながら、どこでもRubyの高い拡張性を活かすことができます。<del>これが内部DSLの力だ！(©実践Vim#TIP30)</del>

ちなみにここのselfは`Chef::Resource::HttpsvAuthBasic`というクラスのインスタンスをになっとります。LWRPはファイル名がClassifyされて、他のリソースと同じようにChef::Resourceのサブクラスとして、Cookbookのコンパイル時に追加されるようになっているんですね。

ちょっと悪乗りしてレシピ修正の手を抜いてみましたが、こういうこともできるという参考程度で覚えておくと良いでしょう。

----

そろそろ長くなってきたので記事を分けます、Part2へどうぞ。

[実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.2 of 3](http://qiita.com/sawanoboly@github/items/79c7cdf782d64980677d)

----

実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.1 of 3
[実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.3 of 3](http://qiita.com/sawanoboly@github/items/89095877826cab27c932)
