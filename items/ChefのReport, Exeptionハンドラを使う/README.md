<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Chefのクライアント(ChefSolo/ChefClient)を実行して、締めの処理をなにかしら行いたい。

詳細はこちら。 [Chef Docs: About Exception and Report Handlers](http://docs.opscode.com/essentials_handlers.html)


## 2つのハンドラ

- exception handler: 例外発生がトリガ
- report handler: 正常終了がトリガ

Chef_runの実行結果によって、このどちらか一方のみトリガされます。
同じ物を指定するのも構いません。

## ハンドラ仕様

ハンドラについてはこんな感じです。

- `Chef::Handler`を継承したクラス
- `report`メソッドを持つ
- `run_status`を持っているので中身を適当に処理

トリガ原因が`exception`でも`report`でも、`report`メソッドが実行されます。

### サンプルハンドラ

ではシンプルな[chef-handler-updated-resources](https://github.com/jtimberman/chef-handler-updated-resources)というのを使ってみます。

クライアント実行後に、今回更新されたリソース一覧をINFOレベルでログに出力します。

```ruby:updated_resources.rb
#
# Copyright:: 2011, Joshua Timberman <chefcode@housepub.org>
#
# Licensed under the Apache License, Version 2.0 (the "License");

require 'chef/handler'

module SimpleReport
  class UpdatedResources < Chef::Handler

    def report
      Chef::Log.info "Resources updated this run:"
      run_status.updated_resources.each {|r| Chef::Log.info " #{r.to_s}"}
    end
  end
end
```

`def report`ありますね。


### gem & Chefのコンフィグに書く

さきほどの`chef-handler-updated-resources`はGemなのでRubyGemsから導入します。

`gem install chef-handler-updated-resources --no-ri --no-rdoc`


このハンドラを使うためには`solo.rb/client.rb`で指定すればOKです。

```solo.rb/client.rb
require 'chef/handler/updated_resources'

report_handlers << SimpleReport::UpdatedResources.new
exception_handlers << SimpleReport::UpdatedResources.new
```

#### 実行サンプル: report

ファイルを置く、nginxをenableにするレシピを書きました。

```ruby:hand/recipes/default.rb
file '/tmp/hogehoge' do
  content Time.now.to_s
end                                                                                                                                   

file '/tmp/mogemoge' do
  content Time.now.to_s
end

service 'nginx' do
  action :enable
end
```

**nginxが止まっている時**

```bash:Shell_Out
$ chef-solo -c solo.rb -o 'hand' -l info
Starting Chef Client, version 11.4.4
[2013-07-23T06:33:40+00:00] INFO: *** Chef 11.4.4 ***
-- snip --
[2013-07-23T06:33:43+00:00] INFO: Chef Run complete in 0.491601172 seconds
[2013-07-23T06:33:43+00:00] INFO: Running report handlers
[2013-07-23T06:33:43+00:00] INFO: Resources updated this run:
[2013-07-23T06:33:43+00:00] INFO:   file[/tmp/hogehoge]
[2013-07-23T06:33:43+00:00] INFO:   file[/tmp/mogemoge]
[2013-07-23T06:33:43+00:00] INFO:   service[nginx]
[2013-07-23T06:33:43+00:00] INFO: Report handlers complete
Chef Client finished, 3 resources updated
```

`Running report handlers`から数行、レポートが出力されています。
最近は標準のログもわかりやすくなりましたね。

**nginxが動いている時**

```bash:Shell_Out
$ chef-solo -c solo.rb -o 'hand' -l info
Starting Chef Client, version 11.4.4
[2013-07-23T06:33:40+00:00] INFO: *** Chef 11.4.4 ***
-- snip --
[2013-07-23T06:38:16+00:00] INFO: Chef Run complete in 0.391962671 seconds
[2013-07-23T06:38:16+00:00] INFO: Running report handlers
[2013-07-23T06:38:16+00:00] INFO: Resources updated this run:
[2013-07-23T06:38:16+00:00] INFO:   file[/tmp/hogehoge]
[2013-07-23T06:38:16+00:00] INFO:   file[/tmp/mogemoge]
[2013-07-23T06:38:16+00:00] INFO: Report handlers complete
Chef Client finished, 2 resources updated
```
ファイルたちは中身が`Time.now`なので更新されていますが、nginxについてはノータッチだった事がわかりますね。

#### 実行サンプル: exception

例外を起こすために、レシピをちょっといじりました。

```ruby:hand/recipes/default.rb
file '/tmpo/hogehoge' do   # 親ディレクトリが存在しない
  content Time.now.to_s
end                                                                                                                                   

file '/tmp/mogemoge' do
  content Time.now.to_s
end
```
```bash:Shell_Out
$ chef-solo -c solo.rb -o 'hand' -l info
Starting Chef Client, version 11.4.4
[2013-07-23T06:33:40+00:00] INFO: *** Chef 11.4.4 ***
-- snip --
[2013-07-23T06:46:32+00:00] ERROR: Running exception handlers
[2013-07-23T06:46:32+00:00] INFO: Resources updated this run:
[2013-07-23T06:46:32+00:00] ERROR: Exception handlers complete
Chef Client failed. 0 resources updated
[2013-07-23T06:46:32+00:00] FATAL: Stacktrace dumped to /var/chef/cache/chef-stacktrace.out
[2013-07-23T06:46:32+00:00] FATAL: Chef::Exceptions::EnclosingDirectoryDoesNotExist: file[/tmpo/hogehoge] (han::default line 10) had an error: Chef::Exceptions::EnclosingDirectoryDoesNotExist: Parent directory /tmpo does not exist.
```

少々見づらいですが `Running exception handlers` から`Exception handlers complete`までが例外ハンドラのお勤めです。
一つ目でコケたため、残りのリソースに手を出していないことがわかります、途中の場合はそれまでに更新があったリソースが列挙されました。

では一つ目の例外を無視するようレシピを変更します。`ignore_failure`は共通オプションなので、どのリソースタイプでも有効です。

```ruby:hand/recipes/default.rb
file '/tmpo/hogehoge' do   # 親ディレクトリが存在しない
  content Time.now.to_s
  ignore_failure true      # 共通オプションの ignore_failure でエラーを無視してみる
end                                                                                                                                   

file '/tmp/mogemoge' do
  content Time.now.to_s
end
```

この場合failしたリソースは無視され、次以降のリソースは更新されます。

```bash:Shell_Out
$ chef-solo -c solo.rb -o 'hand' -l info
Starting Chef Client, version 11.4.4
[2013-07-23T06:33:40+00:00] INFO: *** Chef 11.4.4 ***
-- snip --
[2013-07-23T06:48:48+00:00] INFO: Chef Run complete in 0.222575733 seconds
[2013-07-23T06:48:48+00:00] INFO: Running report handlers
[2013-07-23T06:48:48+00:00] INFO: Resources updated this run:
[2013-07-23T06:48:48+00:00] INFO:   file[/tmp/mogemoge]
[2013-07-23T06:48:48+00:00] INFO: Report handlers complete
Chef Client finished, 1 resources updated
```
`file[/tmp/mogemoge]`のみ更新されたようですね、ハンドラもreportの方が使われています。
この場合途中で例外があっても最終的な`run_status`のexcptionは`null` になります、使い所の見極めが重要ですね。

一応`ignore_failure`としている場合も標準出力のログにはレベル=ERRORで出てきます。

```bash:STDOUT(LEVEL=>ERROR)
-- snip --
  * file[/tmpo/hogehoge] action create[2013-07-23T06:51:04+00:00] INFO: Processing file[/tmpo/hogehoge] action create (hand::default li
ne 10)

    * Parent directory /tmpo does not exist.[2013-07-23T06:51:04+00:00] ERROR: file[/tmpo/hogehoge] (hand::default line 10) had an erro
r: Parent directory /tmpo does not exist.; ignore_failure is set, continuing
-- snip --
```

ignoreしているのでそもそもではありますが、レベル=ERRORのログを見ておけば一応捕捉はできまるようです。


### LWRP

ハンドラ自体がLWRPになっているCookbookをChefRepoに追加するタイプです。
レシピにハンドラを記述できます。

詳しくはこちら。[https://github.com/opscode-cookbooks/chef_handler](https://github.com/opscode-cookbooks/chef_handler)

`compile phase`にはなにもしないけどchef_run時にのみ実行するという怪しげなやり方も案内されています。

## 用途

`report handler`は日常的にログをためておきたい時に、mongodbやfluentdに投げたり、メトリクスとして実行時間の推移を集計するような使い方ですね。
標準添付の`json_file`もノード状態、リソース状態をまるごと書きだしてくれるのでじっくりデバッグしたいときなどによさそうです(`json_file`のレポートは1回分で60kを超えるほどの量があるので放置に注意。)。

`exception handler`は公式のサンプルとプラグインにあるように例外をemailやIRCでお知らせしたり、自力で復旧を試みさせたりインスタンスに切腹させるなど考えられますね。
