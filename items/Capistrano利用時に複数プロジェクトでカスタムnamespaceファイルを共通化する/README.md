<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Capistranoでプロジェクトが複数になってくるとNamespaceの使い回しが多くなる。
loadメソッドで外部からレシピを取り込める。

<pre><code>.
├── proj_a
│   └── config
│       ├── deploy
│       │   ├── production.rb
│       │   └── staging.rb
│       └── deploy.rb
├── proj_b
│   └── config
│       └── deploy
├── lib
    └── echo.rb
</code></pre>

libあたりに共通化したいレシピのecho.rbを置いてみる。


```ruby:echo.rb
namespace :hello do
  task :world do
    puts "good night."
  end
end
```

そしたら`deploy.rb`内でloadする。

    load '../lib/echo'
