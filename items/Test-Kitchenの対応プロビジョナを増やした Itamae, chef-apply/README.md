
[Test-Kithen](http://kitchen.ci)は、

- どこかにVMをつくって
- 好きなプロビジョニングを適用して
- 適当なテストスイートを実行する

というツールです。

## あらすじ

1. 普段から[Test-Kithen](http://kitchen.ci)を色々な用途で使っているので、解説をまとめようと思った。
1. 拡張の仕方も説明できるかな ?
    - VMのドライバは作ったことがある： [kitchen-zcloudjp](https://github.com/higanworks/kitchen-zcloudjp)
    - テストのプラグインも作ったことがある: [busser-shindo](https://github.com/OpsRockin/busser-shindo)
    - プロビジョナは作ったことがない。
1. ついでだから何かプロビジョナの対応を作る。

『何かプロビジョナ』、すでにTest-Kitchen対応があると思っていたが無かった次の２つにした。

- [Itamae](http://itamae.kitchen/)
- [chef-apply](https://docs.chef.io/ctl_chef_apply.html)

## Test-Kitchenのプロビジョナづくり。

Test-Kitchenの拡張は、基本的に対象のBaseを継承してから任意のメソッドを上書きすることで行います。

プロビジョナ実行内のメソッドならこんな感じです。

|Provisioerのメソッド(一部抜粋)|軽い解説|
|----|----|
|install_command| VM内にプロビジョナをインストールするコマンドライン(`sh -c`で実行) |
|init_command| VM内のSandboxを初期化するコマンドライン(`sh -c`で実行)   |
|prepare_command| VM内でプロビジョナを実行する前に実行するコマンドライン(`sh -c`で実行) |
|run_command| VM内で実行するプロビジョナのコマンドライン(`sh -c`で実行)  |
|create_sandbox| ホストOSでレシピ等を一時的に配置(Sandbox)するRubyコード |
|cleanup_sandbox| ホストOSのSandboxを掃除するRubyコード |

これらを次のような感じで上書き。

1. VMにItamaeをインストール
    - Chef-ApplyはChefに便乗
1. `kitchen`以下のファイルをVMの`/tmp/kitchen`に転送
1. 事前準備があったらVM内で実行
1. ランリストを元にItamae/Chef-Applyを実行


## Itamaeでプロビジョニング

[kitchen-itamae](https://github.com/OpsRockin/kitchen-itamae)はRubygemsで公開しました。

実際に使って見る場合、[kitchen-itamaeのデモ](https://github.com/OpsRockin/kitchen-itamae_playground)として実装済の機能をほぼ試せるようにしてるので参考になるはずです。

Test-KithenでItamaeプロビジョナを使用するには、`kitchen/(任意)`以下にItamaeのRecipeを配置します。


```
kitchen/
├── nodes
│   ├── node1.json
│   └── node2.json
└── recipes
    ├── package1.rb
    └── package2.rb
```

で、`.kitchen.yml`の`provisioner`として`itamae`を指定。

```yaml:.kitchen.yml(itamae)
---
driver:
  name: vagrant
  customize:
    cpus: 2

provisioner:
  name: itamae

platforms:
  - name: ubuntu-14.04
    provisioner:
      node_json: nodes/node1.json
  - name: centos-6.6
    provisioner:
      node_json: nodes/node2.json

suites:
  - name: default
    run_list:
      - recipes/package1.rb
      - recipes/package2.rb
```

Test-Kitchenは定義ファイルの`.kitchen.yml`内にある、`driver`, `provisioner`, `platforms`, `suites`の各所から同じキーをマージするので色々な所で個別の設定を生やしておけます。
`kitchen diagnose`コマンドでコンフィグの重なり具合をいつでも確認できるので、何か変更したら都度確認してみるといいでしょう。

ちなみに`run_list`、`attribute`だけは特別な名前で、アレイ同士が合成されたりと[ChefでのDeep Merge](http://docs.chef.io/attributes.html#about-deep-merge)を踏襲しています。

> 追記：
> `.kitchen.yml`に`attribute`を記述すると、node_jsonの中身とマージします。
> 同じキーは`attribute`のほうを優先し、ファイルの使い回しや、環境依存設定の上書きをしやすいように。


あとは`kitchen converge`(kitchen test)でVM作成とItamaeが実行されます。


### Itamaeプラグインを使う

これは[kitchen-itamaeのデモ](https://github.com/OpsRockin/kitchen-itamae_playground)にあるとおり、`kitchen/`に`Gemfile`を設置すればItamae実行前にBundlerでインストールするようにしています。

```
kitchen/
├── Gemfile
├── nodes
│   ├── node1.json
│   └── node2.json
├── recipes
│   ├── file.rb
│   ├── git.rb
│   ├── mail_alias.rb
│   ├── package1.rb
│   ├── package2.rb
│   └── selinux.rb
└── templates
    └── template.erb
```

`Gemfile`に使用するプラグインを書いたら、あとは`include_recipe`するなど好きなように。


## Chef-Applyでプロビジョニング

Test-Kitchenのプロビジョナ、よく見たら標準に`chef-apply`がありませんでした。
既にある`shell`プロビジョナでも普通にまかなえますけど、折角なのでこちらは単品公開ではなく本家にPRしています。

[Add Provisioner chef_apply by sawanoboly · Pull Request #623 · test-kitchen/test-kitchen](https://github.com/test-kitchen/test-kitchen/pull/623 "Add Provisioner chef_apply by sawanoboly · Pull Request #623 · test-kitchen/test-kitchen")


Recipeを置くディレクトリは`apply/(変更可)`です。

```
apply/
├── hoge.rb
├── recipe1.rb
└── recipe2.rb
```

あとは`run_list`にレシピの名前を書いたらおわり。

```yaml:.kitchen.yml(chef_apply)
---
driver:
  name: vagrant

provisioner:
  name: chef_apply

platforms:
  - name: ubuntu-12.04
  - name: centos-6.4

suites:
  - name: default
    run_list:
      - recipe1
      - recipe2
```

と、こんな感じの設定になりますね。
