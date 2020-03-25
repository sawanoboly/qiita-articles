<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[Cucumber-chef](http://www.cucumber-chef.org/)はBDDでのインフラ構築を支援するためのツール。
**テスト実行用の仮想マシン**を用意してレシピを適用するヘルパが強力だ。

公式のチュートリアルをさっと流してみる。

## 準備

チュートリアル通りにやるにはこれらが必要。

* opscodeのアカウントとOrganaization、無料版で可
  * クライアントキー
  * valirdation key
  * テストサーバはOpscodeにNodeとして登録されるのでNode1枠
* awsのアカウント
  * アクセスキー (権限はEC2だけでいいはず)
  * シークレット
* rvm (必須ではない)


knife.rb を書けるなら`opscode`の部分は既存のchef-serverでも大丈夫だとおもう。

## Setup
gemsetを作って`bundle init`したら、Gemfileで開発版の`cucumber-chef`を指定しよう安定版はnodeのアトリビュートが古い。
`chef`もAPIに変更があるので`0.10.x`を使う。


```ruby:Gemfile
source "https://rubygems.org"

gem "cucumber-chef", :git => "git://github.com/Atalanta/cucumber-chef.git"
gem "chef", ["0.10.10"]

```

## まずは init
bundleが終わったら、
`cucumber-chef init`
とすればインタラクティブに必要情報を入力することができる、opscodeのアカウントとか、aws関係のキーだ。

入力できたら一応`.cucumber-chef/config.rb `を確認しよう。


## setup、検証サーバがAWS上に作られる

`cucumber-chef setup`は**テストするコンテナをホストするためのインスタンス**をEC2上に作成する。

* cucumber-chef用のイメージ、ubuntu12.04のマシンが作られる(いくらか変更可)
* lxcのホスト、リゾルバとゲートウェイを兼ねる
  * bridgeとiptablesを使い、コンテナ用内部ネットワークをマスカレードする
* テストするcookbookを置くchef-server
  * opscodeのaptリポジトリからセットアップして起動
* レシピ適用とテストは**このホスト上のコンテナ**で行われる


セットアップのログを見れば何をしているかわかる。

```
$ cucumber-chef setup                                                               [1443/1900]
cucumber-chef v2.0.3.pre

Provisioning cucumber-chef test lab platform.
Waiting for instance...done.
Waiting for 20 seconds...done.
Waiting for SSHD...done.
Bootstrapping AWS EC2 instance...done.
Waiting for Chef-Server...done.
Waiting for Chef-WebUI...done.
Downloading chef-server credentials...done.
Building 'cc-knife' configuration...done.
Uploading cucumber-chef cookbooks...\zlib(finalizer): the stream was freed prematurely.
done.
Uploading cucumber-chef test lab role...done.
Tagging cucumber-chef test lab node...done.
Setting up cucumber-chef test lab run list...done.
Performing chef-client run to setup and configure the cucumber-chef test lab...done.
Downloading container SSH credentials...done.
Rebooting test lab; please wait...done.
Waiting for SSHD...done.
Waiting for Chef-Server...done.
Waiting for Chef-WebUI...done.

Your cucumber-chef test lab has now been provisioned!

Be sure to log into the chef-server webui and change the default credentials.

  Chef-Server WebUI:
    http://xxx.xxx.xxx.xxx:4040/
  Username:
    admin
  Password:
    *******
```

ローカルでvagrantを使うという選択肢もある、I/Oのスペックに余裕があるなら。

    追記：VagrantのBootstrapはまだ実装されていなかった。

## cookbookとproject(テストのためのセット)作成

testbookというcookbookのレシピを、
`cc-knife cookbook create testbook`

usersという名のコンテナでテストする。
`cucumber-chef create users`


`cucumber-chef create`で作られるものはcucumberのテストが含まれ、プロジェクトと呼んでいる。



## cookbookをアップロード
testbookには何もしてないが、とりあえずlabにアップする。

`cc-knife cookbook upload testbook`

cc-knifeはknifeのラッパ、コンフィグに`.cucumber-chef/knife.rb`を使う。

chefserverのWEBUIからもtestbookがリストされていることを確認できる。

## テストの概要を修正
`features/users/users.feature`を修正、runlistを`testbook::default`にする。


```cucumber:features/users/users.feature
-- snip --
  Background:
    * I have a server called "users"
    * "users" is running "ubuntu" "lucid"
    * "users" has been provisioned
    * the "testbook::default" recipe has been added to the "users" run list
    * the chef-client has been run on "users"
    * I ssh to "users" with the following credentials:
      | username | keyfile |
      | root | ../.ssh/id_rsa |
-- snip --
```


## テスト実行
プロジェクト作成時に付いてくるサンプルのテストをそのまま流す。
ランリストの適用とCucumberのテストが実施される。

```
$ cucumber-chef test 
cucumber-chef v2.0.3.pre

Using features directory: /Users/sawanoboriyu/Dev/sandbox/chef-cucumber/features
Cucumber-Chef Test Runner Initalized!
Cleaning up any previous test runs...done.
Uploading files required for this test run...done.
Executing Cucumber-Chef Test Runner
@users
Feature: Perform test driven infrastructure with Cucumber-Chef
  In order to learn how to develop test driven infrastructure
  As an infrastructure developer
  I want to better understand how to use Cucumber-Chef

  Background:                                                            # ./users/users.feature:7
  >>> servers are being preserved
    * I have a server called "users"                                     # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/provision_steps.rb:22
    * "users" is running "ubuntu" "lucid"                                # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/provision_steps.rb:26
    * "users" has been provisioned                                       # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/provision_steps.rb:46
    * the "test4::default" recipe has been added to the "users" run list # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/provision_steps.rb:54
  >>> chef-client ran on node 'users'
    * the chef-client has been run on "users"                            # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/provision_steps.rb:62
    * I ssh to "users" with the following credentials:                   # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:57
      | username | keyfile        |
      | root     | ../.ssh/id_rsa |

  Scenario: Can connect to the provisioned server via SSH authentication # ./users/users.feature:17
    When I run "hostname"                                                # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see "users" in the output                              # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81

  Scenario: Default root shell is bash                                   # ./users/users.feature:21
  >>> servers are being preserved
  >>> chef-client ran on node 'users'
    When I run "echo $SHELL"                                             # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see "bash" in the output                               # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81

  Scenario: Default gateway and resolver are using Cucumber-Chef Test Lab # ./users/users.feature:25
  >>> servers are being preserved
  >>> chef-client ran on node 'users'
    When I run "route -n | grep 'UG'"                                     # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see "192.168.255.254" in the output                     # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
    When I run "cat /etc/resolv.conf"                                     # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see "192.168.255.254" in the output                     # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
    And I should see "8.8.8.8" in the output                              # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
    And I should see "8.8.4.4" in the output                              # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81


  Scenario: Primary interface is configured with my IP address and MAC address # ./users/users.feature:33
  >>> servers are being preserved
  >>> chef-client ran on node 'users'
    When I run "ifconfig eth0"                                                 # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see the "IP" of "users" in the output                        # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:89
    And I should see the "MAC" of "users" in the output                        # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:89

  Scenario: Local interface is not configured with my IP address or MAC address # ./users/users.feature:38
  >>> servers are being preserved
  >>> chef-client ran on node 'users'
    When I run "ifconfig lo"                                                    # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see "127.0.0.1" in the output                                 # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
    And I should not see the "IP" of "users" in the output                      # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:89
    And I should not see the "MAC" of "users" in the output                     # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:89

  Scenario: Chef-Client is running as a daemon                           # ./users/users.feature:44
  >>> servers are being preserved
  >>> chef-client ran on node 'users'
    When I run "ps aux | grep [c]hef-client"                             # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:77
    Then I should see "chef-client" in the output                        # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
      expected: /chef-client/
           got: nil (using =~) (RSpec::Expectations::ExpectationNotMetError)
      ./users/users.feature:46:in `Then I should see "chef-client" in the output'
    And I should see "-d" in the output                                  # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
    And I should see "-i 1800" in the output                             # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
    And I should see "-s 20" in the output                               # cucumber-chef-2.0.3.pre/lib/cucumber/chef/steps/ssh_steps.rb:81
      exit (SystemExit)
      /home/ubuntu/features/support/env.rb:50:in `exit'
      /home/ubuntu/features/support/env.rb:50:in `After'

Failing Scenarios:
cucumber ./users/users.feature:44 # Scenario: Chef-Client is running as a daemon

6 scenarios (1 failed, 5 passed)
58 steps (1 failed, 3 skipped, 54 passed)
0m34.626s
```

chef-clientが `-d -i 1800 -s 20`で上がってないのでそのステップがこけたようだが、lxc上でテストが走った。


## コンテナにssh

labからlxc-consoleとかでもよいが。

```
$ cucumber-chef ssh mytestserver
cucumber-chef v2.0.3.pre

Attempting SSH connection to cucumber-chef container 'mytestserver'...
      _____                           _                _____ _           __
     / ____|                         | |              / ____| |         / _|
    | |    _   _  ___ _   _ _ __ ___ | |__   ___ _ __| |    | |__   ___| |_
    | |   | | | |/ __| | | | '_ ` _ \| '_ \ / _ \ '__| |    | '_ \ / _ \  _|
    | |___| |_| | (__| |_| | | | | | | |_) |  __/ |  | |____| | | |  __/ |
     \_____\__,_|\___|\__,_|_| |_| |_|_.__/ \___|_|   \_____|_| |_|\___|_|


    Welcome to the Cucumber Chef Test Lab v2.0.3.pre

    You are now logged in to the LXC 'mytestserver'

```

`chef-client -d -i 1800 -s 20` して、もう一度テストをしてみようか。

