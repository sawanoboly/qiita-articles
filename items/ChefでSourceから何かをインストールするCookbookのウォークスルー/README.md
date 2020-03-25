<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Chefのレシピでソースから何かをmakeしてinstallをするやり方は、個人によってまちまちかと思います。

私はこんな感じでやっています。

## 概要

Joyent SmartOSにirdサーバデーモンの`ngircd`をインストールして、サービスとして起動します。

このCookbookはGithubに公開しています。 [higanworks-cookbooks/ngircd_smartos(v0.1.1)](https://github.com/higanworks-cookbooks/ngircd_smartos/tree/v0.1.1)

追記：続編できました！ [[LWRPによる]続・ChefでSourceから何かをインストールするCookbookのウォークスルー](http://qiita.com/items/e4d840da4d91b0379c65)

## レシピのざっくり解説

1. ローカルに目的のファイルが無かったら取ってくる
2. ファイルを取ってきたら`make & install`、ローカルがあれば何もしない
3. サービス登録
4. サービススタート


### attributes

```ruby:attributes/default.rb
## base settings
default['ngircd']['conf_dir'] = '/opt/local/etc'
default['ngircd']['conf_motd'] = '/opt/local/etc'


## repository
default['ngircd']['site_url'] = 'http://ngircd.barton.de/pub/ngircd/'
default['ngircd']['arch_file'] = 'ngircd-20.2.tar.gz'

## for tempolary working
default['ngircd']['working_dir'] = ::File.join(Chef::Config[:file_cache_path], 'ngircd')
default['ngircd']['configure_flags'] = ' --prefix=/opt/local --with-openssl'
```


`attributes` には環境によって変更するかも、という要素を置いています。
ローカルのファイルチェックには、`file_cache_path`を使うようにしているので、 `Chef::Config[:file_cache_path]`はサーバインスタンスのリブートで**中身が消えないパスに**しておくとよいです。


### recipes

リソースごとに注釈をいれてみます。

```ruby:recipes/default.rb
directory node['ngircd']['working_dir'] do
  action :create
end

bash 'make and install ngircd' do
  action :nothing
  flags '-ex'
  cwd node['ngircd']['working_dir']
  code <<-EOH
tar xzf #{node['ngircd']['arch_file']}
cd #{::File.basename(node['ngircd']['arch_file'], '.tar.gz')}
./configure #{node['ngircd']['configure_flags']}
make -j2
make install
EOH
end

remote_file ::File.join(node['ngircd']['working_dir'], node['ngircd']['arch_file']) do
  action :create_if_missing
  source node['ngircd']['site_url'] + node['ngircd']['arch_file']
  notifies :run, 'bash[make and install ngircd]', :immediately
end

smf 'ngircd' do
  start_command '/opt/local/sbin/ngircd'
  start_timeout 120
  stop_command '/usr/bin/pkill ngircd'
  stop_timeout 120
end

service 'ngircd' do
  action :enable
end
```


#### directory node['ngircd']['working_dir']

`file_cache_path`の下に`ngircd`というディレクトリを作って、`ngircd`関係はそこにまとめるようにしています。数が増えてくるとよごれるので。

#### bash 'make and install ngircd'

毎回走らないように`action :nothing`です。
ヨソから`notifies`で呼ばれるために定義しています。

`flags`に`-e`を含めると`bash -e`となり、途中のコマンドが`exit_status ≠ 0`の場合止まってもらいます。

`code`は普通のスクリプトです。

#### remote_file ::File.join(node['ngircd']['working_dir'], node['ngircd']['arch_file'])

`action :create_if_missing`でローカルにファイルがあったら何もしないという選択を指定してます。
ただ`:create`の指定だと、`remote_file`は毎回取ってきてsha比較などをしますがそこまでのものでもないので。

このリソースにアップデートがあったら(※ファイルを実際に取ってきた)、一つ前の`bash[make and install ngircd]`に対し`run`アクションを通知します、`:immediately`としているのでファイル作成直後に実行されます。

`:immediately`を省略したら実行は後回しにされます、後に`service['ngircd']`リソースがあり、ngirc本体のインストールを完了させるため`bash`を即時実行しておく必要があります。

#### smf 'ngircd'

これは`smartos`用のサービス登録リソースです、`svcadm`で`ngircd`を操作出来るようにしています。

#### service 'ngircd'

サービスをスタートします、`bash`の実行が終わってなければ当然failします。


## おわりに

ということでソースから何かを入れるレシピ、私のやり方を紹介してみました。

もっと良い感じのやり方や、こんな時はどうやってるの？という事に興味があればコメント等にて分かる範囲で答えます。

追記：続編できました！ [[LWRPによる]続・ChefでSourceから何かをインストールするCookbookのウォークスルー](http://qiita.com/items/e4d840da4d91b0379c65)
