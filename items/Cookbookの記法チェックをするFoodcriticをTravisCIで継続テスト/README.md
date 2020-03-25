<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[MVT: Foodcritic and Travis CI](http://technology.customink.com/blog/2012/06/04/mvt-foodcritic-and-travis-ci/) より、


OpsCode ChefのCookBookを書いたら評論家[Foodcritic](http://acrmp.github.com/foodcritic/)に見せる。
前述の案内の通りTravisCIでGithubにPushする度に自動でfoodcriticを実行しておいてもらう。

## foodcritic

`gem install foodcritic`かGemfileに書いて`bundle`。

とりあえず作成中の[aptからMongoDBをインストールしてあれこれするレシピ](https://github.com/higanworks-cookbooks/mongodb-10gen)に対して実行してみる。

```
$ foodcritic cookbooks/mongodb-10gen/
FC001: Use strings in preference to symbols to access node attributes: cookbooks/mongodb-10gen/recipes/replica-set.rb:48
FC001: Use strings in preference to symbols to access node attributes: cookbooks/mongodb-10gen/recipes/replica-set.rb:50
FC001: Use strings in preference to symbols to access node attributes: cookbooks/mongodb-10gen/recipes/single.rb:42
FC001: Use strings in preference to symbols to access node attributes: cookbooks/mongodb-10gen/recipes/single.rb:44
FC002: Avoid string interpolation where not required: cookbooks/mongodb-10gen/recipes/replica-set.rb:31
FC002: Avoid string interpolation where not required: cookbooks/mongodb-10gen/recipes/replica-set.rb:59
FC002: Avoid string interpolation where not required: cookbooks/mongodb-10gen/recipes/single.rb:25
FC019: Access node attributes in a consistent manner: cookbooks/mongodb-10gen/recipes/replica-set.rb:48
FC019: Access node attributes in a consistent manner: cookbooks/mongodb-10gen/recipes/replica-set.rb:50
FC023: Prefer conditional attributes: cookbooks/mongodb-10gen/recipes/replica-set.rb:58
```


ルール1,2,19と23に引っかかった、同様の出力をTravisCIからも得られるようにしよう。


## TravisCI

TravisCI用の設定をリポジトリに追加する。

### gitリポジトリに幾つか追加する

一応CookBookと被らないよう、専用のGemfileパスを使うよう案内されている。

```.travis.yml
language: ruby
gemfile:
   - test/support/Gemfile
rvm:
  - 1.9.2
  - 1.9.3
script: bundle exec rake foodcritic
```

`rake` と `foodcritic` をbundleするように指定。

```test/support/Gemfile
source "https://rubygems.org"

gem 'rake'
gem 'foodcritic'
```

```Rakefile
#!/usr/bin/env rake

desc "Runs foodcritic linter"
task :foodcritic do
  if Gem::Version.new("1.9.2") <= Gem::Version.new(RUBY_VERSION.dup)
    sandbox = File.join(File.dirname(__FILE__), %w{tmp foodcritic cookbook})
    prepare_foodcritic_sandbox(sandbox)

    sh "foodcritic --epic-fail any #{File.dirname(sandbox)}"
  else
    puts "WARN: foodcritic run is skipped as Ruby #{RUBY_VERSION} is < 1.9.2."
  end
end

task :default => 'foodcritic'

private

def prepare_foodcritic_sandbox(sandbox)
  files = %w{*.md *.rb attributes definitions files libraries providers
recipes resources templates}

  rm_rf sandbox
  mkdir_p sandbox
  cp_r Dir.glob("{#{files.join(',')}}"), sandbox
  puts "\n\n"
end
```


`rake`コマンドのデフォルトで`foodcritic`を実行するように指定。

下記URLで実行開始。

[https://travis-ci.org/#!/higanworks-cookbooks/mongodb-10gen](https://travis-ci.org/#!/higanworks-cookbooks/mongodb-10gen)


ローカルでfoodcriticを叩いた時と同じ出力をTravisCI上で確認できる。

では直していこう。
