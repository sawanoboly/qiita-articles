
packagecloud.ioとBundlerをつかって、任意バージョンのRubygemを依存ごと手軽にホスティングしてみましょう。

![packagecloud_io.png](https://qiita-image-store.s3.amazonaws.com/0/7454/5e954166-92c0-0c7c-6f1f-a181e6b208eb.png "packagecloud_io.png")


## リポジトリを作成する

まずサインアップしたら、次に適当な名前でリポジトリを作成します。

> フリープランだとパッケージ数の上限蛾は25です。


![packagecloud_io.png](https://qiita-image-store.s3.amazonaws.com/0/7454/35566055-bcd5-7bfb-560e-925cd893b23e.png "packagecloud_io.png")


![packagecloud_io.png](https://qiita-image-store.s3.amazonaws.com/0/7454/06d6e1fc-eb02-0ce0-2e25-20a6fa583d34.png "packagecloud_io.png")


できました、rpm,deb,Rubygemに対応しています。


## CLIを導入する

package_cloudのCLIはRubygemです。インストールしましょう。

```
$ gem install package_cloud

...

Successfully installed package_cloud-0.2.11
7 gems installed
```


`package_cloud repository list`を実行すると、初回のみ認証情報の入力を求められ、トークンを作成してファイルに保存します。楽ですね。

```
$ package_cloud repository list
No config file exists at /Users/user1/.packagecloud. Login to create one.
Email:
user1@example.com
Password:

Got your token. Writing a config file to /Users/user1/.packagecloud... success!

```

トークン作成後は、先程作成したリポジトリがリストで確認できます。

```
$ package_cloud repository list
Your repositories:

  sawanoboly/long_live_serverspec_v1 (public)
  last push: never | packages: no packages
```

これでCLIの準備はOKです。


## Bundlerで任意バージョンのRubyGemを依存を含めて収集する

ではアップロードするパッケージ(Gems)を収集しましょう。

`bundle init`してGemfileを作成します。
サンプルとしてServerspecを1.x系で固めることにします。

```Gemfile
# A sample Gemfile
source "https://rubygems.org"

gem "serverspec", '< 2.0'
```

`bundle package`コマンドで`./vendor/bundle`ディレクトリにGemsが保存されていきます。

```
$ bundle package
Fetching gem metadata from https://rubygems.org/.....
Installing diff-lcs 1.2.5
Installing net-ssh 2.9.1
Installing highline 1.6.21
Installing rspec-mocks 2.99.2
Using bundler 1.7.3
Installing specinfra 1.27.5
Installing rspec-core 2.99.2
Installing rspec-expectations 2.99.2
Installing rspec 2.99.0
Installing rspec-its 1.0.1
Installing serverspec 1.16.0
Your bundle is complete!
It was installed into ./vendor/bundle
Updating files in vendor/cache
  * diff-lcs-1.2.5.gem
  * highline-1.6.21.gem
  * net-ssh-2.9.1.gem
  * rspec-core-2.99.2.gem
  * rspec-expectations-2.99.2.gem
  * rspec-mocks-2.99.2.gem
  * rspec-2.99.0.gem
  * rspec-its-1.0.1.gem
  * specinfra-1.27.5.gem
  * serverspec-1.16.0.gem



$ ls vendor/cache/
diff-lcs-1.2.5.gem		net-ssh-2.9.1.gem		rspec-core-2.99.2.gem		rspec-its-1.0.1.gem		serverspec-1.16.0.gem
highline-1.6.21.gem		rspec-2.99.0.gem		rspec-expectations-2.99.2.gem	rspec-mocks-2.99.2.gem		specinfra-1.27.5.gem
```

これで収集OK。

## Gemsをpackagecloudにアップロードする

CLIの`push`サブコマンドで`./vendor/bundle`以下に保存されたGemsをアップロードすることができます。

```
package_cloud push user/repo[/distro/version] /path/to/packages
```

こんな感じですね。

```
$ package_cloud push sawanoboly/long_live_serverspec_v1 vendor/cache/*.gem
Are you sure you want to push 10 packages? (y/n): y
Looking for repository at sawanoboly/long_live_serverspec_v1... success!
Pushing vendor/cache/highline-1.6.21.gem... success!
Pushing vendor/cache/net-ssh-2.9.1.gem... success!
Pushing vendor/cache/rspec-2.99.0.gem... success!
Pushing vendor/cache/rspec-core-2.99.2.gem... success!
Pushing vendor/cache/rspec-expectations-2.99.2.gem... success!
Pushing vendor/cache/rspec-its-1.0.1.gem... success!
Pushing vendor/cache/rspec-mocks-2.99.2.gem... success!
Pushing vendor/cache/serverspec-1.16.0.gem... success!
Pushing vendor/cache/specinfra-1.27.5.gem... success!
Pushing vendor/cache/diff-lcs-1.2.5.gem... success!
```

WebのUIでも確認出来ました。

![packagecloud_io.png](https://qiita-image-store.s3.amazonaws.com/0/7454/8418a858-a09f-0601-af64-c4429cd392c3.png "packagecloud_io.png")


`installation`の項目を見れば各種導入方法が載っています。
何れのパッケージ形式も、公式のリポジトリ追加手順にのっとっているので使いやすいですね。


## インストールテスト

バージョン無指定の`Gemfile`を作成してみます。

```Gemfile 
# A sample Gemfile
source 'https://packagecloud.io/sawanoboly/long_live_serverspec_v1/'

gem 'serverspec'
```

`bundle`してみると`source`で指定した`packagecloud.io`にRubyGemsを取りに行っています。


```
$ bundle install --path vendor/bundle --binstubs
Fetching source index from https://packagecloud.io/sawanoboly/long_live_serverspec_v1/
Installing diff-lcs 1.2.5
Installing highline 1.6.21
Installing net-ssh 2.9.1
Installing specinfra 1.27.5
Using bundler 1.7.3
Installing rspec-core 2.99.2
Installing rspec-mocks 2.99.2
Installing rspec-expectations 2.99.2
Installing rspec 2.99.0
Installing rspec-its 1.0.1
Installing serverspec 1.16.0
Your bundle is complete!
It was installed into ./vendor/bundle
```

固定したバージョンのGemsが入りました。


今回サンプルに使用したリポジトリはこちらです。 https://packagecloud.io/sawanoboly/long_live_serverspec_v1
