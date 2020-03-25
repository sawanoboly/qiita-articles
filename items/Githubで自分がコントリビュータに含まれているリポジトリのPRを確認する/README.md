
気づいたら500くらいのリポジトリがあり、PRをたまに放置してしまっているのです。
とりいそぎプライベートトークンを発行し、その権限で取れるリポジトリ一覧からPR数だけ取得してみた。


```ruby:Rakefile
require 'octokit'

@client = Octokit::Client.new(:access_token => ENV["GH_TOKEN"])
@client.auto_paginate = true

task :default do
  repos = @client.repos
  repos.map do |repo|
    count = @client.pull_requests(repo.full_name, status: "open").length
    puts [count, repo.full_name].join(" - ")
  end
end
```

担当やオーナで絞れば精度も良くなるが、とりいそぎ全部だ。

## 実行。

まずは数と名前だけずらずらと。0はスキップでよかったかな。

```
$ rake

(略)

0 - giraffi/dokku-pg-plugin
0 - giraffi/dokku-postgresql-plugin
1 - giraffi/dokku-slack - 
0 - giraffi/dokku_sample_ruby

(略)
```

完全放置(一例)発見!! ごめんなさい。


https://github.com/giraffi/dokku-slack/pull/1

あああ、丸一年放置していたのだね。ちょっとリマインダを開発しよう。

