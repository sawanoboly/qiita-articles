<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

ウェブのコンテンツをGitで管理している際に、Webサーバ上でGitリポジトリをチェックしつつ、同期して何かスクリプトを実行する。という手順をChefのレシピで実行してみます。


## 手順におこす

まずやりたいことをスクリプト(台本)にしましょう。

1. 初回はgit clone してシェルスクリプトを流す
2. 定期的にgit ls-remoteして新しいものをチェックする
3. ls-remote の結果、最新がローカルのリポジトリと違ったらpull
4. pull があったらシェルスクリプト実行

まあ`bash`で十分な内容です。４なんかは gitのhookを使うやり方もありますね。
ただ従来bashや他のスクリプト型言語で作られている実行手順はレシピにして`chef-apply`で実行すると読みやすい内容で十分な結果を得ることができます。

## chef-apply用のレシピ

Chef11からCookbook形式にまとめること無くレシピを実行できる`chef-apply`というユーティリティが追加されました。
先程の台本をレシピにするとこうなります。

```ruby:apply.rb
bash 'after_sync_script' do
  action :nothing
  flags '-x'
  code <<-__EOL__
  install -o www -g www -m 0600 /usr/local/project/index.html /usr/local/www/index.html
  install -o www -g www -m 0600 /usr/local/project/title.jpg /usr/local/www/title.jpg
  __EOL__
end

git '/usr/local/project' do
  action :sync
  repository 'file:///root/git_work'
  notifies :run, 'bash[after_sync_script]'
end
```

### リソース：`bash 'after_sync_script'`

`action :nothing` がポイントです、通常何もしませんが他から呼べるようにリソースとして定義だけしておきます。
中身はgitを同期したあとに実行したい、インストールスクリプトのサンプルですね。

### リソース：`git '/usr/local/project'`

gitのリモートリポジトリを`/usr/local/project` にも置きましょうというリソースです。
`action :sync`なので同期ですね、リモート側に更新があったらpull(fetch&checkouy)してくれます。
最後にある`notifies :run, 'bash[after_sync_script]'`で、この**リソースに更新があった場合** 先程のbashを`:run`してくれということを記述しています。



## 実行の様子

### 初回

最初の実行なのでローカルのリポジトリ、`/usr/local/project`ディレクトリはありません。
gitのcloneとcheckoutが実行されて、 `bash[after_sync_script] action run` が呼び出されているのがわかります。

```bash:ShellOut(chef-apply)
$ chef-apply apply.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * bash[after_sync_script] action nothing (skipped due to action :nothing)
  * git[/usr/local/project] action sync
    - clone from file:///root/git_work into /usr/local/project
    - checkout ref 4518501d9787c68a94150be5065ed62a7db103d3 branch HEAD
  * bash[after_sync_script] action run
    - execute "bash" -x "/tmp/chef-script20130729-67139-2jdojy"
```

### git更新なしの2回目以降

リモートリポジトリに変更がないので、なにも起こりません。
リソース`git[/usr/local/project]` が **up to date** と判断されていますね。

```bash:ShellOut(chef-apply)
# chef-apply apply.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * bash[after_sync_script] action nothing (skipped due to action :nothing)
  * git[/usr/local/project] action sync (up to date)
```

### gitに更新があった場合

gitの更新分をsyncして、bashのリソースが呼ばれているのがわかります。

```bash:ShellOut(chef-apply)
$ chef-apply apply.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * bash[after_sync_script] action nothing (skipped due to action :nothing)
  * git[/usr/local/project] action sync
    - set up remote tracking branches for file:///root/git_work at origin
    - fetch updates for origin
  * bash[after_sync_script] action run
    - execute "bash" -x "/tmp/chef-script20130729-76349-1yi7vfw"
```

あとはchef-applyをcronにでも放り込んでおけばOKですね。

## おわりに

bashのややこしいシェルスクリプトをchef-applyで置き換えるとスッキリするケースは結構あるんじゃないでしょうか。メンテする行数が少なくなれば品質の低下も抑えられるのでお奨めです。

なお今回紹介したgit リソースには沢山のオプションがあり、単純にsyncするほか.gitを除去したり、ブランチを指定したりとそこそこ色々なノウハウをそのままレシピにすることができるよう工夫がされています。

**Git: Opscode Docs**
[http://docs.opscode.com/resource_git.html](http://docs.opscode.com/resource_git.html)

