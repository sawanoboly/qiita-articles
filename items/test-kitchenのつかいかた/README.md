
> 更新：v1.2.2.devに対応。

[Chef Inc. (旧Opscode)のtest-kitchen](https://github.com/opscode/test-kitchen)について、テストのライフサイクルとサブコマンドの使い方を説明する。

1.2系について。


## Test-Kitchenヘルプ

まずはkitchenコマンドを叩くとヘルプが表示される。

```bash:kitchen
$ kitchen 
Commands:
  kitchen console                                 # Kitchen Console!
  kitchen converge [INSTANCE|REGEXP|all]          # Change instance state to converge. Use a provisioner to configure one or more instances
  kitchen create [INSTANCE|REGEXP|all]            # Change instance state to create. Start one or more instances
  kitchen destroy [INSTANCE|REGEXP|all]           # Change instance state to destroy. Delete all information for one or more instances
  kitchen diagnose [INSTANCE|REGEXP|all]          # Show computed diagnostic configuration
  kitchen driver                                  # Driver subcommands
  kitchen driver create [NAME]                    # Create a new Kitchen Driver gem project
  kitchen driver discover                         # Discover Test Kitchen drivers published on RubyGems
  kitchen driver help [COMMAND]                   # Describe subcommands or one specific subcommand
  kitchen exec INSTANCE|REGEXP -c REMOTE_COMMAND  # Execute command on one or more instance
  kitchen help [COMMAND]                          # Describe available commands or one specific command
  kitchen init                                    # Adds some configuration to your cookbook so Kitchen can rock
  kitchen list [INSTANCE|REGEXP|all]              # Lists one or more instances
  kitchen login INSTANCE|REGEXP                   # Log in to one instance
  kitchen setup [INSTANCE|REGEXP|all]             # Change instance state to setup. Prepare to run automated tests. Install busser and related gems on one or more instances
  kitchen test [INSTANCE|REGEXP|all]              # Test (destroy, create, converge, setup, verify and destroy) one or more instances
  kitchen verify [INSTANCE|REGEXP|all]            # Change instance state to verify. Run automated tests on one or more instances
  kitchen version                                 # Print Kitchen's version information
```

`kitchen help {subcomand}`で詳細ヘルプが表示される。

```bash:kitchen_help_init
$ kitchen help init
Usage:
  kitchen init

Options:
  -D, [--driver=one two three]                   # One or more Kitchen Driver gems to be installed or added to a Gemfile 
                                                 # Default: kitchen-vagrant
  -P, [--provisioner=PROVISIONER]                # The default Kitchen Provisioner to use 
                                                 # Default: chef_solo
      [--create-gemfile], [--no-create-gemfile]  # Whether or not to create a Gemfile if one does not exist. Default: false 

Description:
  Init will add Test Kitchen support to an existing project for convergence integration testing. A default .kitchen.yml file (which is intended to be customized) is created in the 
  project's root directory and one or more gems will be added to the project's Gemfile.
```

流れがわかっていれば後はこのヘルプを引けば良いので、都度参照する。

## 用語：Instance

テストに使うVMは`Instance`と呼ばれ、命名法則と数は下記の式で決まる。

`Instance` = `${suite}-${platform}`の組み合わせ全部。

`platform`に`[ubuntu,centos]`とあって、`suite`に`[web, db]`とあった場合は

- web-ubuntu
- web-centos
- db-ubuntu
- db-centos

と4つのインスタンス定義ができる。

## Test-kitchenのライフサイクル

### 初期化

|コマンド| 効能 |
|---|---|
|kitchen init| .kitchen.yml を作成する、まずはこれから。デフォルトのドライバはVagrant。 |
|kitchen driver discover |  Rubygemsからtest-kitchen用に公開されているドライバ一覧を表示する。使いたいものをGemでインストールする。<br /> 依存にtest-kitchenがあり、`kitchen-` ではじまるものを抽出しているようだ。|

### テスト

ここが大事。


|コマンド| 効能 |
|---|---|
|kitchen create| インスタンスを作成する |
|kitchen setup| chefとbusserをインストールしてChef-repoをアップロード、初回のconvergenceを行う。 |
|kitchen converge | chef-reposが再度アップロードされ、Chef-Solo(またはローカルモード)を実行する。<br /> 繰り返し実行可能。 |
|kitchen verify| testディレクトリがアップロードされ、busser経由でテストスイートをインストールし、テストを実行する。<br /> 繰り返し実行可能。 |
|kitchen destroy| インスタンスを破棄する。 |
|kitchen test| 上記テストのライフサイクルを一括で行う。 <br />実行時にインスタンスがすでに存在する場合は初手でも**Destroy**が実行される。 |

test一発でもよいが、個別に何をやっているか知っていると捗る。

## 単発のタスク

|コマンド| 効能 |
|---|---|
|kitchen exec|起動中のインスタンスに対して任意のコマンドを実行する。createの直後にごにょごにょとか、verifyの前にちょっと実行しておきたい事があったらこれで。 |


## 構成確認、デバッグ

|コマンド| 効能 |
|---|---|
|kitchen list|`.kitchen.yml`にある`Instance`リストとライフサイクル上の状態を表示する。<br /> `Instance`の状態は逐一`.kitchen/`以下にyamlで吐かれているのでそれをみてもよい。 |
|kitchen login|`Instance`にSSHでログインする。execで物足りない時に。|
|kitchen console| .kitchen.yml と設定を読み込んだRubyのプロンプトが立ち上がる。<br /> `@instances`や`@suites`で各種設定が妥当か確認できる。 |
|kitchen diagnose|`.kitchen.yml`ほかによって最終的に有効になっている各種設定をダンプする。consoleの簡易版。 |

あれば便利という程度。

## ドライバ開発

|コマンド| 効能 |
|---|---|
|kitchen driver| ドライバ作成用のGemプロジェクト雛形を作成する。 <br /> 最低限必要なメソッドが用意されるので、穴埋めをするだけでドライバが作成できる。 |

細かい作り方は既存のプラグインを参考にするしかなさそう。

## おすすめの使い方

### ローカルで

create, setupと一つづつ実行しても良いが、`kitchen test`には `--destroy=never`というオプションをつけて実行するとインスタンスが破棄されずに保持される。

まず`kitchen test --destroy=never`で一気に`verify`まで行った後、適当にレシピを修正しながら`converge, verify`を繰り返すといった使い方が良いだろう。

シメにはもう一度`kitchen test`(`--destroy`オプションなし)で、インスタンスのBootから一気通貫テストをする。


### CIと

CIから継続的にテストを行うならば、`--destroy=always`オプションが良い。
通常は`converge, verify`に失敗したらインスタンスは保持されて、デバッグのチャンスが与えられるところだが、CIから自動でテストする場合はインスタンスに残られても困る。

結果に関係なく、インスタンスを破棄しておくため`always`を使う。

