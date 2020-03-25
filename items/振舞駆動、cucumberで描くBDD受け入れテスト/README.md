<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[cucumber](http://cukes.info/)は受け入れテストにいい。
開発者が書くのはまったくもってよろしくないが、承認側としては読めないプロダクション用のコードをレビューするより実用的だ。

## 受け入れ側の書くシナリオ

冗談のようだが日本語でいい。
`ssl_check.feature`というファイルには正にこのように書いてある。


```cucumber:ssl_check.feature
# language: ja
機能: 証明書期限のテスト
  証明書の更新漏れを確かめるために
  利用者の立場から
  期限を確かめたい

シナリオ: 証明書期限について
  前提 my.z-cloud.jp にアクセスして
  もし 証明書の情報がとれた
  ならば 期限が 360 日より長いか確認する

シナリオ: 証明書期限について
  前提 my.z-cloud.jp にアクセスして
  もし 証明書の情報がとれた
  ならば 期限が 30 日より長いか確認する
```

## 受け入れテストを実施する

先ほどのシナリオを実行する。

```
$ cucumber 
# language: ja
機能: 証明書期限のテスト
  証明書の更新漏れを確かめるために
  利用者の立場から
  期限を確かめたい

  シナリオ: 証明書期限について           # features/ssl_check.feature:7
    前提my.z-cloud.jp にアクセスして # features/step_definitons/steps.rb:2
    もし証明書の情報がとれた            # features/step_definitons/steps.rb:17
    ならば期限が 360 日より長いか確認する   # features/step_definitons/steps.rb:21
      証明書の残り期限は 34日 のようです (RuntimeError)
      ./features/step_definitons/steps.rb:23:in `/^期限が (\d+) 日より長いか確認する$/'
      features/ssl_check.feature:10:in `ならば期限が 360 日より長いか確認する'

  シナリオ: 証明書期限について           # features/ssl_check.feature:12
    前提my.z-cloud.jp にアクセスして # features/step_definitons/steps.rb:2
    もし証明書の情報がとれた            # features/step_definitons/steps.rb:17
    ならば期限が 30 日より長いか確認する    # features/step_definitons/steps.rb:21
      証明書の残り期限は 34日 あります

Failing Scenarios:
cucumber features/ssl_check.feature:7 # Scenario: 証明書期限について

2 scenarios (1 failed, 1 passed)
6 steps (1 failed, 5 passed)
0m0.206s
```

想定したシナリオ通りになっていないものをFailさせる。
