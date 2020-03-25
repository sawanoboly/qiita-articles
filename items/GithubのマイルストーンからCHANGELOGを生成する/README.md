<!-- permanent -->

IssueとPRベースのタスク管理は利用者にとって楽なものだけど、デプロイだけやる人などがまとめて見るにはちと追いづらかったりする。

そんな時にChangelogがしっかりしてると助かるので、バージョニングとマイルストーンの作成に一定の規約を設けて自動生成する運用を試しています。


## マイルストーンとPRのルール

自動生成のための要件はこちら。

- バージョン番号のマイルストーンを作成する
    - v0.0.1, v1.0.0.rc1 などセマンティックバージョニング風でつける。
        - 一旦Gemのバージョンとしてソートするので、`0.0.9`と`0.0.10`がひっくり返らない
    - マイルストーンにDescriptionを書いておくと、blockquoteで取り込む。
- 変更は基本的にPRで取り込む
    - CHANGELOGに並ぶのはマイルストーンに紐付けたPRのタイトル
    - それぞれ差分へのリンク付き
- 変更がIssueであっても、マイルストーンを付けるといちおう並ぶ
- ラベルを活用する
    - Bug, FeatureなどつけておくとPRの下に表示される


## PRやIssueに対して、次のような感じで対応する。

![ヘッダにRakeの注意書きを追加_by_sawanoboly_·_Pull_Request__1_·_higanworks_changelog_from_milestones.png](https://qiita-image-store.s3.amazonaws.com/0/7454/91540536-4063-dae9-85e4-63bb13cfae65.png "ヘッダにRakeの注意書きを追加_by_sawanoboly_·_Pull_Request__1_·_higanworks_changelog_from_milestones.png")


## コードと出力例

`Rakefile`を含む出力例は下記Githubリポジトリに置いてます。
[higanworks/changelog_from_milestones](https://github.com/higanworks/changelog_from_milestones)


### Rakeタスク

一応こんな感じ。とてもベタ書き。

`ENV['GITHUB_REPO']`でリポジトリ、プライベートの場合は`ENV['GITHUB_API_TOKEN']`をセットして使います。

```ruby:Rakefile
require 'bundler'
Bundler.setup
require 'octokit'
require 'sanitize'
# @client = Octokit::Client.new(:access_token => ENV['GITHUB_API_TOKEN'], auto_paginate: true)
@client = Octokit::Client.new(auto_paginate: true)

@milestones = @client.milestones(ENV['GITHUB_REPO'], {state: 'all'})

task :default do
  puts File.read('./_templates/header.txt')
  titles = create_array_of_milestone_titles
  titles.each do |title|
    ms = get_milestone_by_title(title)
    next unless ms
    puts "## [#{ms.title}](#{ms.url}) (#{ms.state})"
    puts "created_at: #{ms.created_at.to_s}  "
    streak = ""
    if ms.closed_at
      ((ms.closed_at - ms.created_at).to_i / 60 / 60 / 24).times {streak << "■"}
      puts "streak: #{streak}  "
      puts "closed_at: #{ms.closed_at.to_s if ms.closed_at}"
    end
    puts "<blockquote>"
    if ms.description.empty?
      puts "No Description."
    else
      puts ms.description
    end
    puts "</blockquote>"
    if ms.respond_to?(:number)
      issues = collect_issues_by_ms(ms.number)
      issues.each do |issue|
        str = "- "
        str <<  Sanitize.fragment(issue.title, Sanitize::Config::RESTRICTED)
        if issue.assignee
          str << " by #{issue.assignee.login}"
        end
        if issue.pull_request
          str << " at "
          str << "[PR-#{issue.number}](#{issue.pull_request.html_url}/files)"
        end
        puts str
        label_cols = issue.labels.map do |label|
          %Q{![#{label.name}](https://img.shields.io/badge/L-#{URI.encode(label.name)}-#{label.color}.svg)}
        end
        puts "    - #{label_cols.join(' ')}" unless label_cols.empty?
      end
    else
      puts "- Nothing comment"
    end

    puts ""
    puts "----"
  end
  puts File.read('./_templates/footer.txt')
end

def create_array_of_milestone_titles
  tags = @milestones.map do |a|
    begin
      Gem::Version.new(a.title.delete('v'))
    rescue
      next
    end
  end.sort.reverse

  tags.map {|tag| "v" + tag.to_s}
end

def get_milestone_by_title(title)
  @milestones.find {|m| m.title == title}
end

def collect_issues_by_ms(ms_number)
  @client.issues(ENV['GITHUB_REPO'], {state: 'all', milestone: ms_number})
end
```


## 出力例

IssueやPRはサンプルプロジェクト([higanworks/changelog_from_milestones](https://github.com/higanworks/changelog_from_milestones))で実際に作ったので、どんな様子か確認できるはず。


![スクリーンショット_3_6_15__6_22_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/bf742187-9625-3206-85f2-0081ca953832.png "スクリーンショット_3_6_15__6_22_PM.png")


mdとして出力するので、今はプロジェクトのwikiに`CHANGELOG.md`を置いて運用中。
