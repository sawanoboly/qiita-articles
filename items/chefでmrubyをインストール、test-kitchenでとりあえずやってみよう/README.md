<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


以前、[ChefをつかってmrubyをCookbookから手軽に入れよう | Opsrock](http://opsrock.in/2013/09/24/768 "ChefをつかってmrubyをCookbookから手軽に入れよう | Opsrock")という記事を公開しました。
mrubyのCookbookはOpscodeのコミュニティCookbookとしても公開されているので、どうぞご自由にご利用ください。

なのですが、試すサーバも無いよという方のために、今日はtest-kitchen(with vagrant-driver)でmrubyを触ってみる方法を紹介。

> 追記: mod_mrubyにも対応した


## test-kitchenの事前準備

事前にこれらを公式のパッケージで入れましょう、あとRubyね。

- virtualbox
- vagrant

入れましたね。

## test-kitchenのセットアップ

mruby Cookbookをリポジトリからクローンします。

```shell:bash-cli_git_clone
$ git clone https://github.com/higanworks-cookbooks/mruby.git
$ cd mruby
```


bundlerで tes-kitchenをインストールしましょう。

```shell:bash-cli_bundle
$ bundle
Using rake (10.1.0) 
Using archive-tar-minitar (0.5.2) 
-- snip --
Using test-kitchen (1.1.0) 
Using kitchen-vagrant (0.13.0) 
Using librarian (0.1.0) 
Using librarian-chef (0.0.1) 
Using bundler (1.3.5) 
Your bundle is complete!
Use `bundle show [gemname]` to see where a bundled gem is installed.


$ kitchen -v
Test Kitchen version 1.1.0
```

はい、できましたね。


## 作成できる環境 + mrubyのセットを確認する

`kitchen list`コマンドで、作れる環境をの一覧を表示してみましょう。

```shell:bash-cli_kitchen_list
$ kitchen list
Instance               Driver   Provisioner  Last Action
default-ubuntu-1204    Vagrant  ChefSolo     <Not Created>
default-centos-64      Vagrant  ChefSolo     <Not Created>
rbenv-ubuntu-1204      Vagrant  ChefSolo     <Not Created>
rbenv-centos-64        Vagrant  ChefSolo     <Not Created>
ngx-mruby-ubuntu-1204  Vagrant  ChefSolo     <Not Created>
ngx-mruby-centos-64    Vagrant  ChefSolo     <Not Created>
```

2系統のプラットフォームで3種類ずつの環境が作れます。


## 仮想マシンに普通のmubyを使える環境を構築

`default-*`がそれにあたります。UbuntuとCentOSとありますね。
`kitchen converge`コマンドで両方作ってみましょう。



```shell:bash-cli_converge_default
$  kitchen converge default
-----> Starting Kitchen (v1.1.0)
-----> Creating <default-ubuntu-1204>...
       Bringing machine 'default' up with 'virtualbox' provider...
       [default] Importing base box 'opscode-ubuntu-12.04'...
```

Ubuntu環境の作成が始まりました、しばらく待ちましょう。


```
...
Build summary:       
       
================================================       
      Config Name: host       
 Output Directory: build/host       
         Binaries: mruby, mrbc, mirb       
    Included Gems:       
             mruby-sprintf - 0.0.0       
             mruby-print - 0.0.0       
             mruby-math - 0.0.0       
             mruby-time - 0.0.0       
             mruby-struct - 0.0.0       
             mruby-enum-ext - 0.0.0       
             mruby-string-ext - 0.0.0
             mruby-numeric-ext - 0.0.0
             mruby-array-ext - 0.0.0
             mruby-hash-ext - 0.0.0
             mruby-range-ext - 0.0.0
             mruby-proc-ext - 0.0.0
             mruby-symbol-ext - 0.0.0
             mruby-random - 0.0.0
             mruby-object-ext - 0.0.0
             mruby-objectspace - 0.0.0
             mruby-fiber - 0.0.0
             mruby-toplevel-ext - 0.0.0
             mruby-bin-mirb - 0.0.0
               - Binaries: mirb
             mruby-bin-mruby - 0.0.0
               - Binaries: mruby
             mruby-io - 0.0.0
       ================================================
...
```

mrubyビルドぽいログが。

間髪入れずにCentOSも始まります。

```
-----> Creating <default-centos-64>...
       Bringing machine 'default' up with 'virtualbox' provider...
       [default] Importing base box 'opscode-centos-6.4'...
```

まだまだ放っておきます。

```
...
       Build summary:
       
       ================================================
             Config Name: host
        Output Directory: build/host
         Binaries: mruby, mrbc, mirb
           Included Gems:
...
```

ビルド来てますね。


#### mrubyを確認する

`kitchen login`コマンドで仮想マシンにログインして確認してみましょう、Ubuntuから。

```
$ kitchen login default-ubuntu-1204
Welcome to Ubuntu 12.04.2 LTS (GNU/Linux 3.5.0-23-generic x86_64)

## 以下仮想マシン内

vagrant@default-ubuntu-1204:~$ mruby --version
mruby - Embeddable Ruby  Copyright (c) 2010-2013 mruby developers
```

次はCentOSへ。

```
$ kitchen login default-centos-64
Last login: Fri Dec  6 05:14:40 2013 from 10.0.2.2

## 以下仮想マシン内

[vagrant@default-centos-64 ~]$ mruby --version
mruby - Embeddable Ruby  Copyright (c) 2010-2013 mruby developers
```

やったね。

ちなみにmrubyのバージョンはmasterのHEADです、自由に選べるようにもしているので興味があれば変更してみましょう。


### おかたづけ

mrubyを存分に触ったら、仮想マシンを消しましょう。
`kitchen destroy`コマンドです。


```shell:bash-cli_
$  kitchen destroy default
-----> Starting Kitchen (v1.1.0)
-----> Destroying <default-ubuntu-1204>...
       [default] Forcing shutdown of VM...
       [default] Destroying VM and associated drives...
       Vagrant instance <default-ubuntu-1204> destroyed.
       Finished destroying <default-ubuntu-1204> (0m2.49s).
-----> Destroying <default-centos-64>...
       [default] Forcing shutdown of VM...
       [default] Destroying VM and associated drives...
       Vagrant instance <default-centos-64> destroyed.
       Finished destroying <default-centos-64> (0m2.51s).
-----> Kitchen is finished. (0m5.30s)
```

消したね。

## ngx_mrubyを組み込んだnginx

`ngx-mruby-*`はmrubyをインストールするついでに、ngx_mrubyモジュールを組み込んだNginxをビルドする環境です。
ついでにやってみましょう。

ngx_mrubyについてはこちらをどうぞ。

[人間とウェブの未来 - ngx_mrubyの紹介 ならびに nginx+mruby+Redisによる動的なリバースプロキシの実装案](http://blog.matsumoto-r.jp/?p=3737 "人間とウェブの未来 - ngx_mrubyの紹介 ならびに nginx+mruby+Redisによる動的なリバースプロキシの実装案")

ちなみにこちらの環境はmrubyのバージョンが[9b2f4c4423ed11f12d6393ae1f0dd4fe3e51ffa0](https://github.com/mruby/mruby/tree/9b2f4c4423ed11f12d6393ae1f0dd4fe3e51ffa0)に固定となってます。指定は`.kitchen.yml`ファイルで。

<blockquote class="twitter-tweet" lang="ja"><p>mod_mrubyとngx_mruby、submoduleで保存しているmrubyのcommitバージョンじゃないと動かない（最新mrubyのバイトコードの持ち方が大きく変更されているため）ので注意して下さい。年末対応予定です。</p>&mdash; MATSUMOTO, Ryosuke (@matsumotory) <a href="https://twitter.com/matsumotory/statuses/408833398040305664">2013, 12月 6</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


さあ、なにはともあれ`kitchen converge`です。


```shell:bash-cli_converge_ngx_mruby
$ kitchen converge ngx-mruby
-----> Starting Kitchen (v1.1.0)
-----> Creating <ngx-mruby-ubuntu-1204>...
-- snip --
```

またしばらく待ちましょう。


```shell:bash-cli_ngx_mruby
$ kitchen login ngx-mruby-ubuntu-1204
Welcome to Ubuntu 12.04.2 LTS (GNU/Linux 3.5.0-23-generic x86_64)

vagrant@ngx-mruby-ubuntu-1204:~$ /opt/nginx-1.4.2/sbin/nginx -V
nginx version: nginx/1.4.2
built by gcc 4.6.3 (Ubuntu/Linaro 4.6.3-1ubuntu5) 
TLS SNI support enabled
configure arguments: --prefix=/opt/nginx-1.4.2 --conf-path=/etc/nginx/nginx.conf --sbin-path=/opt/nginx-1.4.2/sbin/nginx --with-debug --add-module=/opt/chef_mruby/ngx_mruby --add-module=/opt/chef_mruby/ngx_mruby/dependence/ngx_devel_kit --with-http_ssl_module --with-http_geoip_module --with-ld-opt='-Wl,-R,/usr/local/lib -L /usr/local/lib' --with-http_realip_module --with-http_stub_status_module --with-http_gzip_static_module
```

`--add-module=/opt/chef_mruby/ngx_mruby`がついてます、やったね。


## mod_mrubyをapache httpdに追加(12/13追記)

<blockquote class="twitter-tweet" lang="ja"><p>これ、mod_mrubyのイメージもあったら開発環境として個人的に最高だな…</p>&mdash; MATSUMOTO, Ryosuke (@matsumotory) <a href="https://twitter.com/matsumotory/statuses/409342266017210368">2013, 12月 7</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

つかまえたフィードバックから、レシピを追加することにしました。

```
$ kitchen list
Instance               Driver   Provisioner  Last Action
default-ubuntu-1204    Vagrant  ChefSolo     <Not Created>
default-centos-64      Vagrant  ChefSolo     <Not Created>
rbenv-ubuntu-1204      Vagrant  ChefSolo     <Not Created>
rbenv-centos-64        Vagrant  ChefSolo     <Not Created>
ngx-mruby-ubuntu-1204  Vagrant  ChefSolo     <Not Created>
ngx-mruby-centos-64    Vagrant  ChefSolo     <Not Created>
mod-mruby-ubuntu-1204  Vagrant  ChefSolo     Converged
mod-mruby-centos-64    Vagrant  ChefSolo     Converged
```

mod_mruby環境のお試しはこちら。

```
$ kitchen converge mod-mruby
```

VM内で確認してみますよ。

```
$ kitchen login mod-mruby-centos

$ sudo httpd -M
Loaded Modules:
 core_module (static)
 mpm_prefork_module (static)
 http_module (static)
 so_module (static)
 mruby_module (shared)
 alias_module (shared)
 auth_basic_module (shared)
 authn_file_module (shared)
 authz_default_module (shared)
 authz_groupfile_module (shared)
 authz_host_module (shared)
 authz_user_module (shared)
 autoindex_module (shared)
 deflate_module (shared)
 dir_module (shared)
 env_module (shared)
 log_config_module (shared)
 logio_module (shared)
 mime_module (shared)
 negotiation_module (shared)
 setenvif_module (shared)
 status_module (shared)
Syntax OK
```

適当にファイルを置いて動作チェック。

```
$ cat -n /var/www/html/hoge.rb 
     1	if server_name == "NGINX"
     2	  Server = Nginx
     3	elsif server_name == "Apache"
     4	  Server = Apache
     5	end
     6	
     7	Server::rputs "Hello #{Server::module_name}/#{Server::module_version} world!\n"
```

```
$ curl localhost/hoge.rb
Hello mod_mruby/0.9.4 world!
```

でましたねえ。

## おわりに

CookbookのREADMEに任意のmgemを組み込む方法が書いてありますので、それも是非やってみましょう。

##　追記:12/27

<blockquote class="twitter-tweet" lang="ja"><p><a href="https://twitter.com/sawanoboly">@sawanoboly</a> <a href="https://t.co/02uAltxjmJ">https://t.co/02uAltxjmJ</a> 最新のmruby対応しましたので、advent calendarの環境構築ツールも最初書いていたのに合わしていただけると嬉しいです！</p>&mdash; MATSUMOTO, Ryosuke (@matsumotory) <a href="https://twitter.com/matsumotory/statuses/416182604526653440">2013, 12月 26</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

mruby最新のビルドに対応してきたようです。
ただ、元から任意のリビジョンのmrubyをビルドできるようにしているから、レシピに変更するところはないのです。

どこを変えたらいいかな? この記事を読んだ人は是非やってみよう。
