<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Rubyを入れてChefやCucumber、serverspecなどを使いたいインフラさん向けに、UbuntuとCentOSでRVMを使ったRubyのインストール方法を紹介。

標準のPKG操作で1.8系を入れたりするのはちょっとね。。。
ちなみにBundlerをきっちり使えるなら`rbenv & ruby-build`でもいいです。

## RVMのインストール

RVMのインストールはスクリプトが用意されています。

```
curl -L https://get.rvm.io | bash -s stable
```

### Ubuntu

```shell:rvm.io
Installing RVM to /usr/local/rvm/
    Creating group 'rvm'

# RVM:  Shell scripts enabling management of multiple ruby environments.
# RTFM: https://rvm.io/
# HELP: http://webchat.freenode.net/?channels=rvm (#rvm on irc.freenode.net)
# Cheatsheet: http://cheat.errtheblog.com/s/rvm/
# Screencast: http://screencasts.org/episodes/how-to-use-rvm

# In case of any issues read output of 'rvm requirements' and/or 'rvm notes'

Installation of RVM in /usr/local/rvm/ is almost complete:

  * First you need to add all users that will be using rvm to 'rvm' group,
    and logout - login again, anyone using rvm will be operating with `umask u=rwx,g=rwx,o=rx`.

  * To start using RVM you need to run `source /etc/profile.d/rvm.sh`
    in all your open shell windows, in rare cases you need to reopen all shell windows.

# root,
#
#   Thank you for using RVM!
#   I sincerely hope that RVM helps to make your life easier and
#   more enjoyable!!!
#
# ~Wayne
```

RVMがインストールされました。
インストールが完了したので指示通り`source /etc/profile.d/rvm.sh`を読み込みます。

### CentOS

```shell:rvm.io
Installing RVM to /usr/local/rvm/
    Creating group 'rvm'

# RVM:  Shell scripts enabling management of multiple ruby environments.
# RTFM: https://rvm.io/
# HELP: http://webchat.freenode.net/?channels=rvm (#rvm on irc.freenode.net)
# Cheatsheet: http://cheat.errtheblog.com/s/rvm/
# Screencast: http://screencasts.org/episodes/how-to-use-rvm

# In case of any issues read output of 'rvm requirements' and/or 'rvm notes'

Installation of RVM in /usr/local/rvm/ is almost complete:

  * First you need to add all users that will be using rvm to 'rvm' group,
    and logout - login again, anyone using rvm will be operating with `umask u=rwx,g=rwx,o=rx`.

  * To start using RVM you need to run `source /etc/profile.d/rvm.sh`
    in all your open shell windows, in rare cases you need to reopen all shell windows.

# root,
#
#   Thank you for using RVM!
#   I sincerely hope that RVM helps to make your life easier and
#   more enjoyable!!!
#
# ~Wayne
```

こちらも同様にRVMがインストールされました。
インストールが完了したので指示通り`source /etc/profile.d/rvm.sh`を読み込みます。


## `rvm requirement` Rubyインストール前にパッケージを整える

先に`apt-get udpate`や`yum update`でリモート情報は新しいものにしておきましょう。

`rvm requirement`コマンドでRubyを入れる前にインストールしておいたほうがよいパッケージ群を教えてくれます。

### Ubuntu 

```shell:rvm_requirement
# For ruby:
apt-get --no-install-recommends install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev libgdbm-dev ncurses-dev automake libtool bison subversion pkg-config libffi-dev
```

`apt-get`のコピペ用文字列がもらえるのでそのままやっときましょう。

### CnetOS


```shell:rvm_requirement
# For ruby:
yum install gcc-c++ patch readline readline-devel zlib zlib-devel libyaml-devel libffi-devel openssl-devel make bzip2 autoconf automake libtool bison iconv-devel
```

こちらは`yum`用のコピペ文字列がもらえます、やっときましょう。


## rubyインストール

### 1.9.3のインストール

ここから先は共通です。

`rvm install 1.9.3`

rvm側の準備したバイナリがある場合はダウンロード、そうでなければソースからコンパイルされます。
Rubyのstableバージョンならリリース後にしばらくするとバイナリが用意されています。


パッチバージョンを指定する場合は

`rvm install 1.9.3-p385`


#### gems

デフォルトで少しGemsが入っています。

```shell:gem_list
# gem list

*** LOCAL GEMS ***

bigdecimal (1.1.0)
bundler (1.2.3)
io-console (0.3)
json (1.5.4)
minitest (2.5.1)
rake (10.0.3, 0.9.2.2)
rdoc (3.9.5)
rubygems-bundler (1.1.0)
rvm (1.11.3.6)
```

### デフォルトRubyに指定

ログインして使うぶんにはもう十分ですが、他から呼んだりする用にデフォルトで呼ばれるRubyのバージョンを指定します。

`rvm use 1.9.3 --default`

指定したバージョンのバイナリが`/usr/local/rvm/bin/ruby`に設置されます。

以降はPATHに`/usr/local/rvm/bin`を加えればだいたい何も考えずにRubyが使用出来ます。


#### デフォルトにしない場合や、他のユーザからrvmを使いたい場合

ユーザが各自でRVMをインストールすることができます。
また、rootでインストールしたrvmを使いまわす場合にはグループ`rvm`に追加します。

## 更新など

### rvm自身の更新

`rvm get stable`
`rvm reload`


### rubyの更新

任意のバージョンをインストールして、デフォルトを差し替えます。

`rvm install 1.9.3`
`rvm use 1.9.3 --default`



## 終わりに

rvmは少々好き嫌いが別れるツールですが、使えるようになっておくとよいです。

