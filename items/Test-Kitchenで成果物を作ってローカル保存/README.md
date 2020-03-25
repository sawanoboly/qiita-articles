<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[Test Kitchen](http://kitchen.ci/)はChefのCookbookをテストする用途に使うのが一般的です。
が、実際やっていることを要素として抜き出すと次の通りです。

- テンポラリなインスタンスをつくる
- プロビジョナで何かする
- 好きなテストスイート(主にRubyの)でテストする

ということで、『プロビジョナで何かする』のフェーズでパッケージをビルドしてみましょう。

ちなみにChef社が依存ライブラリ込みパッケージのクロスビルド用に保守している[Omnibus](https://github.com/opscode/omnibus)も、(便宜上の)バージョン3から内部ではTest-Kitchenを使っています。


## サンプルの概要

> この記事で使われているコードのリポジトリはこちら: https://github.com/OpsRockin/kitchen-build-example

次表のとおり作ってみました。

|対象|値|余談|
|----|----|---|
|導入するプロダクト |jemalloc(3.6.0)|手軽だったのでチョイス。(※Ubuntuには`libjemalloc`がパッケージであるのは気にしないで。)|
|ビルドするパッケージ|debとrpm|Ubuntu14.04とCentOS6.5上で、checkinstallを用いて作成|
|プロビジョナ|Ansible |たまには私もAnsible。 (※Chef以外を使うことで、選べるプロビジョナを演出) |
|テストスイート|Serverspec |デファクトスタンダード |

[Ansible](http://www.ansible.com/home)でjemallocのパッケージをビルドして、ついでにインストールしてみた後に[Serverspec](http://serverspec.org/)でテストするというのを`Test-Kitchen`でやってみましょうかと。

## 解説

> この記事で使われているコードのリポジトリはこちら: https://github.com/OpsRockin/kitchen-build-example

まずはGemfileです。

```ruby:Gemfile
# A sample Gemfile
source "https://rubygems.org"

gem 'test-kitchen'
gem 'kitchen-ansible'
gem 'kitchen-vagrant'

group :integration do
  gem 'serverspec'
end
```

プロビジョナとしてAnsibleを使うため、[kitchen-ansible](https://github.com/neillturner/kitchen-ansible)が入っています。使い方は割愛、[kitchen-ansibleのGithubリポジトリ](https://github.com/neillturner/kitchen-ansible)や[入門Ansible](http://www.amazon.co.jp/dp/B00MALTGDY)あたりを見れば良いと思います。


`.kitchen.yml`はこんな感じです、リポジトリには無いけども、コメントを入れておきます。

```yaml:.kitchen.yml
---
driver:
  name: vagrant
  synced_folders:
    - ["pkg", "/opt/pkg"]  ## ローカルのMacなりと共有するディレクトリ

provisioner:
  name: ansible_playbook   ## kitchen-ansibleを使う設定
  hosts: all
  ansible_verbose: true
  extra_vars:
    build_name: jemalloc
    build_vers: 3.6.0  ## group_varsの内容をここでも上書きOK

platforms:
  - name: ubuntu-14.04
  - name: centos-6.5

suites:
  - name: build  ## パッケージをビルドするだけのSuite、デフォルトのsite.ymlが使われる。
  - name: test   ## パッケージをインストールしてみて、Serverspecを流すSuite
    provisioner:
      playbook: test.yml

```

この`.kitchen.yml`で、VagrantによってVirtualBoxにVMがが作られ、使いたいPlaybookが実行されるという塩梅です。


## 実行

`build`と名前をつけたSuiteを実行するため、`kitchen converge build`します。Ubuntu14.04とCentOS6.5でそれぞれ実行されます。


```
$ kitchen converge build
-----> Starting Kitchen (v1.2.1)
-----> Creating <build-ubuntu-1404>...

-- sinp --

PLAY [all] ********************************************************************        
       
GATHERING FACTS ***************************************************************        
ok: [localhost]       
       
TASK: [prepare | debug ] ******************************************************        
ok: [localhost] => {       
    "item": "",        
    "msg": "Hello Ubuntu"       
}       
       
TASK: [prepare | file path={{build_dir}} state=directory] *********************        
changed: [localhost] => {"changed": true, "gid": 0, "group": "root", "item": "", "mode": "0755", "owner": "root", "path": "/opt/jemalloc", "size": 4096, "state": "directory", "uid": 0}       
       
TASK: [prepare | apt name=] ***************************************************        
changed: [localhost] => (省略)       
       
TASK: [prepare | file path=/root/rpmbuild state=directory] ********************        
skipping: [localhost]       
       
TASK: [prepare | file path=/root/rpmbuild/SOURCES state=directory] ************        
skipping: [localhost]       
       
TASK: [prepare | yum name=] ***************************************************        
skipping: [localhost]       
       
       TASK: [jemalloc | get_url url={{source_url}} dest={{build_dir}}/{{source_file}}] *** 
changed: [localhost] => {"changed": true, "dest": "/opt/jemalloc/jemalloc-3.6.0.tar.bz2", "gid": 0, "group": "root", "item": "", "md5sum": "e76665b63a8fddf4c9f26d2fa67afdf2", "mode": "0644", "msg": "OK (338964 bytes)", "owner": "root", "sha256sum": "", "size": 338964, "src": "/tmp/tmp6Lqcyg", "state": "file", "uid": 0, "url": "http://www.canonware.com/download/jemalloc/jemalloc-3.6.0.tar.bz2"}       
       
TASK: [jemalloc | command tar xaf {{source_file}} chdir={{build_dir}} creates={{build_name}}-{{build_vers}}] ***        
changed: [localhost] => {"changed": true, "cmd": ["tar", "xaf", "jemalloc-3.6.0.tar.bz2"], "delta": "0:00:00.085939", "end": "2014-08-13 15:34:41.330784", "item": "", "rc": 0, "start": "2014-08-13 15:34:41.244845", "stderr": "", "stdout": ""}       
       
TASK: [jemalloc | command ./configure] ****************************************        
changed: [localhost] => {省略}
       
TASK: [jemalloc | command make] ***********************************************        
changed: [localhost] => {省略}       
       
TASK: [jemalloc | command checkinstall -y --nodoc --pakdir={{pkg_dir}}] *******        
changed: [localhost] => {省略}       
       
PLAY RECAP ********************************************************************        
localhost                  : ok=9    changed=7    unreachable=0    failed=0          
       
       Finished converging <build-ubuntu-1404> (2m57.43s).

## ここからCnetOS

-----> Creating <build-centos-65>...

-- snip --

       PLAY [all] ******************************************************************** 
       
       GATHERING FACTS *************************************************************** 
       ok: [localhost]
       
       TASK: [prepare | debug ] ****************************************************** 
       ok: [localhost] => {
           "msg": "Hello CentOS"
       }
       
       TASK: [prepare | file path={{build_dir}} state=directory] ********************* 
       changed: [localhost] => {"changed": true, "gid": 0, "group": "root", "mode": "0755", "owner": "root", "path": "/opt/jemalloc", "size": 4096, "state": "directory", "uid": 0}
       
       TASK: [prepare | apt name={{item}}] ******************************************* 
       skipping: [localhost]
       
       TASK: [prepare | file path=/root/rpmbuild state=directory] ******************** 
       changed: [localhost] => {"changed": true, "gid": 0, "group": "root", "mode": "0755", "owner": "root", "path": "/root/rpmbuild", "size": 4096, "state": "directory", "uid": 0}
       
       TASK: [prepare | file path=/root/rpmbuild/SOURCES state=directory] ************ 
       changed: [localhost] => {"changed": true, "gid": 0, "group": "root", "mode": "0755", "owner": "root", "path": "/root/rpmbuild/SOURCES", "size": 4096, "state": "directory", "uid": 0}
       
       TASK: [prepare | yum name={{item}}] ******************************************* 
       changed: [localhost] => (省略}
       
       TASK: [jemalloc | get_url url={{source_url}} dest={{build_dir}}/{{source_file}}] *** 
       changed: [localhost] => {"changed": true, "dest": "/opt/jemalloc/jemalloc-3.6.0.tar.bz2", "gid": 0, "group": "root", "md5sum": "e76665b63a8fddf4c9f26d2fa67afdf2", "mode": "0644", "msg": "OK (338964 bytes)", "owner": "root", "sha256sum": "", "size": 338964, "src": "/tmp/tmpcUAwXP", "state": "file", "uid": 0, "url": "http://www.canonware.com/download/jemalloc/jemalloc-3.6.0.tar.bz2"}
       
       TASK: [jemalloc | command tar xaf {{source_file}} chdir={{build_dir}} creates={{build_name}}-{{build_vers}}] *** 
       changed: [localhost] => {"changed": true, "cmd": ["tar", "xaf", "jemalloc-3.6.0.tar.bz2"], "delta": "0:00:00.097923", "end": "2014-08-13 15:38:54.325594", "rc": 0, "start": "2014-08-13 15:38:54.227671", "stderr": "", "stdout": ""}
       
       TASK: [jemalloc | command ./configure] **************************************** 
changed: [localhost] => {省略}
       
       TASK: [jemalloc | command make] *********************************************** 
changed: [localhost] => {省略}
       
       TASK: [jemalloc | command checkinstall -y --nodoc --pakdir={{pkg_dir}}] ******* 
       changed: [localhost] => {省略}
       
       PLAY RECAP ******************************************************************** 
       localhost                  : ok=11   changed=9    unreachable=0    failed=0   
       
       Finished converging <build-centos-65> (3m15.34s).
-----> Kitchen is finished. (7m43.64s)

```

`kitchen`コマンドを実行したホストの`pkg/`には`checkinstall`で作成したパッケージができています。


```
$ ls pkg/
jemalloc-3.6.0-1.x86_64.rpm	jemalloc_3.6.0-1_amd64.deb
```


### Serverspecでテスト

`Test-Kitchen`のVerifyフェーズで、`test/`以下の内容を実行することができます。ちなみにconverge(Provisioning)とtestと、作る順番はどちらが先でも構いません。


- 事前のタスクとして、作成済みのパッケージをインストール(rpm or deb)する。
    - これもPlaybookで。
- パッケージのインストール後に確かめておきたい内容をServerspecで。

スイート`test`で実行するPlaybookはこのように直接記述しました。

```yaml:test.yml
---
- hosts: all
  sudo: yes
  tasks:
    - shell: "ls -1 {{ pkg_dir }}/*.deb | sort -r | head -n 1"
      register: deb_file
      when: ansible_distribution == "Ubuntu"
    - command: dpkg -i {{ deb_file.stdout }}
      when: ansible_distribution == "Ubuntu"
    - shell: "ls -1 {{ pkg_dir }}/*.rpm | sort -r | head -n 1"
      register: rpm_file
      when: ansible_distribution == "CentOS"
    - yum: name={{ rpm_file.stdout }}
      when: ansible_distribution == "CentOS"
```


で、パッケージインストール後の[spec](https://twitter.com/myb1126/status/499551825121402881)を簡単に記述してみます。

```rspec:jemalloc_spec.rb
require 'spec_helper'

describe package('jemalloc') do
  it { should be_installed }
end

describe command('/usr/local/bin/jemalloc.sh env') do
  it { should return_stdout /^LD_PRELOAD=.+libjemalloc.+/ }
end
```

そうしたらあとは`kitchen verify`です。

```shell:
$ kitchen verify test
-----> Starting Kitchen (v1.2.1)
-----> Creating <test-ubuntu-1404>...

-- snip --

## Ubuntuで
-----> Running serverspec test suite       
/opt/chef/embedded/bin/ruby -I/tmp/busser/suites/serverspec -S /opt/chef/embedded/bin/rspec /tmp/busser/suites/serverspec/localhost/jemalloc_spec.rb --color --format documentation       
       
Package "jemalloc"       
  should be installed       
       
Command "/usr/local/bin/jemalloc.sh env"       
  should return stdout /^LD_PRELOAD=.+libjemalloc.+/       
       
Finished in 0.08954 seconds       
2 examples, 0 failures       
       Finished verifying <test-ubuntu-1404> (0m1.78s).


## CentOSで

-----> Creating <test-centos-65>...

-- snip --

-----> Running serverspec test suite
       /opt/chef/embedded/bin/ruby -I/tmp/busser/suites/serverspec -S /opt/chef/embedded/bin/rspec /tmp/busser/suites/serverspec/localhost/jemalloc_spec.rb --color --format documentation
       
       Package "jemalloc"
         should be installed
       
       Command "/usr/local/bin/jemalloc.sh env"
         should return stdout /^LD_PRELOAD=.+libjemalloc.+/
       
       Finished in 0.08964 seconds
       2 examples, 0 failures
       Finished verifying <test-centos-65> (0m1.75s).


-----> Kitchen is finished. (9m16.28s)

```

Ubuntu(deb)、CentOS(rpm)のどちらも、`2 examples, 0 failures`と望んだ結果になっていることが確認できました。
これでこのパッケージは期待通りの動きをしてくれそうな気がします。



## さいごに

このサンプルではcheckinstallを軽めに扱っているので、そのまま再利用というわけにもいかないでしょう、そのへんは各自の知見の活用しどころだと思います。

例えばこういう形のtest-KitchenをCIなどで自動実行するようにして、作ったパッケージをしれっと[packagecloud.io](https://packagecloud.io/)などでホストすれば、自前パッケージのデリバリも楽になるかもしれないですね。
