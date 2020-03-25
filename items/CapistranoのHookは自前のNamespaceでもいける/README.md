<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

deployをloadする必要も特に無く、`before`と`after`で好きなようにHookが可能。

```ruby:Capfile
namespace :first do
  task :one   do logger.info "1-1"; end
  task :two   do logger.info "1-2"; end
  task :three do logger.info "1-3"; end
end

namespace :sec do
  task :one   do logger.info "2-1"; end
  task :two   do logger.info "2-2"; end
end

namespace :third do
  task :bang do
    logger.important "I'm third."
  end
end

before "third:bang" do
  top.first.one
  top.first.two
end

before "first:one" do
  top.sec.one
end

before "sec:one" do
  top.sec.two
end

after "third:bang" do
  top.first.three
end
```

Hookされて数代前のBeforeも実行してくれる。
`before/after`にはもちろん普通の処理を書いてもOK。

```Shell:CLI_output
$ cap -n third:bang 
  * executing `third:bang'
    triggering before callbacks for `third:bang'
  * executing `first:one'
    triggering before callbacks for `first:one'
  * executing `sec:one'
    triggering before callbacks for `sec:one'
  * executing `sec:two'
 ** 2-2
 ** 2-1
 ** 1-1
  * executing `first:two'
 ** 1-2
*** I'm third.
    triggering after callbacks for `third:bang'
  * executing `first:three'
 ** 1-3
```

リモートそっちのけでアレコレやってもよいでしょう。ループにはご注意。
