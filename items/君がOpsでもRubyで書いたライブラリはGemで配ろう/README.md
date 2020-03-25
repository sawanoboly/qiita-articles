<!-- permanent -->

`Infrastracture as code`流行の副産物として、もうOpsはある程度Rubyでライブラリを書けるようになりました。

折角Rubyでライブラリを書いたなら、安全＆ラクに配布するためGemパッケージにしましょう、出来る人には今更でしょうが知らない人は真似してみてね。
ちなみに意外と誤解されてる点、`gemにする＝Rubygems.orgで公開する`、ではありません、してもOKというだけで。

## 目標

gemファイルを置いて、gemコマンドで自分のライブラリをサーバに導入する。

こんなかんじで。
`gem install -l my_libs-0.0.1.gem`

じゃあやってみましょう。


## Gemの雛形をつくろう

Gemの作り方は色々あるようですが、私はもっぱらbundlerです。
`bundle gem`で必要なファイル群を作成します、便利ですね。

```shell:Shell
$ bundle gem my_libs
      create  my_libs/Gemfile
      create  my_libs/Rakefile
      create  my_libs/LICENSE.txt
      create  my_libs/README.md
      create  my_libs/.gitignore
      create  my_libs/my_libs.gemspec
      create  my_libs/lib/my_libs.rb
      create  my_libs/lib/my_libs/version.rb
```
作られたファイルで、触るのはこれだけ。

|ファイル名|役割|
|---|---|
|my_libs.gemspec|gemの説明と、依存する他のrubygemsがあればそれを記述します。設定ファイルのコメントくらいに捉えていればOK。|
|lib/my_libs/version.rb|このgemバージョンを書きます、zoneファイルのシリアルよろしく何かする度適当に増やしていきましょう。|
|lib/my_libs.rb|自前の関数をここに書きましょう。|

メインは`lib/my_libs.rb`です。

## 概要をgemspecに

初めにこのGemが何するものぞを記述する必要があります、特に難しいことではありません。

`my_libs.gemspec`の初期状態はこのようになっています。

```ruby:my_libs.gemspec(initial)
# coding: utf-8
lib = File.expand_path('../lib', __FILE__)
$LOAD_PATH.unshift(lib) unless $LOAD_PATH.include?(lib)
require 'my_libs/version'

Gem::Specification.new do |spec|
  spec.name          = "my_libs"
  spec.version       = MyLibs::VERSION
  spec.authors       = ["sawanoboly"]
  spec.email         = ["sawanoboriyu@higanworks.com"]
  spec.description   = %q{TODO: Write a gem description}
  spec.summary       = %q{TODO: Write a gem summary}
  spec.homepage      = ""
  spec.license       = "MIT"

  spec.files         = `git ls-files`.split($/)
  spec.executables   = spec.files.grep(%r{^bin/}) { |f| File.basename(f) }
  spec.test_files    = spec.files.grep(%r{^(test|spec|features)/})
  spec.require_paths = ["lib"]

  spec.add_development_dependency "bundler", "~> 1.3"
  spec.add_development_dependency "rake"
end
```


親切なことに**TODO**がありますね、`my_libs.gemspec`この２行を編集します。


```ruby:my_libs.gemspec
  spec.description   = %q{TODO: Write a gem description}
  spec.summary       = %q{TODO: Write a gem summary}
```

`description`と`summary`は意味的にそのままですが、今回は特に区別しません。短めにgemの説明を書いておきましょう。

```ruby:my_libs.gemspec
  spec.description   = %q{ruby sample libs for me}
  spec.summary       = %q{ruby sample libs for me}
```

### 依存するRubyGems

自前ライブラリ内で他のgemを使いたいことがある場合は`my_libs.gemspec`に追加しておきます。

RubyGems.orgで公開されている`twitter`が必要なら適当な箇所に１行この様に書けばOK、`gem install`実行時に警告を出してくれます。

```ruby:my_libs.gemspec
  spec.add_dependency 'twitter'
```
今回は特に何も追加せずに先へいきましょう。

## my_libを読み込んでみる

この時点ですでに他のgemのように機能します、中身はカラですが読み込んでみましょう。
Rubyの対話型インタプリタ`Irb`を起動します。

```shell:Irb
$ bundle exec irb
> require 'my_libs'
 => true 
> MyLibs
MyLibs
> MyLibs::VERSION
 => "0.0.1" 
```

## ライブラリを書こう

では`my_libs/lib/my_libs.rb`に自前ライブラリを書いて行きましょう、初期状態は下記です。

```ruby:my_libs/lib/my_libs.rb
require "my_libs/version"

module MyLibs
  # Your code goes here...
end
```

こちらもわかりやすく**`# Your code goes here…`**となっています。

サンプルとして外から呼べる`hello`とinclude/extend用の`hello2`メソッドを実装します。

```ruby:my_libs/lib/my_libs.rb
require "my_libs/version"

module MyLibs
  def self.hello
    'hello gem!'
  end

  def hello2
    'hello gem! 2'
  end
end
```

モジュールを記述したので、先程の要領で読み込んで使っていきましょう。


## ライブラリのメソッドを実行しよう

```shell:Irb
$ bundle exec irb
> require 'my_libs'
 => true 

> MyLibs.methods(false)
 => [:hello] 

> MyLibs.hello
 => "hello gem!" 

> include MyLibs
 => Object 

> self.methods.include?(:hello2)
 => true 

> hello2
 => "hello gem! 2" 
```

`require 'my_libs'`から後なら`MyLibsモジュール`が使えるようになっていますね。
これを`gem install`で使えるようにするために、gemパッケージにします。

## Gemをビルドしよう

`bundle gem`で作った雛形にはgemパッケージ用のタスクが含まれています。

```shell:Shell
$ rake -vT
rake build    # Build my_libs-0.0.1.gem into the pkg directory.
rake install  # Build and install my_libs-0.0.1.gem into system gems.
rake release  # Create tag v0.0.1 and build and push my_libs-0.0.1.gem to Rubygems
```
このうち`rake build`がgemファイルを作るタスクです、早速実行してみます。

```shell:Shell
$ rake build
my_libs 0.0.1 built to pkg/my_libs-0.0.1.gem.

$ ls pkg/
my_libs-0.0.1.gem
```

`pkg/`以下にgemファイルができました。`my_libs-0.0.1.gem`とバージョン情報が付与されてますね。


試しにバージョンを更新してみましょう、ライブラリは特に変更せず。


```ruby:my_libs/lib/my_libs/version.rb
module MyLibs
  VERSION = "0.0.2"
end
```

再び`rake build`

```shell:Shell
$ rake build
my_libs 0.0.2 built to pkg/my_libs-0.0.2.gem.
$ ls pkg/
my_libs-0.0.1.gem	my_libs-0.0.2.gem
```

目論見通り`my_libs-0.0.2.gem`となりました。

これでGemファイルが作れるようになったので、配布したいサーバに適当な手段で持っていきましょう。


## 作ったGemをインストールしよう

ローカルのgemファイルをインストールするには、`gem install`時に`-l`オプションでファイル名を指定します。
※ 依存にRubyGemsで公開されているGemがある時は、`-b` オプションにしましょう。


```shell:Shell
$ gem install -l pkg/my_libs-0.0.2.gem -V
Installing gem my_libs-0.0.2
PATH_TO_RUBY/gems/my_libs-0.0.2/.gitignore
PATH_TO_RUBY/gems/my_libs-0.0.2/Gemfile
PATH_TO_RUBY/gems/my_libs-0.0.2/LICENSE.txt
PATH_TO_RUBY/gems/my_libs-0.0.2/README.md
PATH_TO_RUBY/gems/my_libs-0.0.2/Rakefile
PATH_TO_RUBY/gems/my_libs-0.0.2/lib/my_libs.rb
PATH_TO_RUBY/gems/my_libs-0.0.2/lib/my_libs/version.rb
PATH_TO_RUBY/gems/my_libs-0.0.2/my_libs.gemspec
Successfully installed my_libs-0.0.2
1 gem installed
```

一旦インストールしてしまえばあとは通常のgemsと扱いは変わりません。

```shell:Shell
$ gem list | grep my_libs
my_libs (0.0.2)
```

ついでに実行テストをしてみます。

```shell:Shell
$ ruby -e "require 'my_libs'; puts MyLibs.hello"
hello gem!
```

できましたね。


## 最後に

一つ注意点として、作成するmodule,classの一番親に来る名前は他と被らないようにしましょう。

これはOKですが、

```ruby:sample.rb
module MyOriginModule
  class File
    # hogehogehoge
  end
end
```

この様にド頭に一般名称などを持ってくると他のモジュールとミックスされて、**色々と他人がヒドイ目に遭う**のでお気をつけて。

```ruby:sample.rb
class File
    # hogehogehoge
end
```
