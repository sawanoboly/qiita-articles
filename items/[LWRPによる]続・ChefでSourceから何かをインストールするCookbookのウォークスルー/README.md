<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


前回 [ChefでSourceから何かをインストールするCookbookのウォークスルー](http://qiita.com/items/9c09afc49f5229f646ed) で、IRCサーバの`ngircd`をインストールするためにtarballを展開してmakeするサンプルを出しました。

折角なので同じことを **LWRP(Lightweight Resources and Providers)** で表すことでLWRPについて解説をしてみます。

これもGithubでCookbookを確認できます。タグは`v0.2.1`。 [higanworks-cookbooks/ngircd_smartos(v0.2.1)](https://github.com/higanworks-cookbooks/ngircd_smartos/tree/v0.2.1)


# LWRPの取り扱い

[前回](http://qiita.com/items/9c09afc49f5229f646ed)作成したレシピでは以下の事をやりました。

1. `tarball`をダウンロード
2. `coufigure && make && make install`

`LWRP`は関数みたいに捉えられがちなので、一連の`configure, make`を使いまわせるようにまとめておくところかな？ と思われていたりしますが、実はそうではありません。

そういうのは [Definitions](http://docs.opscode.com/essentials_cookbook_definitions.html) であったり、 [Libraries](http://docs.opscode.com/essentials_cookbook_libraries.html)の役回りですね、`LWRP`の考え方とはまた別の話です。

## Resoucesの定義

さて**Resouces**です。
Chefはレシピの`new_resource`と現状の`current_resouce`を比較して**Converge**するフレームワークなので、そこを中心に考えていきます。

前回の`ngircd`のレシピで大事な要素だったのはなんでしょうか。
リモートのリポジトリ？ コンフィグのオプション？ makeのやり方？ それらは実際のところResouceの定義とは関連が無くても困りません。

極端な話、`ngircd`のバイナリが目当ての場所にあればOKです。


## ngircdについてのAttributes

Resourceは決まりました、ではそれの持つ要素をつけて行きましょう。
目当てのバージョンやincludeしているライブラリを定義可能にして、適切なものにしたいですね。

とりあえずVersion情報を取ります。

```bash:Shell-Out
$ ngircd -V
ngIRCd 20.2-IRCPLUS+SSL+SYSLOG+ZLIB-x86_64/pc/solaris2.11
Copyright (c)2001-2013 Alexander Barton (<alex@barton.de>) and Contributors.
Homepage: <http://ngircd.barton.de/>

This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

<del>ひどいよ、Ops怒るよ。</del>

`20.2-IRCPLUS+SSL+SYSLOG+ZLIB`、このへんから`ngircd`バイナリの要素を定義できそうです。

### Versionを要素に

バイナリにattibute[version]を持たせるためにパースしてみます。

```ruby:Irb-Out
> `ngircd -V`.split[1].split('-')[0]
=> "20.2"
```

アーカイブのバージョンと一致していて良い感じです。

### includeしているライブラリを要素に

バイナリにattibute[option]を持たせるためにパースしてみます。

```ruby:Irb-Out
> `ngircd -V`.split[1].split('-')[1].split('+')
=> ["IRCPLUS", "SSL", "SYSLOG", "ZLIB"]
```

これはまあ、SSLあたりに気をつければいいかなと思いつつ、一応Arrayに。


## レシピに書く内容

`Recouce`と`attributs`を決めたので、レシピには次のように書けばあとはよろしくやってくれるようにします。

```ruby:recipes/binary.rb 
ngircd_smartos_binary 'ngircd' do
  action :create
  version '20.2'
  options ["SSL"]
end
```
`current_resouce`の`version`が違う、または`option`のアレイに目的のものが入っていなければ**Convergence**を起こしてもらいます。
もちろんバイナリ本体が無ければ**手段は特に問わないので**置くようにという事でもあります。


# LWRP版Cookbook[ngircd]をウォークスルー

比較しやすいよう前回のレシピをほぼ使いまわしつつ、`LWRP[NgircdSmartosBinary]`を作りました。
順に注釈を入れてみます。

## attributes

attributesは大きく変わってません。`version,options`の省略時に使うデフォルトの値を追加したこと、`arch_file`の指定で`version`を使いまわせるようにしたくらいです。

```ruby:attributes/default.rb
## base settings
default['ngircd']['prefix_dir'] = '/opt/local'
default['ngircd']['conf_dir'] = ::File.join(node['ngircd']['prefix_dir'], 'etc')
default['ngircd']['version'] = '20.2'
default['ngircd']['config_options'] = ["SSL"]


## repository
default['ngircd']['site_url'] = 'http://ngircd.barton.de/pub/ngircd/'
default['ngircd']['arch_file'] = "ngircd-#{node['ngircd']['version']}.tar.gz"

## for tempolary working
default['ngircd']['working_dir'] = ::File.join(Chef::Config[:file_cache_path], 'ngircd')
```

## Resources

さてresourceですが、シンプルです。
指定できる`Actions`、それと要素の型やデフォルト値、必須かどうかをここで定義します。

```ruby:resources/binary.rb
actions :create, :delete

default_action :create

attribute :version, :kind_of => String, :default => node['ngircd']['version']
attribute :options, :kind_of => Array, :default => node['ngircd']['config_options']
```

Cookbookの`resouces/`以下に定義したリソースは、Chef実行時には`NameSpace`が与えられます。大雑把に`cookbookの名前＋ファイル名`に変換されるので覚えておくとよいです。


## Providers

そして**ツケを全部払う場所**、`Providers`です。[Definitions](http://docs.opscode.com/essentials_cookbook_definitions.html) や [Libraries](http://docs.opscode.com/essentials_cookbook_libraries.html)によるリファクタリングの余地は十分にありますが前回レシピの使い回しということで。


まずはざっと眺めるだけで、だいたい何をしているかわかると思います。ちなみに`action :delete`の部分は作ってません、`unlink`で済むんですが。

```ruby:providers/binary.rb
action :create do
  directory node['ngircd']['working_dir'] do
    action :create
  end

  unless new_resource.version == @current_resource.version and new_resource.options - @current_resource.options == []
    ::Chef::Log.info "Install ngircd"
    node.set['ngircd']['arch_file'] = "ngircd-#{new_resource.version}.tar.gz"

    remote_file ::File.join(node['ngircd']['working_dir'], node['ngircd']['arch_file']) do
      action :create
      source node['ngircd']['site_url'] + node['ngircd']['arch_file']
    end

    bash 'make and install ngircd' do
      action :run
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

    ## clean up workdir
    directory ::File.join(node['ngircd']['working_dir'], ::File.basename(node['ngircd']['arch_file'], '.tar.gz')) do
      action :delete
      recursive true
    end

    new_resource.updated_by_last_action(true)
  end
end

def load_current_resource
  @current_resource = Chef::Resource::NgircdSmartosBinary.new(new_resource.name)
  if ::File.exist?(::File.join(node['ngircd']['prefix_dir'], 'sbin/ngircd'))
    @current_resource.version `ngircd -V`.split[1].split('-')[0]
    @current_resource.options `ngircd -V`.split[1].split("-")[1].split("+")
  else
    @current_resource.version '0.0'
    @current_resource.options []
  end
end
```

### action :create do

レシピにこの`Resouce`が定義されていた時に`:create`に対して何をするか、そのままですね。

###   unless new_resource.version == @current_resource (以下略

まさにここで  `new_resource`**と現状の**`current_resouce`**を比較**  が行われます。
違った場合、とにかく`new_resource`に合わせるように処理を書いていきます。

### def load_current_resource

ちょっと一番下に飛んで、`def load_current_resource`です。
`@current_resource`を読むとこのdefが呼ばれます、既存のバイナリに対して`attributes`を収集していますね。

### node.set['ngircd']['arch_file']

リモート取得対象のattributeを`set`で上書きします。`attributes`すべてをリビルドすることもできますが手っ取り早さで`set`を使いました。

### remote_file ::File.join(node['ngircd']['working_dir'], node['ngircd']['arch_file']) 以降

ここで[前回のレシピ](http://qiita.com/items/9c09afc49f5229f646ed)が役に立つ、というかほぼ使いまわしです。

ポイントとしては

- `bash 'make and install ngircd'` に実行条件がない、`action :run`にしている。
  - ここに入った時点で無条件で実行してOKですよね。
- `remote_file`がnotifyしない
  - 同上な感じです、順序も変えてます。
- ## clean up workdir
  - ビルド後は一応掃除しておこうかなと、追加してます。


### new_resource.updated_by_last_action(true)

レシピ内でnotifyのフラグを立てます。
`ngircd_smartos_binary`に`notification`をさせる場合に必要ですが、何かしらConversion実行がなされる時は無条件で立てておくのがよいでしょう。

## 実行してみる

前述のレシピの内容を実行しました。`chef-shell`でもOKです。

### バイナリがない状態で実施 or なにかオプションが違う、バージョンが違う場合

`action :create`の内容がすべて実行されました。

```bash:Shell-out
$ chef-solo -c solo.rb -o 'ngircd_smartos::binary' 
Starting Chef Client, version 11.4.4
[2013-05-01T23:18:01+09:00] WARN: Run List override has been provided.
[2013-05-01T23:18:01+09:00] WARN: Original Run List: []
[2013-05-01T23:18:01+09:00] WARN: Overridden Run List: [recipe[ngircd_smartos::binary]]
Compiling Cookbooks...
Converging 1 resources
Recipe: ngircd_smartos::binary
  * ngircd_smartos_binary[ngircd] action create

Recipe: <Dynamically Defined Resource>
  * directory[/var/tmp/chef/ngircd] action create (up to date)
  * remote_file[/var/tmp/chef/ngircd/ngircd-20.2.tar.gz] action create (up to date)
  * bash[make and install ngircd] action run
    - execute "bash" -ex "/tmp/chef-script20130501-84816-19exjd8"

  * directory[/var/tmp/chef/ngircd/ngircd-20.2] action delete
    - delete existing directory /var/tmp/chef/ngircd/ngircd-20.2

Chef Client finished, 3 resources updated
```


### もう一度実行してみる、`up to date`

`new_resouce == current_resource` と判断したのでとくに影響のある動作はしてませんね。

```bash:Shell-out
$ chef-solo -c solo.rb -o 'ngircd_smartos::binary' 
Starting Chef Client, version 11.4.4
[2013-05-02T00:53:41+09:00] WARN: Run List override has been provided.
[2013-05-02T00:53:41+09:00] WARN: Original Run List: []
[2013-05-02T00:53:41+09:00] WARN: Overridden Run List: [recipe[ngircd_smartos::binary]]
Compiling Cookbooks...
Converging 1 resources
Recipe: ngircd_smartos::binary
  * ngircd_smartos_binary[ngircd] action create (up to date)
Recipe: <Dynamically Defined Resource>
  * directory[/var/tmp/chef/ngircd] action create (up to date)
Chef Client finished, 0 resources updated
```


## おわりに

`LWRP`の勘所、ひいては`Convergence`を実現する為のリソース比較について伝わったでしょうか。

OpsCode公式や`Community Cookbook`ではもっと沢山の - 例えば`mongodb`のShardingをLWRPで定義するなど - 色々と面白い例が見られます。
`chef_gem`リソースを使えば少々のgemならレシピ/Providers内で使えるので、定義と収束の基本を押さえておけばとても便利にLWRPを活用することができるかも？

