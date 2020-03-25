<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

> Note: serverspecのバージョンアップにより、多分この内容そのままでは失敗しそう。

構築済みサーバの確認には[serverspec][serverspec_url]が使いやすいようです。
そして私は受け入れテストの共有に[Cucumber][cucumber_url](Turnipも)を好んで使います。

さて`serverspec`の書式はもちろんRSpecで、英(単)語とプログラミングが嫌いなコミュニティでテストを共有しようとするのが結構気が引けたりします。`cucumber`のGherkinは癖がありますが、人に見てもらうのには適しています。

色々経緯もあって両方捨てがたいなーという状況だったので、`cucumber`から`serverspec`使ってみようということにしてみました。

[serverspec_url]:http://serverspec.org
[cucumber_url]:http://cukes.info/

## featureの作成

ということで、手始めとしてcentosにepelリポジトリが登録されている状態をテストしてみます。

featureのシナリオをつくります。

```cucumber:features/epel.feature
# coding:utf-8
# language: ja

機能: epelリポジトリ

前提: 何かしらの仕組みで構築済みです

シナリオ: epelのリポジトリが登録されている
  もし "/etc/yum.repos.d/epel.repo" ファイルが存在する
  ならば "epel" の状態が "enable" だ                                                                                                                        
```

cucumberを実行するとstep定義の案内が表示されます。

```shell:steps_information
You can implement step definitions for undefined steps with these snippets:

もし(/^"(.*?)" ファイルが存在する$/) do |arg1|
  pending # express the regexp above with the code you wish you had
end

ならば(/^"(.*?)" の状態がenableだ$/) do |arg1|
  pending # express the regexp above with the code you wish you had
end
```

この各Stepで`serverspec`を使ってみます。

## support/env の記述

stepで`serverspec`を使うために、cucumberの共通設定ファイル`features/support/env.rb`に各種記述をしていきます。
`serverspec-init`の生成する`spec_helper`を参考にしました。

```features/support/env.rb
require 'serverspec' 
include RSpec::Matchers
include Serverspec::Helper::Exec  # リモートが対象ならここはSsh
include Serverspec::Helper::DetectOS
```

迷ったら`serverspec-init`を実際に実行して、`spec_helper.rb`から記述をまるっと持ってきましょう。SSH経由のテストなら`RSpec.configure`周りが必須になります。

## Steps の記述

さてStepです、拡張されたMatchersはそのまま使えるわけでもなく、backendの選択が必要です。
ということでserverspecのリソースタイプをインスタンスにしてから、それに対してマッチャを使うやり方になります。

```ruby:features/step_definitions/epel_steps.rb
# coding: utf-8

もし(/^"(.*?)" ファイルが存在する$/) do |filename|
  @file = Serverspec::Type::File.new(filename)
  @file.should be_file
end

ならば(/^"(.*?)" の状態がenableだ$/) do |repo|
  @file.should contain('^\s*enabled=1').from(/\[#{repo}\]/).to(/^\n/)
end
```

serverspecの便利なマッチャ、`be_file`や`contain`(+from_to)をこの様に使うことができました。


## 実行してみる

それでは`cucumber`を実行してみましょう。

```shell:execute_cucumber
$ cucumber features/epel.feature 
# coding:utf-8
# language: ja
機能: epelリポジトリ
  
  前提: 何かしらの仕組みで構築済みです

  シナリオ: epelのリポジトリが登録されている                   # features/epel.feature:8
    もし"/etc/yum.repos.d/epel.repo" ファイルが存在する # features/step_definitions/epel_steps.rb:3
    ならば"epel" の状態がenableだ                    # features/step_definitions/epel_steps.rb:8

1 scenario (1 passed)
2 steps (2 passed)
0m0.111s

```

パスしました！ serverspecのマッチャが使えるのは嬉しいですね。


## もっとサンプルを、シナリオテンプレートと

とりあえず`cucumber`と`serverspec`との連携ができたので、より実践的なサンプルを作ってみました。

### feature samples

デーモンの状態と、パッケージの導入状況テストです。内容は見たらだいたい分かるかと。

```cucumber:features/service.feature
# coding:utf-8
# language: ja

機能: サーバが適当なデーモンを上げている

前提: 何かしらの仕組みで構築済みです

シナリオテンプレート: サービスチェック
  もし サービス "<servicename>" の状態を調べる
  ならば サービスの状態は "<state>" で
  かつ サービスの自動起動は "<on_boot>" です

  サンプル: 
    | servicename | state   | on_boot |
    | crond       | running | enable  |
    | rsyslog     | running | enable  |
    | mdmonitor   | stopped | disable |
```


```cucumber:features/package.feature
# coding:utf-8
# language: ja                 

機能: サーバが適当なパッケージを導入している

前提: 何かしらの仕組みで構築済みです
  
シナリオテンプレート: パッケージチェック
  もし パッケージ "<pkgname>" の状態を調べる 
  ならば パッケージの状態は "<state>" です

  サンプル:
    | pkgname   | state         |   
    | git       | installed     |   
    | syslog-ng | not_installed |   
    | tmux      | installed     |   
```


ちなみにテーブルのインデントを作るときは[Tabular.vim](http://vimcasts.org/episodes/aligning-text-with-tabular-vim/)というvimのプラグインを使って自動で整うようにしています。

### steps for features

先程のfeaturesに対するstepです。引数を工夫すればもっとメタに短く定義できると思いますがこんなもんで。

```ruby:features/step_definitions/service_steps.rb 
# coding: utf-8

もし(/^サービス "(.*?)" の状態を調べる$/) do |servicename|
  @service = Serverspec::Type::Service.new(servicename)
end

ならば(/^サービスの状態は "(.*?)" で$/) do |state|
  if state == 'running'
    @service.should be_running
  else
    @service.should_not be_running
  end
end

ならば(/^サービスの自動起動は "(.*?)" です$/) do |on_boot|
  if on_boot == 'enable'
    @service.should be_enabled
  else
    @service.should_not be_enabled
  end
end
```

```ruby:features/step_definitions/package_steps.rb 
# coding: utf-8

もし(/^パッケージ "(.*?)" の状態を調べる$/) do |pkgname|
  @package = Serverspec::Type::Package.new(pkgname)
end

ならば(/^パッケージの状態は "(.*?)" で$/) do |state|
  if state == 'installed'
    @package.should be_installed
  else
    @package.should_not be_installed
  end
end
```

stepではfailする場合に例外を出す必要があるのですが、serverspecのマッチャとshould(_not)の組み合わせでうまいこと例外になります。


### results

これらを実行すると下記の結果が得られます。

```shell:execute_cucumber
$ cucumber 
# coding:utf-8
# language: ja
機能: サーバが適当なパッケージを導入している
  
  前提: 何かしらの仕組みで構築済みです

  シナリオテンプレート: パッケージチェック         # features/package.feature:8
    もしパッケージ "<pkgname>" の状態を調べる # features/step_definitions/package_steps.rb:4
    ならばパッケージの状態は "<state>" で    # features/step_definitions/package_steps.rb:8

    サンプル: 
      | pkgname   | state         |
      | git       | installed     |
      | syslog-ng | not_installed |
      | tmux      | installed     |

# coding:utf-8
# language: ja
機能: サーバが適当なデーモンを上げている
  
  前提: 何かしらの仕組みで構築済みです

  シナリオテンプレート: サービスチェック             # features/service.feature:8
    もしサービス "<servicename>" の状態を調べる # features/step_definitions/service_steps.rb:3
    ならばサービスの状態は "<state>" で        # features/step_definitions/service_steps.rb:7
    かつサービスの自動起動は "<on_boot>" です    # features/step_definitions/service_steps.rb:15

    サンプル: 
      | servicename | state   | on_boot |
      | crond       | running | enable  |
      | rsyslog     | running | enable  |
      | mdmonitor   | stopped | disable |

6 scenarios (6 passed)
15 steps (15 passed)
0m0.287s
```



ついでにcucumberのHTMLフォーマットで出力してみました。

![Cucumber.png](https://qiita-image-store.s3.amazonaws.com/0/7454/c03a99af-50ea-e599-4982-b41ff0f29c2e.png "Cucumber.png")

見やすいので結果を共有しやすいかもしれない。


## おわりに

serverspec使いたいけど、RSpecの記法が導入のネックになっていたり、人におすすめしづらいというお悩みを持っておられたら、思い切ってGherkin(cucumber)という選択もどうでしょう。

何がおすすめか、というのは場合によりますわね。

----

追記：

SSH経由でリモートサーバをテストする際はこのように書いてもいけました。Featuresに織り交ぜることができますね。


```cucumber:step_with_ssh_example
Then(/^The server has cron entry '(.*)'/) do |line|
  @cron = Serverspec::Type::Cron.new
  Net::SSH.start('test.example.com', 'root') do |ssh|
    expect(@cron.have_entry(line).with_user('www')).to be_true
  end
end
```
