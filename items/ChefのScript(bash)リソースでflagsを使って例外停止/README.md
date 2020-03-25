<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

chefのscriptリソースにbashを選ぶと、そのままではスクリプトの途中でエラーが出ても停止しない。

```ruby:recipe.rb
script "echo_test" do
  interpreter "bash"
  code <<-"EOH"
      ehco
      echo
  EOH
end
```

Scriptリソースは単に`code`の中身をファイルに書いてbashで実行している、exitコードは最後に実行したコマンドに依存するため、上記レシピではtypoの`ehco`があっても続きが実行されてしまう。

## flagsを使う

`flags "-e"`を指定すると、bashに -e `("set -e" と同じ)`オプションをつけてくれるので止まる。

```ruby:recipe_with_exception.rb
script "echo_test" do
  interpreter "bash"
  flags "-e"
  code <<-"EOH"
      ehco
      echo
  EOH
end

```

## 実行例

```
---- Begin output of "bash" -e "/tmp/chef-script20120815-11796-1cn4njv" ----
STDOUT: 
STDERR: /tmp/chef-script20120815-11796-1cn4njv: line 1: ehco: command not found
---- End output of "bash" -e "/tmp/chef-script20120815-11796-1cn4njv" ----
Ran "bash" -e "/tmp/chef-script20120815-11796-1cn4njv" returned 127
```

`ehco: command not found` で無事に止まった。
