Chef Infra Client(※)のバージョン15くらいから、ライセンス同意を促されてますよね。(※他との区別のためchef-clientについた名前)

```
+---------------------------------------------+
            Chef License Acceptance

...
```

**ライセンスApache2ちゃうの**、と思うでしょうがそれはソースコードのお話とのこと。
Chef Software社がビルドした実行ファイル(※)とそれを含むrpm,debの利用に対してはこちらのChef Licenseに同意してね、と。

> ※ 実行ファイルだけ別のRubyGemになってたりする => https://github.com/chef/chef/tree/master/chef-bin

##  CINC Project

で、あのライセンス同意をしなくてよいバージョンをコミュニティビルドという体で出しましょう、というCINC Projectが発足。

https://gitlab.com/cinc-project

で、ちょっと名前違いの中身おんなじというパッケージが配布されだしたようです。
よくわからない？ 私もですから気にしないでいこうな。

Chef Infra Clientの範囲で、CINCとの違いは、、

- 実行関係のファイル`chef-*`が`cinc-*`に
  -  でもsymlinkがあるよ
- Chef License Acceptance不要
- <del>デフォルトの`/etc/chef`が`/etc/cinc`に</del> => <del>`/etc/chef`に変わりました。</del> => やっぱり`/etc/cinc` だよ！(また変わるかもね。。)
  - (ただしohaiは多分ずっと`/etc/chef`のまま)

これだけ。この先変わってしまうこともあるかもですが。


で、コミュニティビルドパッケージの配布は今の所こちらから。

http://downloads.cinc.sh


> 追記: Projectのサイトが出来ていた。 [CINC](https://cinc-project.gitlab.io/) omnibus-installerもここにあるよ。

## Chef Infra Client => CINC Clientに差し替える

せっかくなのでchef-clientからcinc-clientに差し替えてみよう。違いはさっき挙げただけなので、まるまる差し替えてしまっても大丈夫です。

まずChef Softwareビルド版のchefパッケージを消しますね。(例はUbuntu16)

```shelll
$ sudo dpkg -r chef
(Reading database ... 72883 files and directories currently installed.)
Removing chef (15.3.14-1) ..
```

[Download • CINC](https://cinc-project.gitlab.io/download/) から、インストーラのシェルを実行すればよいです。

以下、互換性維持のためのいくつかのお世話がされています。

- `cinc-client`ほか`cinc-*` => `cher-client`ほか`cher-*`へのsymlinkはCINCパッケージインストール時に作られます
-- uninstall時に消さないけど
- <del>設定ディレクトリは `/etc/chef` を使う</del>
- => デフォルトの設定ディレクトリはやっぱり`/etc/cinc`に戻った。


差し替えの場合、`/etc/cinc`と`/etc/chef`はリンクしておくのが無難。 普段から`--config`オプション付きで実行してもよいです。

```
$ sudo ln -s /etc/chef /etc/cinc
```


見た目何も変わらないので、適当に入れ替えてしまっても構わない感じです。

----

> 追記: 以下は古い情報です。

差し替えの場合、`/etc/cinc`と`/etc/chef`はリンクしておくのが無難。

```
$ sudo ln -s /etc/chef /etc/cinc
```

http://downloads.cinc.sh から適当なdebを入れればok.

```shelll
$ wget http://downloads.cinc.sh/files/stable/cinc/ubuntu/16.04/cinc_15.3.14-1_amd64.deb
$ sudo dpkg -i cinc_15.3.14-1_amd64.deb

...
Thank you for installing cinc, the community build based on Chef!

$ cinc-client -v
Cinc Client: 15.3.14
```

ついでに主要な実行ファイルをSymlinkにしてしまえば、互換性の面で安心ですかね。

```shelll
$ for x in apply client shell solo ; do sudo ln -sfv /usr/bin/cinc-$x /usr/bin/chef-$x ; done
'/usr/bin/chef-apply' -> '/usr/bin/cinc-apply'
'/usr/bin/chef-client' -> '/usr/bin/cinc-client'
'/usr/bin/chef-shell' -> '/usr/bin/cinc-shell'
'/usr/bin/chef-solo' -> '/usr/bin/cinc-solo'

$ chef-client -v
Cinc Client: 15.3.14
```


Chef License Acceptanceをなかったことにして、CINC Clientを実行してみました。

```shell
$ sudo rm -rfv /etc/chef/accepted_licenses/
removed '/etc/chef/accepted_licenses/inspec'
removed '/etc/chef/accepted_licenses/chef_infra_client'
removed directory '/etc/chef/accepted_licenses/'


$ sudo chef-client -z -l fatal
Starting Cinc Client, version 15.3.14
resolving cookbooks for run list: []
Synchronizing Cookbooks:
Installing Cookbook Gems:
Compiling Cookbooks...
Converging 0 resources

Running handlers:
Running handlers complete
Cinc Client finished, 0/0 resources updated in 01 seconds
```

止まらずに走りますね。
