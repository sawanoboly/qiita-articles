<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


rvmにdoというオプションがある、使用例は[公式の`rvm test`](https://rvm.io/set/tests/)。

渡したシェルを複数のrvmコンテキストで別々に実行してくれる。主にテスト向けの機能。


### バージョン表示のrakeタスク

それぞれのRubyでバージョンを表示するだけのタスクを走らせてみる。

```ruby:Rakefile
desc "print version"
task :print_ruby_version do
  puts ENV['RUBY_VERSION']
end
```


### カンマ区切りでバージョンを複数指定して実行

```
$ rvm ruby-1.9.2-p320,ruby-1.9.3-p194 do rake print_ruby_version
ruby-1.9.2-p320
ruby-1.9.3-p194
```

### allでインストール済みのすべてのバージョンで実行

```
$ rvm all do rake print_ruby_version
jruby-1.6.7.2
ruby-1.9.2-p320
ruby-1.9.3-p194
ruby-1.9.3-p125
```

例えば`.travis.yml`を勝手に拾ってテストしてみるのも良い。
