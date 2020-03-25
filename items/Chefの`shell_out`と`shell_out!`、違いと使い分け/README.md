<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


## `Chef::Mixin::ShellOut`

`Chef::Mixin::ShellOut`はプラットフォームにあったCLI(`Unix=>Shell`, `Windows=>cmd`)を呼び出して、コマンドを実行するライブラリです。
コマンドをフォークしたプロセスで実行し、ExitStatusと標準出力/標準エラー出力を使いやすい形にしてくれることが特徴で、モックなども書きやすいため、ChefのCookbook内では`system`などのrubyメソッドより`ShellOut`を使用することが推奨されています。


ちなみに継承元は[opscode/mixlib-shellout](https://github.com/opscode/mixlib-shellout "opscode/mixlib-shellout")です。これも単体で使用できますが、`Chef::Mixin::ShellOut`とは少し使い勝手が違います。


## `shell_out`

では早速つかってみましょう。


```Ruby:Pry_Console
> require 'chef'
=> true

> include Chef::Mixin::ShellOut
=> Object

> result = shell_out('pwd')
=> <Mixlib::ShellOut#23553880:
  command: 'pwd'
  process_status: #<Process::Status: pid 16170 exit 0>
  stdout: '/root'
  stderr: ''
  child_pid: 16170
  environment: {"LC_ALL"=>"C"}
  timeout: 600
  user:
  group:
  working_dir:
>
```

コマンドの実行結果は`Mixlib::ShellOut`のインスタンスとなり、結果にまつわるアレコレを取得することができます。

```Ruby:Pry_Console
> result.class
=> Mixlib::ShellOut

> result.stdout
=> "/root\n"

> result.exitstatus
=> 0

```

コマンドの実行結果ステータスが0以外でも大丈夫。

```Ruby:Pry_Console
> result = shell_out('ps | grep hogehgoe')
=> <Mixlib::ShellOut#27543860:
  command: 'ps | grep hogehgoe'
  process_status: #<Process::Status: pid 16313 exit 1>
  stdout: ''
  stderr: ''
  child_pid: 16313
  environment: {"LC_ALL"=>"C"}
  timeout: 600
  user:
  group:
  working_dir:
>

> result.exitstatus
=> 1
```



## `shell_out!`

`shell_out!`は`shell_out`と使い方は一緒ですが、exitstatusが

```Ruby:Pry_Console
> result = shell_out!('ps | grep hogehgoe')
Mixlib::ShellOut::ShellCommandFailed: Expected process to exit with [0], but received '1'
---- Begin output of ps | grep hogehgoe ----
STDOUT: 
STDERR: 
---- End output of ps | grep hogehgoe ----
Ran ps | grep hogehgoe returned 1
from /opt/chef/embedded/lib/ruby/gems/1.9.1/gems/mixlib-shellout-1.2.0/lib/mixlib/shellout.rb:251:in `invalid!'
```


Rubyではしばしば`!`が付くメソッドは自身を変更する破壊的メソッドをあわらす記号として使われます。
`shell_out!`もまあ、例外終了するという理由で破壊的な挙動と覚えておくといいかもしれません。


### 0以外の終了ステータスで`shell_out!`

ExitCodeが0以外であることが要求されるケースでは、`:returns`オプションで求める結果を指定することができます。


```
> result = shell_out!('ps | grep hogehgoe', :returns => 1)
=> <Mixlib::ShellOut#18867260:
  command: 'ps | grep hogehgoe'
  process_status: #<Process::Status: pid 16586 exit 1>
  stdout: ''
  stderr: ''
  child_pid: 16586
  environment: {"LC_ALL"=>"C"}
  timeout: 600
  user: 
  group: 
  working_dir: 
>

## ArrayでもOK
> result = shell_out!('ps | grep hogehgoe', :returns => [1,3,5,7,9])
```

その他使えるオプションはソースを見るとよいです。
[mixlib-shellout/lib/mixlib/shellout.rb at master · opscode/mixlib-shellout](https://github.com/opscode/mixlib-shellout/blob/master/lib/mixlib/shellout.rb#L268 "mixlib-shellout/lib/mixlib/shellout.rb at master · opscode/mixlib-shellout")

プライベートメソッドの`#parse_options`が使えるオプション一覧です。


## 使い分け方


### `shell_out`が必要なケース

実行結果のうち、ステータスや出力を使用して処理をしたい時にこちらを使います。LWRP中でのCurrentResource収集の一貫としてなどですね。
RecipeやLibrary内でも標準出力から文字列を組み立てたい時等に使うとよいでしょう。


### `shell_out!`が必要なケース

CookbookではLWRPが適当ですが、収束をかける処理の一環として、そのコマンドが期待した結果を得られなかったらリソースが収束しないというケースで使用します。
`shell_out`で、ExitStatusが0でなければfailするという分岐を作ることと同様ですが、よりシンプルに記述することができます。


## Command not Foundは流石にダメ

`shell_out`が実行結果の成否にかかわらないといえ、実行するコマンド自体へパスが通ってなかったり、そもそも存在しないという状況は例外で終了します。

```
> shell_out('hogehgoe')
Errno::ENOENT: No such file or directory - hogehgoe
from /opt/chef/embedded/lib/ruby/gems/1.9.1/gems/mixlib-shellout-1.2.0/lib/mixlib/shellout/unix.rb:268:in `exec'
```

これは素直に段取りをちゃんとしましょう。
