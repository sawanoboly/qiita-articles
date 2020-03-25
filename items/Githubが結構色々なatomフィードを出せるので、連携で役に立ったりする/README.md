
自分がオーナのリポジトリはサービス連携やらWebHookやら色々あるので、他との連携は容易です。
ただ、他所様のリポジトリの更新具合を見たり、何かが進んだら結合テストを流しておきたい時などに、とりあえずフィード購読からトリガーを作成できます。



|要素 |説明 |例 |
|----|----|----|
|releases |Githubに登録されたリリース |https://github.com/higanworks/sakurraform/releases.atom|
|wiki |Wikiの更新履歴 |https://github.com/fog/fog-sakuracloud/wiki.atom |
|{org or user}/{repo}/commits/{branch}|任意のブランチのコミットログ |https://github.com/higanworks/sakurraform/commits/master.atom|
|{org or user}/{repo}/commits/{branch}/{path_to_file}|任意のブランチで、特定ファイルに関わる更新を含むコミットログ |https://github.com/higanworks/sakurraform/commits/master/lib/sakurraform/version.rb.atom|


これで IFTTTでFeed購読 => 何か。が捗ります。最近作った連携はこんな感じ。

- リリースを
   - メールで自分におしらせ
   - CIのビルドをキック
- master更新を
   - DockerhubのAutoBuildをキック

他にも、見落としがちなWikiの更新をチャットに突っ込むとかで使えそう。

ちなみにアクティビティのタイムラインもフィードを出していて、ユーザ単位、Org単位、ひいてはGithubグローバル(!)なんかもあるけど、さすがにそれは範囲が広い。
Orgをうまいことフィルタすれば、特定リポジトリのIssueやPRをトリガにできるかも。(メールならwatchで十分ですが)


## プライベートリポジトリの場合

プライベートリポジトリからフィードを取ってくる場合は、今のところBasic認証の形式でユーザ名にTokenを使えばOK。

こうですね、

`https://{Personal access token}:@github.com/higanworks/circleci-private-sandbox/commits/master.atom`
