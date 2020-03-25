
私の管理している`Knife-Zero`([higanworks/knife-zero](https://github.com/higanworks/knife-zero))について＠ITさんで紹介記事が出ていまいした。

[サーバー管理者のためのChef超入門（2）：Knife-ZeroでCookbookの作成／実行／削除＆git cloneコマンドでCookbookの取得 (1/2) - ＠IT](http://www.atmarkit.co.jp/ait/articles/1503/24/news030.html "サーバー管理者のためのChef超入門（2）：Knife-ZeroでCookbookの作成／実行／削除＆git cloneコマンドでCookbookの取得 (1/2) - ＠IT")

丁寧に解説いただいており、使う人も増えたらいいかなと思いました。
しかし次に引用する注釈については、一応やり方があるので書いておきます。

> 注釈
> 
> 　本来のChefの作法では、個人で作成したCookbookは「chef-repo/site-cookbooks」に格納し、Chef社が公開しているCookbookや第三者が作成したCookbookを「chef-repo/cookbooks」に格納することが推奨されています。
> 
> 　しかし2015年3月現在、Chef-Zeroの仕様で、「site-cookbooks」配下を参照しないため、個人作成のCookbookを「chef-repo/cookbooks」に作成するような手順としています。

地味に`site-cookbooks`作法って公式で言ってたかは気になったりします。ドキュメントにちらほら出てくるけどはっきりどこかで明言してたかなあ。
BarkShelfは言ってたかも？ 確かに自分はだいたいLibrarian-Chefでそのようにしているのでそれは置いて本題へ。

> 追記: site-cookbooksのお作法
> http://qiita.com/DQNEO/items/7843c50f1b74ae497b3c のコメントにも書きましたが、
> ごく初期からChefで想定されている使い方でした。(4コミット目)
> 
> https://github.com/chef/chef/blob/3dd77b260393bae93bb27677e3f8a45f407981c1/lib/chef/config.rb
>    def set_defaults
>      @cookbook_path = [ 
>        "/etc/chef/site-cookbook",
>        "/etc/chef/cookbook",
>      ]
>    end
>
> 初期すぎてむしろUndocumentedなんですかね。

## 実はKnifeの設定

Chef-Zero単品ではcookbooksだけですが、実はChef本体側でパスを指定するようになってます。

具体的にはknife.rbの`cookbook_path`。 参考: [knife.rb — Chef Docs](https://docs.chef.io/config_rb_knife.html "knife.rb — Chef Docs")

```knife.rb
local_mode true

cookbook_path [
  File.expand_path('../cookbooks', __FILE__),     ## knfie.rbのある場所からcookbooksへ絶対パス変換
  File.expand_path('../site-cookbooks', __FILE__) ## knfie.rbのある場所からsite-cookbooksへ絶対パス変換
]
```

## Cookbook一覧を確認しよう

ほいほいっと`cookbooks`, `site-cookbooks`の下に1つずつcookbookを作成します。 (※Chef-DKでもOK)

```
$ ./bin/knife cookbook create book1 -o cookbooks
** Creating cookbook book1 in /Users/sawanoboriyu/worktemp/knife-zero-cookbooks/cookbooks
** Creating README for cookbook: book1
** Creating CHANGELOG for cookbook: book1
** Creating metadata for cookbook: book1

$ ./bin/knife cookbook create site_book1 -o site-cookbooks
** Creating cookbook site_book1 in /Users/sawanoboriyu/worktemp/knife-zero-cookbooks/site-cookbooks
** Creating README for cookbook: site_book1
** Creating CHANGELOG for cookbook: site_book1
** Creating metadata for cookbook: site_book1
```

`knife cookbook list`で確認すると、両方のCookbookがリストアップされます。(`knife serve` でも同様)

```
$ ./bin/knife cookbook list
book1        0.1.0
site_book1   0.1.0
```

## site-cookbooks内のレシピを適用してみる

ひとまず分かりやすいように`package "tmux"`とでも書いておきます。

```
$ echo 'package "tmux"' >> site-cookbooks/site_book1/recipes/default.rb 
```

では`knife zero bootstrap`に`--run-list site_book1`を実行、適用されます。

```
$ ./bin/knife zero bootstrap hoge.example.com --run-list site_book1


## 省略

hoge.example.com resolving cookbooks for run list: ["site_book1"]
hoge.example.com Synchronizing Cookbooks:
hoge.example.com   - site_book1
hoge.example.com Compiling Cookbooks...
hoge.example.com Converging 1 resources
hoge.example.com Recipe: site_book1::default     # `site-cookbooks/site_book1` を見つけている。
hoge.example.com   * apt_package[tmux] action install (up to date)
hoge.example.com 
hoge.example.com Running handlers:
hoge.example.com Running handlers complete
hoge.example.com Chef Client finished, 0/1 resources updated in 3.293436307 seconds
```


`knife zero chef_client`もOK。

```
$ ./bin/knife zero chef_client 'name:hoge.example.com' -o site_book1

## 省略

hoge.example.com Recipe: site_book1::default
hoge.example.com   * apt_package[tmux] action install (up to date)
hoge.example.com [2015-03-24T11:51:55+00:00] WARN: Skipping final node save because override_runlist was given
```

以上補足でした。
