<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

実践Opscode Chefシリーズ、[part.2](http://qiita.com/sawanoboly@github/items/79c7cdf782d64980677d)の続きになります。

>  記事中のCookbookとChefSolo設定一式はこちらに公開しています。
> https://github.com/OpsRockin/lwrp_http_userdb


`:create`アクションに足りないものを書こうとしたところで切りました、今のResource/Providerはこんな感じです。


### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-end_of_2
actions :create, :delete
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
attribute :path, :kind_of => String, :required => true
attribute :filemode, :default => 0600

attr_accessor :crypted_passwd
```

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-end_of_2
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

def load_current_resource
  @current_resource = Chef::Resource::HttpsvAuthBasic.new(@new_resource.name)
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  @current_resource.crypted_passwd = htpasswd.get_passwd nil, @new_resource.user, true
end
```

## バックアップをつくる？

今足りない要素、それはユーザDBのバックアップです。多分。

### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-add_backup
# --snip --

def update_user!
  backup!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
  htpasswd.flush
  fix_permission!
end

def delete_user!
  backup!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.delete_passwd nil, @new_resource.user
  htpasswd.flush
  fix_permission!
end

def fix_permission!
  FileUtils.chmod(@new_resource.filemode.to_i, @new_resource.path)
end

def backup!
  htpasswd_file = Chef::Resource::File.new(@new_resource.path)
  backup = Chef::Util::Backup.new(htpasswd_file)
  backup.backup!
end

# --snip --
```

`Chef::Util::Backup`というのがありまして、引数に`Chef::Resource::File`のインスタンス(※正確には少し違う)を入れてあげるといつものリソース(file, cookbook_file, template, etc...)形式のバックアップを取ることができます。
これを使わない手はありませんね。

#### chef-solo run

ログレベルをINFOにしてChefを実行してみます。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample05' -l info
[2013-09-08T00:45:15+09:00] INFO: Forking chef instance to converge...
Starting Chef Client, version 11.6.0
[2013-09-08T00:45:15+09:00] INFO: *** Chef 11.6.0 ***
[2013-09-08T00:45:17+09:00] WARN: Run List override has been provided.
[2013-09-08T00:45:17+09:00] WARN: Original Run List: []
[2013-09-08T00:45:17+09:00] WARN: Overridden Run List: [recipe[httpsv::sample05]]
[2013-09-08T00:45:17+09:00] INFO: Run List is [recipe[httpsv::sample05]]
[2013-09-08T00:45:17+09:00] INFO: Run List expands to [httpsv::sample05]
[2013-09-08T00:45:17+09:00] INFO: Starting Chef Run for sawamba01.local
[2013-09-08T00:45:17+09:00] INFO: Running start handlers
[2013-09-08T00:45:17+09:00] INFO: Start handlers complete.
Compiling Cookbooks...
Converging 2 resources
Recipe: httpsv::sample05
  * httpsv_auth_basic[var/www/site1:hoge1] action create[2013-09-08T00:45:17+09:00] INFO: Processing httpsv_auth_basic[var/www/site1:hoge1] action create (httpsv::sample05 line 3)
[2013-09-08T00:45:17+09:00] WARN: ====== Create http_auth user hoge1
[2013-09-08T00:45:17+09:00] INFO: file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] backed up to /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/backup/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd.chef-20130908004517

    - ====== Create http_auth user hoge1

  * httpsv_auth_basic[var/www/site1:hoge2] action create[2013-09-08T00:45:17+09:00] INFO: Processing httpsv_auth_basic[var/www/site1:hoge2] action create (httpsv::sample05 line 10)
[2013-09-08T00:45:17+09:00] WARN: ====== Create http_auth user hoge2
[2013-09-08T00:45:17+09:00] INFO: file[/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd] backed up to /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/backup/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd.chef-20130908004517

    - ====== Create http_auth user hoge2

[2013-09-08T00:45:17+09:00] INFO: Chef Run complete in 0.273085 seconds
[2013-09-08T00:45:17+09:00] INFO: Running report handlers
[2013-09-08T00:45:17+09:00] INFO: Report handlers complete
Chef Client finished, 2 resources updated
```

ファイルバックアップの様子がログに出ていますね。

`backed up to /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/backup/Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd.chef-20130908004517`

バックアップの取得状況を調べてみましょう。

```shell:tree_of_the_backup
$ tree -a backup/`pwd`
backup//Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb
└── var
    └── www
        └── site1
            └── .htpasswd.chef-20130908004517

3 directories, 1 file
```

いつものChefが作るバックアップです、やった！

#### 世代数を増やしたい

さて、デフォルトのバックアップ世代数は5ですが、これは心持ち少ないかもしれません。
普通の`Chef::Resouce`で使う際はレシピで世代数を管理できますが、このように間借りする場合は場合どうしましょうか。

こうします、`Chef::Resource::File`のインスタンス変数に強引に突っ込んでしまえば良いのです。

```ruby:def_backup!-increase_generation
def backup!
  htpasswd_file = Chef::Resource::File.new(@new_resource.path)
  htpasswd_file.instance_variable_set(:@backup, 30)
  backup = Chef::Util::Backup.new(htpasswd_file)
  backup.backup!
end
```

これで世代数を30にすることができます。レシピ側で世代数を指定するやり方はもう分かりますね。


### そんなタイムスタンプで大丈夫だろうか

先程のバックアップファイルに何か違和感を覚えませんでしたか？
そう、リソース更新は2回あったのにバックアップが一つしかありませんね。なぜならタイムスタンプが秒単位だからです。

通常のChef実行では同一のファイルをごそごそいじることは無いため、秒単位でも十分なのですがこのLWRPでは少々困ります。
折角なのでミリ秒も付けてもらいましょう。

`#backup!`の内容はこのようになります。

```ruby:def_backup!-use-micro_sec
def backup!
  htpasswd_file = Chef::Resource::File.new(@new_resource.path)
  htpasswd_file.instance_variable_set(:@backup, 30)
  backup = Chef::Util::Backup.new(htpasswd_file)
  backup.send(:backup_filename)
  backup.instance_variable_set(:@backup_filename,backup.instance_variable_get(:@backup_filename).gsub(/[\d]+$/,Time.now.strftime("%Y%m%d%H%M%S.%6N")))
  backup.backup!
end
```

ここを解説しようとすると長くなりそうなので割愛します、興味があったら各自でお調べ下さい。

新しい`#backup!`ではファイル名がこうなります。

```shell:tree_of_the_backup-micro_sec
$ tree -a backup/`pwd`
backup//Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb
└── var
    └── www
        └── site1
            ├── .htpasswd.chef-20130908010118.944651
            └── .htpasswd.chef-20130908010118.947960

3 directories, 2 files
```

リソース操作の間隔は3msしかなかったのですね。
しかしタイムスタンプの制度を変更することができたので、Chefの処理速度が10,000倍になるまでは問題ない。

これで`:create`アクションは完成となります、`:delete`時にも`#backup!`が使いまわせていい感じですね。

## 最後のアクション :discard を実装する

`.htpasswd`上のユーザ操作ができるようになりました、もう一つ必要な操作として考えられるのはユーザDB本体の破棄です。

認証自体を解除する場合には、`.htpasswd`ファイル自体不要なものになりますが、既存ユーザを全てdeleteするレシピを書くのはイマイチ現実的ではありません。それに今のやり方ではファイル自体は中身が空っぽで残ってしまいます。

ということで、DB自体を削除するアクション`:discard`を実装してみたResouce/Providerがこちらです。

### `resources/auth_basic.rb`

```ruby:cookbooks/httpsv/resources/auth_basic.rb-action-discard
actions :create, :delete, :discard
default_action :create

attribute :user, :kind_of => String, :required => true
attribute :password, :kind_of => String
attribute :path, :kind_of => String, :required => true
attribute :filemode, :default => 0600

attr_accessor :crypted_passwd
attr_accessor :db_exists
```

`db_exists`属性が増えました。使っている`WEBrick::HTTPAuth::Htpasswd`ライブラリの仕様のため必要になりました。


### `providers/auth_basic.rb`

```ruby:cookbooks/httpsv/providers/auth_basic.rb-action-discard
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

action :discard do
  if @current_resource.db_exists
    converge_by("====== Discard http_auth userdb #{@new_resource.path}") do
      backup!
      ::File.unlink(@new_resource.path)
    end
  else
    Chef::Log.warn "====== http_auth user database #{@new_resource.path} was not found. Nothing to do. (up to date)"
  end
end

def update_user!
  backup!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.set_passwd nil, @new_resource.user, @new_resource.password
  htpasswd.flush
  fix_permission!
end

def delete_user!
  backup!
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  htpasswd.delete_passwd nil, @new_resource.user
  htpasswd.flush
  fix_permission!
end

def fix_permission!
  FileUtils.chmod(@new_resource.filemode.to_i, @new_resource.path)
end

def backup!
  htpasswd_file = Chef::Resource::File.new(@new_resource.path)
  htpasswd_file.instance_variable_set(:@backup, 30)
  backup = Chef::Util::Backup.new(htpasswd_file)
  backup.send(:backup_filename)
  backup.instance_variable_set(:@backup_filename,backup.instance_variable_get(:@backup_filename).gsub(/[\d]+$/,Time.now.strftime("%Y%m%d%H%M%S.%6N")))
  backup.backup!
end

def load_current_resource
  @current_resource = Chef::Resource::HttpsvAuthBasic.new(@new_resource.name)
  unless ::File.exists?(@new_resource.path)
    @current_resource.db_exists = false
    @current_resource.crypted_passwd = nil
    return
  end
  htpasswd = WEBrick::HTTPAuth::Htpasswd.new(@new_resource.path)
  @current_resource.crypted_passwd = htpasswd.get_passwd nil, @new_resource.user, true
  @current_resource.db_exists = true
end
```

`:discard`についてはunlinkしているだけです、`@current_resource`の収集処理に手を加えています。

`WEBrick::HTTPAuth::Htpasswd`がnewする度にファイルを実際に作るので、`load_current_resource`が無条件にnewするとファイルが毎度作成されていました。
それでは`:discard`が永遠に収束しない(毎回空ファイル作成＆unlinkのマッチポンプ)ので、ユーザDBの存在確認を入れてあげました。




#### chef-solo run

Why-Runから行ってみます。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample08' -W 
Starting Chef Client, version 11.6.0
[2013-09-08T12:08:41+09:00] WARN: Run List override has been provided.
[2013-09-08T12:08:41+09:00] WARN: Original Run List: []
[2013-09-08T12:08:41+09:00] WARN: Overridden Run List: [recipe[httpsv::sample08]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample08
  * httpsv_auth_basic[var/www/site1:delete_dummy] action discard
    - Would ====== Discard http_auth userdb /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd

Chef Client finished, 1 resources would have been updated
```

レシピはこうなります。

### `recipes/sample08.rb`

```ruby:cookbooks/httpsv/recipes/sample08.rb
base_dir = ENV['PWD']

httpsv_auth_basic 'var/www/site1' do
  action :discard
  user 'delete_dummy'
  path File.join(base_dir, self.name, '.htpasswd')
  name [self.name, self.user].join(':')
end
```


#### chef-solo run

続いて実行。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample08' 
Starting Chef Client, version 11.6.0
[2013-09-08T12:09:12+09:00] WARN: Run List override has been provided.
[2013-09-08T12:09:12+09:00] WARN: Original Run List: []
[2013-09-08T12:09:12+09:00] WARN: Overridden Run List: [recipe[httpsv::sample08]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample08
  * httpsv_auth_basic[var/www/site1:delete_dummy] action discard
    - ====== Discard http_auth userdb /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd

Chef Client finished, 1 resources updated
```

#### chef-solo run

もう一度実行。

```shell:run_chef-solo
$ chef-solo -c solo.rb -o 'httpsv::sample08' 
Starting Chef Client, version 11.6.0
[2013-09-08T12:09:27+09:00] WARN: Run List override has been provided.
[2013-09-08T12:09:27+09:00] WARN: Original Run List: []
[2013-09-08T12:09:27+09:00] WARN: Overridden Run List: [recipe[httpsv::sample08]]
Compiling Cookbooks...
Converging 1 resources
Recipe: httpsv::sample08
  * httpsv_auth_basic[var/www/site1:delete_dummy] action discard[2013-09-08T12:09:28+09:00] WARN: ====== http_auth user database /Users/sawanoboriyu/github/opsrockin/lwrp_http_userdb/var/www/site1/.htpasswd was not found. Nothing to do. (up to date)
 (up to date)
Chef Client finished, 0 resources updated
```

これで今回のLWRP作成はひと通り終了しました。

## おわりに

パートが3つにもなってしまいましたが、LWRPを理解しましたね。

### 何故LWRP?

いくら内部DSLといっても、レシピ内で様々な処理を行ってしまうとレシピとしての可読性は下がる一方になります。
レシピの保守性を下げないようには、結局のところChefのフレームワークらしさを活かすためにもLWRPとしてリソースを実装することが最善でしょう。

### 作ってみよう

ここまで読むことが出来たのなら、次は自分のLWRPを作ってみましょう。実際大した手間ではありません。
しかし事前に言っておくと、そこそこハマる事うけあいです。今回も`@new_resource.action`の型が省略時Symbol, 指定時はArray(中身Symbol)などという妙な仕様に気づいたりもしました。

### 作らなくてもいい

さんざん解説しておいてなんですが、LWRPにまでしなくても`execute & not_if`程度で十分なケース、そもそもChefでやらないほうが楽なケースもあります。知っておくのは良いことですがこだわらないこともまた大事ですよね。
ただ、危なっかしいexecuteやRuby処理が多すぎて読みにくいレシピがあったら、LWRPの実装を検討してみてください。

----
[実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.1 of 3](http://qiita.com/sawanoboly@github/items/9bdaebcfc98d73f84843)
[実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.2 of 3](http://qiita.com/sawanoboly@github/items/79c7cdf782d64980677d)
実践LWRP、HTTP認証用ファイル(htpasswd,htdigest)をChefのリソースとして管理する part.3 of 3
