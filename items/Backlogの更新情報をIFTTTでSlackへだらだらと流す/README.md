
![スクリーンショット_10_23_14__4_12_PM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d653d606-2b2e-e686-036f-bafae0553787.png "スクリーンショット_10_23_14__4_12_PM.png")

最近[Backlog](http://www.backlog.jp/)を使う相手が多いので、更新情報をSlackにも流すことにしました。


## 方針

連携には自前のサーバやプログラムをできれば使いたくない。
BacklogのダッシュボードにRSS(Basic認証)があるので、それを使ってみよう。

## 段取り

1. RSSフィードが読めるように、Backlogにゲストユーザを作成
1. ゲストユーザをプロジェクトに参加させる
1. IFTTTで`RSS then Slack`


## Backlogにゲストユーザを作成

RSSフィードはAPIキーで取れない気がしたので、Basic認証ができるように適当なゲストユーザを作成しました。メインユーザのID/Passを使うのは一応避けておきます。

![HiganWorks_チケット取り扱い所-_Backlog.png](https://qiita-image-store.s3.amazonaws.com/0/7454/6b9070f3-41bc-bb2a-ef7b-c52eff196d31.png "HiganWorks_チケット取り扱い所-_Backlog.png")

ゲストのメールアドレスはgmailの`+`でつくるエイリアスにしておくといいです。

RSSが取れるかチェックしよう、URLは次のようにすればOK。

`https://{user-id}:{password}@{org-name}.backlog.jp/rss/{Project-ID}`


## あとはIFTTT

で、If Then。

If: new feed item from Backlog(RSS)
Then: post a message to a Slack channel

RSSなので15分おきにまとめて通知されます。

![Slack.png](https://qiita-image-store.s3.amazonaws.com/0/7454/48433655-5685-c306-cbe8-b000c8b64630.png "Slack.png")

RSSにフィルタを掛けて、複数のチャンネルに振り分けてもいいですね。

Messageフィールドのオススメはこう。

```
{{EntryAuthor}} {{EntryContent}}
```

ああ、15分で50件以上の更新があったら漏れますが。
