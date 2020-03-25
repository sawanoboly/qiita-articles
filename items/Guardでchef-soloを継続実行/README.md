<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[Guard](https://github.com/guard/guard)はファイルの変更をチェックして任意のタスクを実行するツール。

この手のものはテストを実行して開発・リファクタリングする用途に使うツールですが、`chef-solo`とレシピでやってもよいじゃないかと。

## 概要

さてこれから何をするか

- Guardに attributes/recipes以下の`default.rb`を監視してもらう。
- 対象のファイルに変更があったらchef-soloを実行する。
- ついでに`node.json`ファイルも監視。

このTipを手軽に再現するために[githubにコードを公開](https://github.com/higanworks/guard_chefsolo_example)してます。

## Guardのインストール

`bundler`を使おう、`guard-shell`はシェルスクリプトを実行する雛形を作るためのプラグインだ。

```ruby:Gemfile
Gemfile
source "https://rubygems.org"

gem 'chef'
gem 'guard'
gem 'guard-shell'
```

こんな感じでbundle.


## Guard-shellの初期化

`guard`のサブコマンドで初期化する、`shell`オプションをつかう。

```bash:shell-out
$ guard init shell
14:37:52 - INFO - Guard here! It looks like your project has a Gemfile, yet you are running
14:37:52 - INFO - Writing new Guardfile to /root/github/guard_chefsolo_example/Guardfile
14:37:52 - INFO - shell guard added to Guardfile, feel free to edit it
```

初期化で作成された`Guardfile`には監視対象と実行コマンドの例が書かれている。


```ruby:Guardfile(sample)
# A sample Guardfile
# More info at https://github.com/guard/guard#readme

# Add files and commands to this file, like the example:
#   watch(%r{file/path}) { `command(s)` }
#
guard 'shell' do
  watch(/(.*).txt/) {|m| `tail #{m[0]}` }
end
```

ぱっと見で`*.txt`ファイルが更新されたら`tail`を実行と分かるね。


## Chef-soloの準備

chef-solo実行のため必要なファイルを用意します。

### solo.rbとnode.json

```ruby:solo.rb 
cookbook_path [
  File.expand_path("../cookbooks", __FILE__)
]
```

```json:node.json
{}
```

jsonは`run_list`を並べたり`attributes`をoverrideするので空ですがひとまず作成。

### Guardfileへの反映

適当なCookBookとして`sandbox`というのを作ります、github参照。

```ruby:Guardfile 
guard 'shell' do
  watch('cookbooks/sandbox/attributes/default.rb') { `chef-solo -c solo.rb -j node.json` }
end

guard 'shell' do
  watch('cookbooks/sandbox/recipes/defaults.rb') { `chef-solo -c solo.rb -j node.json` }
end

guard 'shell' do
  watch('node.json') { `chef-solo -c solo.rb -j node.json` }
end
end
```

分かりやすさ重視で3つの単純なルールを作りました、watchのルールは正規表現などでもっと簡潔にかけます。

この時点のファイル状態を[1_start@github](https://github.com/higanworks/guard_chefsolo_example/tree/1_start)というブランチにしています。お試しはcheckoutでどうぞ。

## Guard起動と動作チェック

では`guard`コマンドを実行します、専用のプロンプトが起動するので別のShellセッションを開いてそちらで。


```shell:shell-out(guard)
$ guard 
15:08:01 - INFO - Guard here! It looks like your project has a Gemfile, yet you are running
15:08:01 - INFO - Guard uses NotifySend to send notifications.
15:08:01 - INFO - Guard uses TerminalTitle to send notifications.
15:08:01 - INFO - Guard is now watching at '/root/github/guard_chefsolo_example'
[Listen warning]:
  Listen will be polling for changes. Learn more at https://github.com/guard/listen#polling-fallback.

[1] guard(main)> 
```

ファイル監視の挙動は環境によります、ここでは`polling`が選択されました(joyent smartos)。

抜ける時は`exit`です。

```shell:shell-out(guard)
[1] guard(main)> exit
15:22:13 - INFO - Bye bye...
```


### ファイル変更の検知とコマンド実行

早速監視対象の`node.json`を更新します。

```shell:shell-out(main)
$ touch node.json
```

程なくタスクが実行されます。


```shell:shell-out(guard)
[2013-04-22T15:12:34+00:00] INFO: *** Chef 11.4.0 ***
[2013-04-22T15:12:36+00:00] INFO: Run List is []
[2013-04-22T15:12:36+00:00] INFO: Run List expands to []
[2013-04-22T15:12:36+00:00] INFO: Starting Chef Run for ipf_test
[2013-04-22T15:12:36+00:00] INFO: Running start handlers
[2013-04-22T15:12:36+00:00] INFO: Start handlers complete.
[2013-04-22T15:12:36+00:00] INFO: Chef Run complete in 0.005674519 seconds
[2013-04-22T15:12:36+00:00] INFO: Running report handlers
[2013-04-22T15:12:36+00:00] INFO: Report handlers complete
[1] guard(main)> 
```

`node.json`に`sandbox::default`の実行を記述してみます。

```json:node.json
{
  "run_list": [
    "sandbox::default"
  ]
}
```

エディタで開いて保存すると、またタスクが自動的に実行されます。


```shell:shell-out(guard)
[2013-04-22T15:19:33+00:00] INFO: *** Chef 11.4.0 ***
[2013-04-22T15:19:36+00:00] INFO: Setting the run_list to ["sandbox::default"] from JSON
[2013-04-22T15:19:36+00:00] INFO: Run List is [recipe[sandbox::default]]
[2013-04-22T15:19:36+00:00] INFO: Run List expands to [sandbox::default]
[2013-04-22T15:19:36+00:00] INFO: Starting Chef Run for ipf_test
[2013-04-22T15:19:36+00:00] INFO: Running start handlers
[2013-04-22T15:19:36+00:00] INFO: Start handlers complete.
[2013-04-22T15:19:36+00:00] INFO: Chef Run complete in 0.00620992 seconds
[2013-04-22T15:19:36+00:00] INFO: Running report handlers
[2013-04-22T15:19:36+00:00] INFO: Report handlers complete
[1] guard(main)> 
```


`run_list`に`["sandbox::default"]`が加わっています。

そしてこの時点のファイル状態を[2_run_guard@github](https://github.com/higanworks/guard_chefsolo_example/tree/2_run_guard)というブランチにしています。こちらもお試しはcheckoutでどうぞ。

## Guardに追われるようにレシピを作成する

ではguardを起動させた状態で**ファイル保存＝即コンバージョン**という適度な緊張感のなかレシピを書いていきましょう。  
この手法でのCookbook開発は**危ないので**クラウドIaaSの適当なインスタンスでやりましょう。


### fileを作成する

手軽な所で`File`リソースをレシピに書いてみます。

```ruby:cookbooks/sandbox/recipes/default.rb
file "/tmp/cha-ra.txt" do
  content "head-chara"
end
```

レシピを保存すると`Guard`によりchef-soloが実行されます。

```shell:shell-out(guard)
[2013-04-22T15:42:08+00:00] INFO: *** Chef 11.4.0 ***
-- snip --
[2013-04-22T15:42:10+00:00] INFO: Processing file[/tmp/cha-ra.txt] action create (sandbox::default line 11)
[2013-04-22T15:42:11+00:00] INFO: entered create
[2013-04-22T15:42:11+00:00] INFO: file[/tmp/cha-ra.txt] created file /tmp/cha-ra.txt
-- snip --
[1] guard(main)> 
```

ちょっとパーミッションを追加してみます。

```ruby:cookbooks/sandbox/recipes/default.rb
file "/tmp/cha-ra.txt" do
  content "head-chara"
  mode "0600"
end
```

早速コンバージョンされます。

```shell:shell-out(guard)
[2013-04-22T15:45:11+00:00] INFO: *** Chef 11.4.0 ***
-- snip --
[2013-04-22T15:45:14+00:00] INFO: Processing file[/tmp/cha-ra.txt] action create (sandbox::default line 11)
[2013-04-22T15:45:14+00:00] INFO: file[/tmp/cha-ra.txt] mode changed to 600
[1] guard(main)> 
-- snip --
```

### attributesを追加して、recpieに反映

ファイルのコンテンツを`attributes`で管理するため、`attributes/default.rb`を作ってみます。

```ruby:cookbooks/sandbox/attributes/default.rb
default['text_contents'] = 'heno-kappa'
```

`guard`が目ざとく`chef-solo`を走らせますが先ほどと変更がないため何も起こりません。

```shell:shell-out(guard)
[2013-04-22T15:50:40+00:00] INFO: *** Chef 11.4.0 ***
-- snip --
[2013-04-22T15:50:43+00:00] INFO: Processing file[/tmp/cha-ra.txt] action create (sandbox::default line 11)
-- snip --
[1] guard(main)> 
```

ではレシピ側で`attributes`を使うように変更を入れます。

```ruby:cookbooks/sandbox/recipes/default.rb
file "/tmp/cha-ra.txt" do
  content node['text_contents']
  mode "0600"
end
```

`chef-solo`が実行され、ファイルが更新されました。

```shell:shell-out(guard)
[2013-04-22T15:55:15+00:00] INFO: *** Chef 11.4.0 ***
-- snip --
[2013-04-22T15:55:17+00:00] INFO: Processing file[/tmp/cha-ra.txt] action create (sandbox::default line 11)
[2013-04-22T15:55:17+00:00] INFO: file[/tmp/cha-ra.txt] backed up to /var/chef/backup/tmp/cha-ra.txt.chef-20130422155517
[2013-04-22T15:55:17+00:00] INFO: file[/tmp/cha-ra.txt] contents updated
-- snip --
[1] guard(main)> 
```

最後の状態は[masterブランチ](https://github.com/higanworks/guard_chefsolo_example)です。


## おわりに

通常`guard`や`watchr`というツールは`RSpec`や`Chefspec`を実行するようにしてテストの手間を減らすものです。  

しかしサーバインスタンスの調達も簡単な今日ではOpsによるこのような強引なインフラ開発もいいんじゃないでしょうか。

ついでにこの手法は冪等を意識してレシピを書かないとサーバがどんどん壊れていくはずなのでChef勘が養える．．．かも。
