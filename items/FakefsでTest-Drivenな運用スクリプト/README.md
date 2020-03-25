<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

見せかけのFilesystemを操作して、運用系のRubyスクリプトをテストする。

FakeFS[https://github.com/defunkt/fakefs]


## ディレクトリ・ファイルを操作するサンプルツール

ディレクトリを操作するメソッドを用意、屋上に屋根状態だが実際はあれこれと処理をしてから実行します。

```ruby:app.rb
#!/usr/bin/env ruby
# cording: utf-8

def create_dir(dir_name)
  Dir.mkdir(dir_name)
end

def remove_dir(dir_name)
  Dir.rmdir(dir_name)
end
```

## Fakefsを使ったテストを書く

fakefsは`Dir`やら`File`,`symlink`あたりをオーバーライドして、仮想的なファイルシステム上での操作にしてくれる。

```ruby:test_app.rb
#!/usr/bin/env ruby                                                                                                                                                       # cording: utf-8
require 'bundler/setup'
require 'test/unit'
require 'fakefs'

require "./app"

class AppTest < Test::Unit::TestCase

  def test_create_dir
    create_dir("hoge")
    assert Dir.exists?("hoge") 
  end

  def test_remove_dir
    Dir.mkdir("piyo")
    remove_dir("piyo")
    assert !Dir.exists?("piyo") 
  end
end
```

このテストで`test_create_dir`の後ならディレクトリ`hoge`が残っているはずだが、実ファイルシステム上には存在しない。

## テスト実施

実行のサンプル

```test
$ ./test_app.rb 
Run options: 

# Running tests:

..

Finished tests in 0.007162s, 279.2516 tests/s, 279.2516 assertions/s.

2 tests, 2 assertions, 0 failures, 0 errors, 0 skips

```

テストOK、`hoge`も`piyo`も実ファイルシステム上には作られない。
