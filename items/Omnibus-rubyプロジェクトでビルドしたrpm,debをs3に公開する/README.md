<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

以前のエントリ、[Omnibus-rubyプロジェクトでツールの周辺依存をまるごとrpm,debに固める - Qiita [キータ]](http://qiita.com/sawanoboly/items/a2c258a235824b91b70f "Omnibus-rubyプロジェクトでツールの周辺依存をまるごとrpm,debに固める - Qiita [キータ]") の続編です。

omnibusには作成したパッケージをAWSのs3にアップ＆公開する機能があるのでついでに紹介しよう。

<blockquote class="twitter-tweet" lang="ja"><p><a href="https://twitter.com/sawanoboly">@sawanoboly</a> ominubusでツールを配布する記事を書いたらバズる。</p>&mdash; Yusuke Ando (@yando) <a href="https://twitter.com/yando/statuses/415345606870044672">2013, 12月 24</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


ominubusの使い方は書いていたけど、配布もOKだと書いてなかったね。

## ツール選定

今回はRubyと適当なRubyGemsを入れただけのものを単純にパッケージにしてみました。
gemのbuildすらしてないですが、たまにはいいかと。

で、RubyGemsのエコシステムから簡単にいけるもの、serverspecにしてみました。

Omnibusプロジェクトはこちら。

[https://github.com/OpsRockin/omnibus-serverspec](https://github.com/OpsRockin/omnibus-serverspec)

で、`vagrant up`して放っておいたら手元にパッケージがそろいました。

```
$ ls -1 pkg/
serverspec-0.14.1_01-1.el6.x86_64.rpm
serverspec-0.14.1_01-1.el6.x86_64.rpm.metadata.json
serverspec_0.14.1-01-1.ubuntu.10.04_amd64.deb
serverspec_0.14.1-01-1.ubuntu.10.04_amd64.deb.metadata.json
serverspec_0.14.1-01-1.ubuntu.12.04_amd64.deb
serverspec_0.14.1-01-1.ubuntu.12.04_amd64.deb.metadata.json
```

それぞれ20MBくらいあるので、Diskの空き容量には注意ってか。


## `omnibus release package`

前エントリではプロジェクト雛形作成に使用した`omnibus `CLIですが、`release`サブコマンドがついてます。
次のように使います。

```
omnibus release package PATH   [ --public] # Upload a single package to S3
```

使用するには少々設定が必要です。


## S3リリースのセットアップ

リリースしたいバケットを事前に作成したら、コンフィグファイルにIAMのキーを記述します。

```ruby:omnibus.rb.example 
release_s3_access_key “S3_ACCESS_KEY”
release_s3_secret_key “S3_SECRET_KEY”
release_s3_bucket ”S3_BUCKET_NAME“
# use_s3_caching true
# solaris_compiler "gcc"
# build_retries 3
```

`omnibus.rb`にコピーして直接入れておいてもよし、サンプルと同じ名前の環境変数にしてもよしです。


## リリースしてみる

`—public`オプションを付けておくと、s3のパーミッションにpublic_readを付けてくれます。


```shell:omnibus_release_package
 $ omnibus release package pkg/serverspec-0.14.1_01-1.el6.x86_64.rpm --public
Using Omnibus configuration file PATH_TO/omnibus.rb
Using Omnibus configuration file PATH_TO/omnibus.rb
Uploaded el/6.5/x86_64/serverspec-0.14.1_01-1.el6.x86_64.rpm.metadata.json
Uploaded el/6.5/x86_64/serverspec-0.14.1_01-1.el6.x86_64.rpm
```

その他のパッケージも順々にアップ。
無事、s3にて公開されました。

![S3_Management_Console.png](https://qiita-image-store.s3.amazonaws.com/0/7454/42c1348a-7e74-024b-bd22-e73e9742b0c4.png "S3_Management_Console.png")



## インストールしてみる

どこかのクラウドにインスタンスを上げて、それぞれ全く初期状態からおもむろにインストールしてみました。

### CentOS

```
   __        .                   .
 _|  |_      | .-. .  . .-. :--. |-
|_    _|     ;|   ||  |(.-' |  | |
  |__|   `--'  `-' `;-| `-' '  ' `-'
                   /  ;  VirtualMachine (centos 6.4-2.4.1)
                   `-'   http://wiki.joyent.com/jpc2/Centos
```

```
# wget https://s3.amazonaws.com/omnibus-serverspec/el/6.5/x86_64/serverspec-0.14.1_01-1.el6.x86_64.rpm
# rpm -ivh ./serverspec-0.14.1_01-1.el6.x86_64.rpm 
Preparing...                ########################################### [100%]
You're about to install serverspec!
   1:serverspec             ########################################### [100%]
Thank you for installing serverspec!


# serverspec-init 
Select OS type:

  1) UN*X
  2) Windows

Select number: 1

Select a backend type:

  1) SSH
  2) Exec (local)

Select number: 2

 + spec/
 + spec/localhost/
 + spec/localhost/httpd_spec.rb
 + spec/spec_helper.rb
 + Rakefile


# rspec 

— snip —

Finished in 0.10964 seconds
6 examples, 6 failures


```

### Ubuntu

```
   __        .                   .
 _|  |_      | .-. .  . .-. :--. |-
|_    _|     ;|   ||  |(.-' |  | |
  |__|   `--'  `-' `;-| `-' '  ' `-'
                   /  ;  Joyent VirtualMachine (ubuntu 12.04-2.4.1)
                   `-'   http://wiki.joyent.com/jpc2/Ubuntu
```

```
# wget https://s3.amazonaws.com/omnibus-serverspec/ubuntu/12.04/x86_64/serverspec_0.14.1-01-1.ubuntu.12.04_amd64.deb
# dpkg -i serverspec_0.14.1-01-1.ubuntu.12.04_amd64.deb 
Selecting previously unselected package serverspec.
(Reading database ... 48405 files and directories currently installed.)
Unpacking serverspec (from serverspec_0.14.1-01-1.ubuntu.12.04_amd64.deb) ...
You're about to install serverspec!
Setting up serverspec (0.14.1-01-1.ubuntu.12.04) ...
Thank you for installing serverspec!


# serverspec-init 
Select OS type:

  1) UN*X
  2) Windows

Select number: 1

Select a backend type:

  1) SSH
  2) Exec (local)

Select number: 2

 + spec/
 + spec/localhost/
 + spec/localhost/httpd_spec.rb
 + spec/spec_helper.rb
 + Rakefile


# rspec 

— snip --

Finished in 0.59345 seconds
6 examples, 6 failures

```


動いてんのかなー
よくわかんないかなー

## 成果物サンプル

今回作ったserverspecインストール用のパッケージはここに公開状態でおいてます。

https://s3.amazonaws.com/omnibus-serverspec/

たまに更新するかもしれない。
