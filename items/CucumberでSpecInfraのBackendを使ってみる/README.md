<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

サンプルコードはこちら。

https://github.com/higanworks/cucumber_specinfra_example


specinfraの使い方ついてはこちらもどうぞ。[specinfraを使ってみよう - Qiita](http://qiita.com/sawanoboly/items/d1e7794739d9d31a5316 "specinfraを使ってみよう - Qiita")


## まずはEnv

spcinfraの`.backend`をステップでお手軽に使えるよう、モジュールにしてWorldで読み込んだ。


```ruby:features/support/env.rb
require 'specinfra'
require 'net/ssh'

module InfraHelper
  include SpecInfra::Helper::DetectOS
  include SpecInfra::Helper::Ssh

  def default_backend(host, user = 'root', ssh_opts = {})
    SpecInfra.configuration.ssh = Net::SSH.start(host, user, ssh_opts)
  end
end

World(InfraHelper)
```

Githubのサンプルコードでは、Givenで対象ホストを指定するためにつくった`#return_backend`が残ってます。



## そしてBefore

とりあえず対象が1台なのと、コードをそのまま公開するために、Beforeからバックエンド設定をしてみました。

```ruby:features/support/hooks.rb
Before do
  default_backend(ENV['CUCUMBER_REMOTE_HOST'])
end
```

複数ならBackgroundに並べるなり、対象を変更しながら叩くなり色々やり方があると思います。


## シナリオへ

### サンプルシナリオ1

ではサンプルその一、OSファミリーが`SmartOS`であることを確かめるシナリオを、ちょっと汎用的なStepで作ってみます。

```cucumber:features/_get_os_family.feature
Feature: Get OS Family

  Scenario: Success Login
    When I ask to backend with "check_os"
    Then I will found "SmartOS" from backend at "family"
```

#### ステップ1

ハッシュで返ってくるメソッドに対して、任意のキーの中身を確かめます。

```ruby:features/step_definitions/ask_with_step.rb
When(/^I ask to backend with "(.*)"$/) do |method|
  @result = backend.send(method.to_sym)
end

Then(/^I will found "(.*)" from backend at "(.*)"$/) do |exp, key|
  raise unless @result[key.to_sym] == exp
end
```

ネストしたキーや、型も合わせたい場合はそれ用に工夫したステップが必要ですね。


### サンプルシナリオ2

特定バージョンのパッケージがインストールされていることを確かめます。
こちらは`specinfra`の`check_installed`を決め打ちする代わりに、シナリオアウトラインでまとめて記述できるようにしてみました。

```cucumber:features/check_package.feature
Feature: Check package

  Scenario Outline: Detect package
    Given I check package "<package>" installed with "<version>"

    Scenarios: Installed
      | package  | version   |
      | openssl  | 1.0.1fnb1 |
      | nginx    | 1.5.7     |
      | gmake    | 4.0       |
      | gcc47    | 4.7.3nb1  |
      | autoconf | 2.69nb3   |
```


#### ステップ2

`specinfra`がブーリアンで返してくれるので、ステップ側は一行で済みます。

```ruby:features/step_definitions/check_by_step.rb
When(/^I check package "(.*)" installed with "(.*)"$/) do |pkg, version|
  raise unless backend.check_installed(pkg, version)
end
```

ステップが完結でかつ、対象が他のOSでも使いまわせるためとても助かります。


## 結果発表

環境変数`CUCUMBER_REMOTE_HOST`をセットして、いざcucumber。

```bash:cucumber-result
$ cucumber 
Feature: Get OS Family

  Scenario: Success Login                                # features/_get_os_family.feature:3
    When I ask to backend with "check_os"                # features/step_definitions/ask_with_step.rb:1
    Then I will found "SmartOS" from backend at "family" # features/step_definitions/ask_with_step.rb:5

Feature: Check package

  Scenario Outline: Detect package                               # features/check_package.feature:3
    Given I check package "<package>" installed with "<version>" # features/step_definitions/check_by_step.rb:1

    Scenarios: Installed
      | package  | version   |
      | openssl  | 1.0.1fnb1 |
      | nginx    | 1.5.7     |
      | gmake    | 4.0       |
      | gcc47    | 4.7.3nb1  |
      | autoconf | 2.69nb3   |

6 scenarios (6 passed)
7 steps (7 passed)
0m14.372s
```

ナイスグリーン。

![nice green.](https://qiita-image-store.s3.amazonaws.com/0/7454/33643567-be70-ce6f-6869-b849da72da93.png "nice green.")
