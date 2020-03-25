
Test-Kitchenを手っ取り早くはじめるには、シェルスクリプトを使うのがいい :fork_and_knife:
シェルで構築シェルでテストする例を作ってみた。

- Test-Kitchen公式: [Welcome to Test Kitchen - KitchenCI](http://kitchen.ci/ "Welcome to Test Kitchen - KitchenCI")
    - Github: [test-kitchen/test-kitchen](https://github.com/test-kitchen/test-kitchen "test-kitchen/test-kitchen")
- この記事のコード： [higanworks/shell-test-kitchen](https://github.com/higanworks/shell-test-kitchen)


## Test-Kitchen環境の準備

手元に必要なものはこの通り。

- Ruby
    - Gem: Bundler
- Vagrant


`Gemfile`には2つのGemを書こう。

```ruby:Gemfile
source "https://rubygems.org"

gem 'test-kitchen'
gem 'kitchen-vagrant'
```

bundleでは(とりあえず)`--binstubs`で`./bin`に各種コマンドが入るようにしておくとこのチュートリアルでは楽。

```
$ bundle install --binstubs --path=vendor/bundle
Fetching gem metadata from https://rubygems.org/..........
Fetching version metadata from https://rubygems.org/..
Resolving dependencies...
Installing safe_yaml 1.0.4
Installing net-ssh 2.9.2
Using bundler 1.9.2
Installing mixlib-shellout 2.0.1
Installing net-scp 1.2.1
Installing thor 0.19.1
Installing test-kitchen 1.3.1
Installing kitchen-vagrant 0.16.0
Bundle complete! 2 Gemfile dependencies, 8 gems now installed.
Bundled gems are installed into ./vendor/bundle.
```

※ direnvで`layout ruby`をしていればコマンドは`bundle`だけでもOK。このへんは各自のスタイルで。

`./bin/kitchen`としてコマンドが叩けるようになっているはず。

```
$ ./bin/kitchen -v
Test Kitchen version 1.3.1
```

次は`kitchen init`でひな形を作成しよう。

```
 $ ./bin/kitchen init
      create  .kitchen.yml
      create  test/integration/default
```



## .kitchen.ymlとdiagnose

> とりあえず試したいならこの項は無視して、`VMにShellを流してみよう`まで飛んでOK。


初期の`.kitchen.yml`はこうなっている、Shell用に書き換えよう。

```
---
driver:
  name: vagrant

provisioner:
  name: chef_solo

platforms:
  - name: ubuntu-12.04
  - name: centos-6.4

suites:
  - name: default
    run_list:
    attributes:
```

書き換え対象は2箇所、次の通り。

```
---
driver:
  name: vagrant

provisioner:
  name: shell    ## Shellに変更

platforms:
  - name: ubuntu-14.04  ## ubuntu-14.04のみに変更

suites:
  - name: default
    run_list:
    attributes:
```

`kitchen list`ではテストに使うVMの一覧が表示される。次のように出力されないならもう一度`.kitchen.yml`を見直そう。


```
$ ./bin/kitchen list
Instance             Driver   Provisioner  Last Action
default-ubuntu-1404  Vagrant  Shell        <Not Created>
```

### Diagnose

Test-Kitchenは設定の記述場所によって色々な要素が上書きされたりマージされたりする(まるでChefのAttributesだ！ :knife: )。
幸い各種設定がどのように反映されているか確認するための`kitchen diagnose`コマンドが用意されているので、設定の重なり具合をいつでも確認できる。

さっきの`.kitchen.yml`でdiagnoseを表示してみよう。

```
$ ./bin/kitchen diagnose default-ubuntu-1404
---
timestamp: 2015-04-03 04:06:20 UTC
kitchen_version: 1.3.1
instances:
  default-ubuntu-1404:
    state_file: {}
    driver:
      box: opscode-ubuntu-14.04
      box_check_update: 
      box_url: https://opscode-vm-bento.s3.amazonaws.com/vagrant/virtualbox/opscode_ubuntu-14.04_chef-provisionerless.box
      box_version: 
      customize: {}
      gui: 
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      log_level: :info
      name: vagrant
      network: []
      port: 22
      pre_create_command: 
      provider: virtualbox
      provision: false
      ssh: {}
      sudo: true
      synced_folders: []
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
      vagrantfile_erb: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/vendor/bundle/ruby/2.2.0/gems/kitchen-vagrant-0.16.0/templates/Vagrantfile.erb"
      vagrantfiles: []
      vm_hostname: default-ubuntu-1404
    provisioner:
      data_path: 
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      log_level: :info
      name: shell
      root_path: "/tmp/kitchen"
      script: 
      sudo: true
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
    busser:
      busser_bin: "/tmp/busser/bin/busser"
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      root_path: "/tmp/busser"
      ruby_bindir: "/opt/chef/embedded/bin"
      sudo: true
      suite_name: default
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
      version: busser
```


とりあえず今は`provisioner:`に注目してみよう。
Test-Kitchenのコマンドや対象は正規表現で解釈されるので、`kitchen diagnose default-ubuntu-1404`は`kitchen diag ubu`なりと省略できる。

```
$ ./bin/kitchen diag ubu | grep 'provisioner:' -A 8
    provisioner:
      data_path: 
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      log_level: :info
      name: shell
      root_path: "/tmp/kitchen"
      script: 
      sudo: true
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
```

`script:`が実行対象っぽさを醸し出しているね。
ここには`bootstrap.sh`(プラットフォームによっては`bootstrap.ps1`)というファイルがあれば自動でそれが使われるようになっている。
もちろん`.kitchen.yml`で指定してもいい。


## VMにShellを流してみよう - kitchen converge

`.kitchen.yml`を次のように編集してある状態から。前項を飛ばした人は編集してね。


```.kitchen.yml(再掲)
---
driver:
  name: vagrant

provisioner:
  name: shell

platforms:
  - name: ubuntu-14.04

suites:
  - name: default
    run_list:
    attributes:
```

`bootstrap.sh`という名前でファイルを作成すると、VM内で実行する対象として勝手に選択される。

```
$ touch bootstrap.sh

$ ./bin/kitchen diag ubu | grep 'provisioner:' -A 8
    provisioner:
      data_path: 
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      log_level: :info
      name: shell
      root_path: "/tmp/kitchen"
>>> 検出された。
      script: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/bootstrap.sh"   
<<<
      sudo: true
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
```

`bootstrap.sh`に書いた内容は実行時にちょっとラップされる、次のように。


```
sh -c '

bootstrap.shに書いた内容がここに入る

'
```

`'`(シングルクォート)でくくられているけど、数さえ合っていればスクリプト内で`'`を使う分には問題ない。

### bootstrap.shに処理を書いて実行する

とりあえずNginxでもいれとくか。

```bootstrap.sh
sudo apt-get update
sudo apt-get -y install nginx
```


準備ができたら`kitchen converge ubu`で。

```
 $ ./bin/kitchen converge ubu
-----> Starting Kitchen (v1.3.1)
-----> Creating <default-ubuntu-1404>...
       Bringing machine 'default' up with 'virtualbox' provider...
       ==> default: Importing base box 'opscode-ubuntu-14.04'...

(略) apt-get updateが始まる
-----> Converging <default-ubuntu-1404>...
       Preparing files for transfer
       Preparing script
       Transferring files to <default-ubuntu-1404>
Ign http://us.archive.ubuntu.com trusty InRelease                           
Ign http://security.ubuntu.com trusty-security InRelease
Ign http://us.archive.ubuntu.com trusty-updates InRelease
Get:1 http://security.ubuntu.com trusty-security Release.gpg [933 B]
Ign http://us.archive.ubuntu.com trusty-backports InRelease                    
Get:2 http://us.archive.ubuntu.com trusty Release.gpg [933 B]                  
Get:3 http://security.ubuntu.com trusty-security Release [63.5 kB]             
Get:4 http://us.archive.ubuntu.com trusty-updates Release.gpg [933 B]          
Get:5 http://us.archive.ubuntu.com trusty-backports Release.gpg [933 B]        
Get:6 http://us.archive.ubuntu.com trusty Release [58.5 kB]                    
Get:7 http://security.ubuntu.com trusty-security/main Sources [76.1 kB]        
(略)

       Processing triggers for ureadahead (0.100.0-16) ...
       Setting up nginx-core (1.4.6-1ubuntu3.2) ...
       Setting up nginx (1.4.6-1ubuntu3.2) ...
       Processing triggers for libc-bin (2.19-0ubuntu6) ...
       Finished converging <default-ubuntu-1404> (1m19.28s).
-----> Kitchen is finished. (1m19.50s)

```

Nginx入ったね。


## batsでVMをテストしてみよう - kitchen verify

テスト用の環境はTest-Kitchen関連プロダクトの`busser`を使って用意される。

`kitchen diag`で軽く`busser`周りを確認してみよう。

```
$ ./bin/kitchen diag ubu | grep busser: -A 8
    busser:
      busser_bin: "/tmp/busser/bin/busser"
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      root_path: "/tmp/busser"
      ruby_bindir: "/opt/chef/embedded/bin"
      sudo: true
      suite_name: default
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
      version: busser
```

`ruby_bindir`があることから分かる通り、Test-Kitchenではテストの実行にはVMの中にRubyが必要なんだ。


### RubyとBusser

Ruby入れたくない？ そうだよね。

でもテスト用のVMにくらい入れたっていいじゃない。初期構築の`bootstrap.sh`にちょっと分岐を作らせてよ。

```bootstrap.sh
sudo apt-get update
sudo apt-get -y install nginx


## For busser
if [ "vagrant" = "$SUDO_USER" ];then
  sudo apt-get -y install ruby2.0
fi
```

`bootstrap.sh`を更新したら、もう一度`kitchen conv ubu`して、Rubyのインストールを済ませておきましょう。

Rubyのインストールに問題がなければ、`.kitchen.yml`にBusserの`ruby_bindir`を上書きするよう仕向けます。

```.kitchen.yml
---
driver:
  name: vagrant

provisioner:
  name: shell

platforms:
  - name: ubuntu-14.04
    busser:
      ruby_bindir: /usr/bin     ## さっきのRubyへのパス

suites:
  - name: default
    run_list:
    attributes:
```

`kitchen diag ubu`でもかくにん。

```
 $ ./bin/kitchen diag ubu | grep busser: -A 8
    busser:
      busser_bin: "/tmp/busser/bin/busser"
      kitchen_root: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen"
      root_path: "/tmp/busser"
      ruby_bindir: "/usr/bin"  ## .kitchen.ymlに書いた値で上書き
      sudo: true
      suite_name: default
      test_base_path: "/Users/sawanoboriyu/github/higanworks/shell-test-kitchen/test/integration"
      version: busser
```

これでテスト実施の環境は整いました。


### Batsのテストを書く

- Bats: Bash Automated Testing System
    - [sstephenson/bats](https://github.com/sstephenson/bats "sstephenson/bats")

今日テストで使う[Bats](https://github.com/sstephenson/bats "sstephenson/bats")は別にRubyが必要ってわけではなく、Shellで書かれたShellの実行結果をテストするためのライブラリだ。

`bats`ディレクトリを作成しよう。

```
$ mkdir test/integration/default/bats
```

ではBatsによるコマンド実行＆出力チェックのテストを`test/integration/default/bats/nginx.bats`に書いていこう。
`@test`の直後に何をテストするかの注釈、そしてブレスでテストを囲めばいい。


```test/integration/default/bats/nginx.bats
@test "it should exists nginx bin with supports TLS SNI" {
  run nginx -V
  [ "$status" -eq 0 ]   ## ExitStatusが0か
  [ "${lines[2]}" = "TLS SNI support enabled" ]  ## 出力の2行目をチェック
}

@test "it should exists nginx process" {
  run pgrep nginx
  [ "$status" -eq 0 ]
}
```

ファイルを保存したら`kitchen verify ubu`でBatsのテストが実行される。


```
$ ./bin/kitchen verify ubu
-----> Starting Kitchen (v1.3.1)
-----> Setting up <default-ubuntu-1404>...
Fetching: thor-0.19.0.gem (100%)
Fetching: busser-0.7.0.gem (100%)
       Successfully installed thor-0.19.0
       Successfully installed busser-0.7.0
       2 gems installed
-----> Setting up Busser
       Creating BUSSER_ROOT in /tmp/busser
       Creating busser binstub
       Plugin bats installed (version 0.3.0)
-----> Running postinstall for bats plugin
       Installed Bats to /tmp/busser/vendor/bats/bin/bats
       Finished setting up <default-ubuntu-1404> (1m1.08s).
-----> Verifying <default-ubuntu-1404>...
       Suite path directory /tmp/busser/suites does not exist, skipping.
       Uploading /tmp/busser/suites/bats/nginx.bats (mode=0644)



>>> ここからテスト

-----> Running bats test suite
 ✓ it should exists nginx bin with supports TLS SNI
 ✓ it should exists nginx process
       
       2 tests, 0 failures
       Finished verifying <default-ubuntu-1404> (0m1.09s).
```

された！

## おわりに

ここまで出来たら`kitchen test`を実行して、眺めるかコーヒーでも汲みに行こう :coffee: 

```
$ ./bin/kitchen test
-----> Starting Kitchen (v1.3.1)
-----> Cleaning up any prior instances of <default-ubuntu-1404>
-----> Destroying <default-ubuntu-1404>...
       ==> default: Forcing shutdown of VM...
       ==> default: Destroying VM and associated drives...
       Vagrant instance <default-ubuntu-1404> destroyed.
       Finished destroying <default-ubuntu-1404> (0m4.06s).
-----> Testing <default-ubuntu-1404>
-----> Creating <default-ubuntu-1404>...

...

```

