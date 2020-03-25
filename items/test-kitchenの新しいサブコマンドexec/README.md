
test-kitchenでも`vagrant exec`みたいなことしたいよとクライアントが言ったので、実装してみたら意外と便利だったので本家にプルリクして採用。
次回リリース(v1.2.2かな?)から使えると思います。

> 1.3.0に含まれました。

[Add new subcommand 'exec' by sawanoboly · Pull Request #373 · test-kitchen/test-kitchen](https://github.com/test-kitchen/test-kitchen/pull/373 "Add new subcommand 'exec' by sawanoboly · Pull Request #373 · test-kitchen/test-kitchen")

`kitchen create`と`converge`の間に一手間加えるとか、`converge`と`verify`の間にちょっと味付けするとかに使えるかもしれない。特にCIの時。


そういえばその目的で、以前にこんな質問もしました。

[I would like to execute some tasks before chef-client run at `kitchen converge`. · Issue #251 · test-kitchen/test-kitchen](https://github.com/test-kitchen/test-kitchen/issues/251 "I would like to execute some tasks before chef-client run at `kitchen converge`. · Issue #251 · test-kitchen/test-kitchen")

その時推奨されたやり方は`test/`の下に何かをprepareするCookbookを用意する、または`data/`ディレクトリを置けばついでにアップする、でした。

`test/`の下にcookbookは便利なので結構使ってます。
ただ、Provisionツールを実行するより完全に前の段階で仕込みたい時は、Chefなら`client/solo.rb`にRubyコードとして仕込んでしまうとか、少々強引な手法を取る必要がありました。


### kitchen exec ヘルプ

使用方法は他のコマンドと一緒。実行するリモートコマンドは`-c`で渡してあげよう。

```
kitchen exec INSTANCE|REGEXP -c REMOTE_COMMAND  # Execute command on one or more instance
```


プラットフォームが1つの時は`INSTANCE|REGEXP`は省略可、あと全部を表す予約語の`all`もつかえます。


```shell
$ kitchen exec all -c 'hostname'
-----> Execute command on default-centos-64.
       default-centos-64
-----> Execute command on mafalt-centos-64.
       hogehoge-centos-64
```


#### 余談：PTY

test-kitchenのリモート実行系はrequiretty(NET::SSHのrequest_pty)に対応しているのでsudoが必要な環境でも安心のexec。
