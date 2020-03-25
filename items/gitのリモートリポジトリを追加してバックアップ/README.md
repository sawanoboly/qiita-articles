<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

共用のリポジトリに上げるほどでも無い運用ツールなどをgitのリモートを使って適当にBackupする。
下手にアーカイブなんかを作るよりも使い勝手が良いのでhookも使って自動化しておくのがよい。


## リモート側受け入れ準備
まず取りたい場所に空のリポジトリを作成します。
リモートにはURLを指定出来るので、リソースとして使用出来るところならどこでも良い。ディレクトリを作ってgit initする。

```bash:git_initialize
mkdir repo01 &&  cd repo01
git init
git config --add receive.denyCurrentBranch ignore
```

denyCurrentBranch をしてPushを受け入れるようにしたらとりあえず準備OK。

## ローカルでリモート追加

作業中のリポジトリで、リモートを追加しよう。urlで指定。

    git remote add remote1 file:///mnt/repo1
    git remote add remote2 ssh://your_server/opt/repo1

ローカルファイル、マウントしたNFSやssh越しなどお好みで。

addしたら remote show で接続できているか確認する。情報が取れたらOK。

```bash:CUI_output(Local)
  $ git remote show remote2
* remote remote2
  Fetch URL: ssh://your_server/opt/repo1
  Push  URL: ssh://your_server/opt/repo1
  HEAD branch: (unknown)
```


## pushする

remote1にmasterをPushする。

    git push remote1 master


### pushされた内容の反映
pushだけでは見える形で反映されないのでresetしてあげる。
git statusを見れば理由がわかるので詳しいことは省略。

```bash:CUI_output(Remote)
$ ls -a
./  ../ .git

$ git reset --hard HEAD
$ ls -a
./  ../ README .git

```


## hookで自動化

### リモートで自動 reset --hard
サンプルからpost-receiveを用意。

    cp .git/hooks/post-receive.sample .git/hooks/post-receive
    chmod +x .git/hooks/post-receive

最後にこんなかんじで追記しておくとpush完了後にresetがかかる。

```bash:post-receive
-- snip --
cd ..
git --git-dir=.git reset --hard HEAD
````


### ローカルコミット時に自動push
ローカルの方はpost-commitに書いておく。

```bash:post-commit
-- snip --
git push remote1 master
```


