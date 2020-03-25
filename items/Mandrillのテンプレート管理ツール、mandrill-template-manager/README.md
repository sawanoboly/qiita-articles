
定型メールの送信に、MailChimpさんとこの[Mandrill](http://mandrill.com/)を使おうとしています。アプリでそれぞれメールのテンプレートを持たないで済むかなと。
で、Mandrill上のテンプレートをバージョン管理に乗せるため、ツールを一個書きました。

https://github.com/higanworks/mandrill-template-manager

Rubygems配布もしてます。


## インストール

基本は単品で動きますが、ついでに入れておくと良いGeｍsと一緒に使うと良いです。

```Gemfile
gem 'mandrill-template-manager'

## 入れておくと使われる、表示成形用のGem達
gem 'formatador', github: 'geemus/formatador' # マルチバイト補正アリバージョン
gem 'unicode'

## テンプレートにHandlebarsを使う場合、ローカルプレビューもしたいときに追加。
# gem 'handlebars'
```

`bundle install --binstubs --path vendor/bundle`などとして、`../bin/mandrilltemplate`で実行できるようにしておくと楽です。

APIキーは環境変数へ。 `export MANDRILL_APIKEY='your api key'` な感じで。

## ワークフロー

1. 新規(generate)か、既存(export)を用いてテンプレートをローカルファイルとして管理開始
2. テンプレートを変更したらコミット
4. 本文プレビュー
3. Mandrillにアップロード、現状Publish済みのものと差分があればわかるように表示
5. ステータスをPublishへ


このへんは[codenize.tools](http://codenize.tools/)の影響を多分に受けています。

Mandrillのテンプレートは単に変更するだけならDraft扱いなので、`--dry-run`の変わりとしてプレビュー(render)をつけました。
WebのUIからならドラフトを元にしたテスト送信もできるのですが、APIには無いのかな？見つからなかった。

## やってみよう

では今あるテンプレートをバージョン管理に移行して、更新してみる。

### Mandrill上にあるテンプレートをエクスポート

まずは`mandrilltemplate list`でMandrill上にあるテンプレートと、ローカル(リポジトリ)にあるテンプレートの一覧を確認する。

```
 $ mandrilltemplate list
Remote Templates
----------------------
  +----------+-------------------+---------------------+-------------------+------------+--------------------+--------+
  | has_diff | name              | slug                | from_email        | from_name  | subject            | labels |
  +----------+-------------------+---------------------+-------------------+------------+--------------------+--------+
  |          | example           | example             | test@example.com  | Boss       | ようこそ           | []     |
  +----------+-------------------+---------------------+-------------------+------------+--------------------+--------+
Local Templates
----------------------
  +------+------+------------+-----------+---------+--------+
  | name | slug | from_email | from_name | subject | labels |
  +------+------+------------+-----------+---------+--------+
  +------+------+------------+-----------+---------+--------+
```

> `mandrilltemplate list`は`-v`オプションでリモートを詳細表示にできます。


リモートに`example`というテンプレートがあるね。ローカルにはまだ何もないです。
これを`mandrilltemplate export`する。

```
$ mandrilltemplate export example
      create  templates/example
      create  templates/example/metadata.yml
      create  templates/example/code
      create  templates/example/text
```

それぞれはテンプレートの管理画面で入力する内容になっています。

```metadata.yml
---
slug: example
labels: []
subject: "ようこそ"
from_email: test@example.com
from_name: Boss
```

codeの中身はHandlebarsのテンプレートを使いました。

```code
<div>
おはよう {{UserName}} くん。<br/>
今日の指令は...<br/>
{{#each ImpossibleMissions}}
  <ul>
    <li>{{this}}</li>
  </ul>
{{/each}}

というわけだ、よろしくやってくれたまえ。<br/>

なお、このメールは自動的に消滅する。検討を祈る。
</div>
```

`text`は今回はカラッポなので省略。

そうするとこれがローカルのテンプレートとして検出されるようになります。name=ディレクトリ名なので、変更に注意。

```
$ mandrilltemplate list

...

Local Templates
----------------------
  +---------+---------+------------------+-----------+----------+--------+
  | name    | slug    | from_email       | from_name | subject  | labels |
  +---------+---------+------------------+-----------+----------+--------+
  | example | example | test@example.com | Boss      | ようこそ   | []     |
  +---------+---------+------------------+-----------+----------+--------+
```

## メールを変更してupload

では本文を更新してみます。

```git:
$ git diff
diff --git a/templates/example/code b/templates/example/code
index fa21456..a365622 100644
--- a/templates/example/code
+++ b/templates/example/code
@@ -10,4 +10,6 @@
 というわけだ、よろしくやってくれたまえ。<br/>
 
 なお、このメールは自動的に消滅する。検討を祈る。
-</div>
\ No newline at end of file
+
+あと、帰りに大根買ってきて。
+</div>
```

`mandrilltemplate upload`でアップします。
APIの戻りを丸々出力していますがまあ気にしないで。

```
$ mandrilltemplate upload example
---
slug: example
name: example
code: |-
  <div>
  おはよう {{UserName}} くん。<br/>
  今日の指令は...<br/>
  {{#each ImpossibleMissions}}
    <ul>
      <li>{{this}}</li>
    </ul>
  {{/each}}

  というわけだ、よろしくやってくれたまえ。<br/>

  なお、このメールは自動的に消滅する。検討を祈る。

  あと、帰りに大根買ってきて。
  </div>
publish_code: |-
  <div>
  おはよう {{UserName}} くん。<br/>
  今日の指令は...<br/>
  {{#each ImpossibleMissions}}
    <ul>
      <li>{{this}}</li>
    </ul>
  {{/each}}

  というわけだ、よろしくやってくれたまえ。<br/>

  なお、このメールは自動的に消滅する。検討を祈る。
  </div>
published_at: '2015-09-10 05:50:37'
created_at: '2015-09-09 03:34:47.89299'
updated_at: '2015-09-10 06:06:27.13888'
draft_updated_at: '2015-09-10 05:50:37'
publish_name: example
labels: []
text: 
publish_text: 
subject: "ようこそ"
publish_subject: "ようこそ"
from_email: test@example.com
publish_from_email: test@example.com
from_name: Boss
publish_from_name: Boss
```

アップロードしたら、`has_diff`パラメータがtrueを持ちました。
DraftとPublishdに違いがあったらここに表示されます。

```
$ mandrilltemplate list -v
Remote Templates
----------------------
  +----------+-------------------+---------------------+-------------------+------------+--------------------+--------+
  | has_diff | name              | slug                | from_email        | from_name  | subject            | labels |
  +----------+-------------------+---------------------+-------------------+------------+--------------------+--------+
  | true     | example           | example             | test@example.com  | Boss       | ようこそ           | []     |
  +----------+-------------------+---------------------+-------------------+------------+--------------------+--------+

...
```


## プレビューしてみる

DraftはWebUIからテスト送信できます。ただ、ローカルに同じ内容を持っているのでCLIで本文くらいはプレビューできるようにしておきました。

まずは`先ほどのテンプレートに適用するパラメータファイルを用意しましょう、merge_vars.json`とします。

```merge_vars.json
[
  {
    "name": "UserName",
    "content": "ExampleUser"
  },
  {
    "name": "ImpossibleMissions",
    "content": [
      "Mission A",
      "Mission B",
      "Mission C"
    ]
  }
]
```

今回のテンプレートをプレビューするには`mandrilltemplate render --handlebars`で。

```
$ mandrilltemplate render --handlebars example templates/example/merge_vars.json 
<div>
おはよう ExampleUser くん。<br/>
今日の指令は...<br/>
  <ul>
    <li>Mission A</li>
  </ul>
  <ul>
    <li>Mission B</li>
  </ul>
  <ul>
    <li>Mission C</li>
  </ul>

というわけだ、よろしくやってくれたまえ。<br/>

なお、このメールは自動的に消滅する。検討を祈る。

あと、帰りに大根買ってきて。
</div>
```

プレビュー良好ですね。

デフォルトはAPIのrenderメソッド(mailchimpフォーマット)を使ってレンダリングします。
しかしHandlebarsを使っている場合はAPIのrenderできないので、とりあえずローカルのRubyで処理することにしました。
`--handlebars`オプションを渡して動作を切り替えます。

## Publishして本番適用

色々と気が済んだら`mandrilltemplate publish`して本番適用です。

```
$ mandrilltemplate publish example
```

これでdraft状態のテンプレートがpublishedになりました。

## おわりに

コードなので何かしらCIと連携することができます。
たとえばCircleCIならアーティファクトとしてファイルに書きだすなり、パイプでメールを実際に送信したらよさそう。Publishへの変更を自動でやるかはお好みで。

なお、Mandrill上に実際にあるテンプレートがいつのものかは日付から判断するしか無いので、もうちょい厳密に見るならばlabelにgitのsha1でもつけておくのも良いかもしれない。
