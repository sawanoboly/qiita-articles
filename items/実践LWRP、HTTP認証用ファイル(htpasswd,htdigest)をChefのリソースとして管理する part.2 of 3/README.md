<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

実践Opscode Chefシリーズ、[part.1](http://qiita.com/sawanoboly@github/items/9bdaebcfc98d73f84843)の続きになります。

>  記事中のCookbookとChefSolo設定一式はこちらに公開しています。
> https://github.com/OpsRockin/lwrp_http_userdb


リソースの名前重複を解決した所で切ったので、今のResource/Providerはこんな感じです。


### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-end_of_1
actions :create
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
attribute :path, :kind_of => String, :required => true
```

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-end_of_1
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

早速ですが、この`:create`アクションは欠陥品です。何故でしょうか？

そう、対象のユーザが既にユーザDBに存在しようが気にしていませんね。`:create`アクションが取るべき行動は、『なければ作成』『変更があれば適用』であり、最後の『変更がなければ何もしない』の場合はupdated_resoucesにカウントされるないことが望ましいのです。


> 参考資料：
> このあと@current_resource , @new_resource を扱う話になります、この概念を知らないなら下記スライドを一読してから先に進むことをお奨めします。
> 
> [What is Chef?(日本語です)](http://www.slideshare.net/YukihikoSawanobori/what-is-chef201303)


## current_resource == new_resource

では`:create`アクションに`@current_resouce`の取得と比較する処理を入れていきます。

必要な考え方はこのとおりです。

- ユーザDBをロードする
- レシピに書いたユーザが存在しなかったら追加
- レシピに書いたユーザが存在したら…
  - パスワードが変更されていたら更新
  - パスワードが変更されていなかったら何もしない(up to date)

やり方は色々考えられますが、一つずつシンプルに片付けて行きましょう。


### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-add_attr_accessor
actions :create
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
attribute :path, :kind_of => String, :required => true

attr_accessor :crypted_passwd
```

attr_accessor が追加されました、これはレシピでは指定することは無いけれど、`current_resouce`の状態を入れるなどに使う属性を用意する時に使います。
今回ユーザDBから取得できるものはハッシュ化されたパスワードになるため、レシピで定義する属性と正面切って比較ができません。なので`current_resouce`のいち属性としてそれを格納します。


### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-case_sensitive_create
require 'webrick'

def whyrun_supported?
  true
end

action :create do
  if @current_resource.crypted_passwd
    Chef::Log.warn "====== Found http_auth user #{@new_resource.user}"
    salt = @current_resource.crypted_passwd[0..1]
    if @current_resource.crypted_passwd == @new_resource.password.crypt(salt)
      Chef::Log.warn "====== http_auth user #{@new_resource.user} was not modified. (up to date)"
    else
      converge_by("====== Update http_auth user #{@new_resource.user}") do
        Chef::Log.warn "====== Update http_auth user #{@new_resource.user}"
        flush_userdb!
      end
    end
  else
    converge_by("====== Create http_auth user #{@new_resource.user}") do
      Chef::Log.warn "====== Create http_auth user #{@new_resource.user}"
      flush_userdb!
    end
  end
end

def flush_userdb!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
  htpasswd.flush
end

def load_current_resource
  @current_resource = Chef::Resource::HttpsvAuthBasic.new(@new_resource.name)
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  @current_resource.crypted_passwd = htpasswd.get_passwd nil, @new_resource.user, true
end
```

いきなり行数が増えてしまいましたが、それぞれ大したことはしていないので解説します。

### `def load_current_resource`

`def load_current_resource`はリソースを利用する際に自動で呼ばれる関数です。`@current_resouce`属性の収集処理を書いておきましょう。


### 新規作成

今回は対象のユーザDBにユーザが存在しない場合、`get_passwd`が`nil`を返してくるため、初っ端の分岐条件に使用できました。新規作成はこれでクリアです。

### 更新するかしないか

既にユーザがいた場合、暗号化されたパスワードをもらえるのでそれをレシピの定義と比較することで更新するかどうかの判断をします。

htpasswdのパスワードフィールドはこういう仕様でした。

- 最初の2文字がsalt
- そのsaltを使ってcrypt処理

ということはこれで条件比較ができますね。
`if @current_resource.crypted_passwd == @new_resource.password.crypt(salt)`

あとはマッチすれば何もしない、そうでなければ更新処理を行うと。

#### Digest認証の時

ちなみにDigest認証の際はパスワードフィールドはMD5になります、生成規則はこうです。

`Digest::MD5::hexdigest([user, realm, pass].join(":"))`

Basicと同じようにレシピの情報で十分照合が可能ですね。


### `recipes/sample05.rb`

```ruby:cookbooks/httpsv/recipes/sample05.rb
base_dir = ENV['PWD']

httpsv_auth_basic 'var/www/site1' do
  user 'hoge1'
  path File.join(base_dir, self.name, '.htpasswd')
  name [self.name, self.user].join(':')
  password 'password1'
end

httpsv_auth_basic 'var/www/site1' do
  user 'hoge2'
  path File.join(base_dir, self.name, '.htpasswd')
  name [self.name, self.user].join(':')
  password rand.to_s
end
```

user 'hoge2'のパスワードを毎回ランダムで生成するようにして、Chefを何度か実行してみます。


#### chef-solo run

まず初回

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample05'
Starting Chef Client, version 11.6.0
[2013-09-07T23:07:52+09:00] WARN: Run List override has been provided.
[2013-09-07T23:07:52+09:00] WARN: Original Run List: []
[2013-09-07T23:07:52+09:00] WARN: Overridden Run List: [recipe[httpsv::sample05]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample05
  * httpsv_auth_basic[var/www/site1:hoge1] action create
[2013-09-07T23:07:52+09:00] WARN: ====== Create http_auth user hoge1

    - ====== Create http_auth user hoge1

  * httpsv_auth_basic[var/www/site1:hoge2] action create
[2013-09-07T23:07:52+09:00] WARN: ====== Create http_auth user hoge2

    - ====== Create http_auth user hoge2

Chef Client finished, 2 resources updated
```

#### chef-solo run

2回め、user1, user2ともに存在。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample05'
Starting Chef Client, version 11.6.0
[2013-09-07T23:08:08+09:00] WARN: Run List override has been provided.
[2013-09-07T23:08:08+09:00] WARN: Original Run List: []
[2013-09-07T23:08:08+09:00] WARN: Overridden Run List: [recipe[httpsv::sample05]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample05
  * httpsv_auth_basic[var/www/site1:hoge1] action create
[2013-09-07T23:08:08+09:00] WARN: ====== Found http_auth user hoge1
[2013-09-07T23:08:08+09:00] WARN: ====== http_auth user hoge1 was not modified. (up to date)
 (up to date)
  * httpsv_auth_basic[var/www/site1:hoge2] action create
[2013-09-07T23:08:08+09:00] WARN: ====== Found http_auth user hoge2
[2013-09-07T23:08:08+09:00] WARN: ====== Update http_auth user hoge2

    - ====== Update http_auth user hoge2

Chef Client finished, 1 resources updated
```

#### chef-solo run

2回め以降のWhy-Run

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample05' -W
Starting Chef Client, version 11.6.0
[2013-09-07T23:08:50+09:00] WARN: Run List override has been provided.
[2013-09-07T23:08:50+09:00] WARN: Original Run List: []
[2013-09-07T23:08:50+09:00] WARN: Overridden Run List: [recipe[httpsv::sample05]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample05
  * httpsv_auth_basic[var/www/site1:hoge1] action create
[2013-09-07T23:08:50+09:00] WARN: ====== Found http_auth user hoge1
[2013-09-07T23:08:50+09:00] WARN: ====== http_auth user hoge1 was not modified. (up to date)
 (up to date)
  * httpsv_auth_basic[var/www/site1:hoge2] action create
[2013-09-07T23:08:50+09:00] WARN: ====== Found http_auth user hoge2

    - Would ====== Update http_auth user hoge2

Chef Client finished, 1 resources would have been updated
```

できました、意外と簡単ですよね。`action :create`はこれで’ほぼ’完成したと言えるでしょう。
何が足りないかって？ それは後ほど。

## deleteアクションの中身を作る

そろそろ`:delete`を作ってもいいでしょう。`:create`が作れるならもう簡単ですよね。

できました。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-action_delete
require 'webrick'

def whyrun_supported?
  true
end

action :create do
  if @current_resource.crypted_passwd
    Chef::Log.warn "====== Found http_auth user #{@new_resource.user}"
    salt = @current_resource.crypted_passwd[0..1]
    if @current_resource.crypted_passwd == @new_resource.password.crypt(salt)
      Chef::Log.warn "====== http_auth user #{@new_resource.user} was not modified. (up to date)"
    else
      converge_by("====== Update http_auth user #{@new_resource.user}") do
        Chef::Log.warn "====== Update http_auth user #{@new_resource.user}"
        update_user!
      end
    end
  else
    converge_by("====== Create http_auth user #{@new_resource.user}") do
      Chef::Log.warn "====== Create http_auth user #{@new_resource.user}"
      update_user!
    end
  end
end

action :delete do
  unless @current_resource.crypted_passwd
    Chef::Log.warn "====== http_auth user #{@new_resource.user} was not found. Nothing to do. (up to date)"
  else
    converge_by("====== Delete http_auth user #{@new_resource.user}") do
      Chef::Log.warn "====== Delete http_auth user #{@new_resource.user}"
      delete_user!
    end
  end
end

def update_user!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
  htpasswd.flush
end

def delete_user!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.delete_passwd nil, @new_resource.user
  htpasswd.flush
end

def load_current_resource
  @current_resource = Chef::Resource::HttpsvAuthBasic.new(@new_resource.name)
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  @current_resource.crypted_passwd = htpasswd.get_passwd nil, @new_resource.user, true
end
```

いなければ何もしない、いれば消す。アクションにあわせて`flush_userdb!`メソッドを2つに分けました。


#### chef-solo run

まずWhy-Runから。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample06' -W
Starting Chef Client, version 11.6.0
[2013-09-07T23:29:17+09:00] WARN: Run List override has been provided.
[2013-09-07T23:29:17+09:00] WARN: Original Run List: []
[2013-09-07T23:29:17+09:00] WARN: Overridden Run List: [recipe[httpsv::sample06]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample06
  * httpsv_auth_basic[var/www/site1:hoge1] action delete
    - Would ====== Delete http_auth user hoge1

  * httpsv_auth_basic[var/www/site1:hoge2] action delete
    - Would ====== Delete http_auth user hoge2

Chef Client finished, 2 resources would have been updated
```

#### chef-solo run

実際に消してみます。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample06' 
Starting Chef Client, version 11.6.0
[2013-09-07T23:46:12+09:00] WARN: Run List override has been provided.
[2013-09-07T23:46:12+09:00] WARN: Original Run List: []
[2013-09-07T23:46:12+09:00] WARN: Overridden Run List: [recipe[httpsv::sample06]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample06
  * httpsv_auth_basic[var/www/site1:hoge1] action delete[2013-09-07T23:46:12+09:00] WARN: ====== Delete http_auth user hoge1

    - ====== Delete http_auth user hoge1

  * httpsv_auth_basic[var/www/site1:hoge2] action delete[2013-09-07T23:46:12+09:00] WARN: ====== Delete http_auth user hoge2

    - ====== Delete http_auth user hoge2

Chef Client finished, 2 resources updated
```


#### chef-solo run

ユーザ削除以降の実行では、特に何もしません。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample06' 
Starting Chef Client, version 11.6.0
[2013-09-07T23:46:16+09:00] WARN: Run List override has been provided.
[2013-09-07T23:46:16+09:00] WARN: Original Run List: []
[2013-09-07T23:46:16+09:00] WARN: Overridden Run List: [recipe[httpsv::sample06]]
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample06
  * httpsv_auth_basic[var/www/site1:hoge1] action delete[2013-09-07T23:46:17+09:00] WARN: ====== http_auth user hoge1 was not found. Nothing to do. (up to date)
 (up to date)
  * httpsv_auth_basic[var/www/site1:hoge2] action delete[2013-09-07T23:46:17+09:00] WARN: ====== http_auth user hoge2 was not found. Nothing to do. (up to date)
 (up to date)
Chef Client finished, 0 resources updated
```

これで`:delete`アクションもできましたね。

## ユーザDBのパーミッションを調整しよう

ライブラリの都合で、ユーザDBのパーミッションが毎回Chefの実行ユーザの'0600'になっています。調整できるようにしておきます。

### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-fix_permission
actions :create, :delete
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
attribute :path, :kind_of => String, :required => true
attribute :filemode, :default => 0600

attr_accessor :crypted_passwd
```

filemode属性を追加して、省略時は'0600' がセットされるようにしておきます。

### `providers/auth_basic.rb`

一部抜粋にします。

```ruby:cookbooks/httpsv/providers/auth_basic.rb-fix_permission
# -- snip --

def update_user!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
  htpasswd.flush
  fix_permission!
end

def delete_user!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.delete_passwd nil, @new_resource.user
  htpasswd.flush
  fix_permission!
end

def fix_permission!
  FileUtils.chmod(@new_resource.filemode.to_i, @new_resource.path)
end

# -- snip --
```

`flush`直後に`#fix_permission!`を呼ぶように書き換えます。ownerやgroupを調整したい場合は`#fix_permission!`に処理を追加していけば良いでしょう。

これでレシピ側でユーザDBファイルのパーミッションを変更できるようになります。

### `recipes/sample07.rb`

```ruby:cookbooks/httpsv/recipes/sample07.rb
base_dir = ENV['PWD']

httpsv_auth_basic 'var/www/site1' do
  user 'hoge1'
  path File.join(base_dir, self.name, '.htpasswd')
  name [self.name, self.user].join(':')
  password 'password1'
  filemode 0640
end

httpsv_auth_basic 'var/www/site1' do
  user 'hoge2'
  path File.join(base_dir, self.name, '.htpasswd')
  name [self.name, self.user].join(':')
  password rand.to_s
  filemode 0640
end
```

既存DBを変更したい際はconvergenceを発生させないと権限が変わらないので、リソース比較の処理を追加しても良いでしょうね。

さて、そろそろ`:create`アクションがほぼ'完成'としたことに触れたいと思います。

----

またしても長くなってきたので記事を分けます、Part.3へどうぞ。

[実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.3 of 3](http://qiita.com/sawanoboly@github/items/89095877826cab27c932)

----

[実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.1 of 3](http://qiita.com/sawanoboly@github/items/9bdaebcfc98d73f84843)
実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.2 of 3
