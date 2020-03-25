
Rubyの多言語対応で探すと(私の守備範囲では)だいたい[i18n](https://github.com/svenfuchs/i18n)。そしてほとんどWebアプリ≒Railsの話。
たまにはただのRubyプログラムやCLIでもロケール(locale/LANG)に対応した出力をしてみるのもいいかなと動かしてみた。


## i18n全体のながれ

- 言語対応表保存のバックエンドを選択
- バックエンドに言語の対応表をロード(都度取得も可？)
- ロケールとキーからそれぞれの中身を返す

準備さえしておけば、ふだんputsやLoggerに渡しているStringをi18nから返せば良さそうである。

## 言語ファイルでやってみる

試しに２つの言語ファイル、`en.yml`, `ja.yml`を作成。


これ英語。

```locales/en.yml
---
en:
  of: of the people
  by: by the people
  for: for the people
```

こっち日本語。

```locales/ja.yml
---
ja:
  of: 人民の
  by: 人民による
  for: 人民のための
```


## シンプルなバックエンドを選択し、起動時にロードする

とりあえず単品で動けばいいので、一番単純そうなバックエンド`I18n::Backend::Simple`を選択。
変換の処理も色々呼び方があるので、適当に変えながら出力してみよう。

```app.rb
#!/usr/bin/env ruby
# coding: utf-8
require 'i18n'
require 'yaml'

I18n.enforce_available_locales = true ## 対応ずみ言語かを実行時にチェックして止める
I18n.backend = I18n::Backend::Simple.new

## Yamlのパスをアレイで渡すとロードする
I18n.backend.load_translations(Dir.glob('locales/*.yml'))

## backend.translateにロケールとキーを渡すパターン
%w[en ja].map do |lang|
  puts I18n.backend.translate lang.to_sym, :of
end

## I18n.tにオプションでロケールを渡すパターン
I18n.backend.available_locales.map do |lang|
  puts I18n.t :by, {locale: lang.to_sym}
end

## I18n.tのオプションを省略するパターン
%w[en ja].map do |lang|
  I18n.locale = lang.to_sym
  puts I18n.t :for
end
```

### 実行してみる

できた。`I18n.t` 単品で呼ぶ形式が楽だ。

```
$ bundle exec ruby app.rb 
of the people
人民の
by the people
人民による
for the people
人民のための
```



## 毎回全部作ってらんない、Fallback

実際採用するとして、新しいメッセージを毎度全部書いていくのは面倒だ。
例えば次のように、歯抜けがある場合を試してみた。

```locales/ja.yml
---
ja:
  of: 人民の
  by: 人民による
  # for: 人民のための
```

この状態だとlocale=jaのとき`:for`の出力は`translation missing: ja.for`となった。
そういうのはデフォルト言語側を出す方がまだましだ。


### fallbackを仕込む

ソースとWikiを見てたらFallbackという機構があった。Backendに組み込んでみる。

```app2.rb
#!/usr/bin/env ruby
# coding: utf-8
require 'i18n'
require 'yaml'

I18n.enforce_available_locales = true
I18n.backend = I18n::Backend::Simple.new

## デフォルト言語にフォールバック https://github.com/svenfuchs/i18n/wiki/Fallbacks
I18n.backend.class.send(:include, I18n::Backend::Fallbacks)

I18n.backend.load_translations(Dir.glob('locales/*.yml'))

%w[en ja].map do |lang|
  puts I18n.backend.translate lang.to_sym, :of
end

I18n.backend.available_locales.map do |lang|
  puts I18n.t :by, {locale: lang.to_sym}
end



I18n.default_locale = :en # デフォルトでも:en

%w[en ja].map do |lang|
  I18n.locale = lang.to_sym
  puts I18n.translate :for
end
```


これで実行結果は

```
$ bundle exec ruby app.rb 
of the people
人民の
by the people
人民による
for the people
for the people
```

抜けはチェックツールで比較して、対応したくなったらすればよいというゆるポリシーのつもりなので、この挙動でいいや。


## 未対応ロケールはデフォルトを使う


`I18n.enforce_available_locales`をtrueでやっていると、言語セットを用意しているロケール以外ではプログラムを止める。
例えば`:de`だと `:de is not a valid locale (I18n::InvalidLocale)` と言われて止まっちゃう。

そういう手合もまあ、とりあえず`enforce_available_locales`をfalseにして、デフォルト出しときゃいいでしょ。


```app2.rb
#!/usr/bin/env ruby
# coding: utf-8
require 'i18n'
require 'yaml'

I18n.enforce_available_locales = false # 未対応言語でも止めない
I18n.backend = I18n::Backend::Simple.new

## デフォルト言語にフォールバック https://github.com/svenfuchs/i18n/wiki/Fallbacks
I18n.backend.class.send(:include, I18n::Backend::Fallbacks)

I18n.backend.load_translations(Dir.glob('locales/*.yml'))

%w[en ja].map do |lang|
  puts I18n.backend.translate lang.to_sym, :of
end

I18n.backend.available_locales.map do |lang|
  puts I18n.t :by, {locale: lang.to_sym}
end


I18n.default_locale = :en # デフォルトでも:en

%w[en ja de].map do |lang|
  I18n.locale = lang.to_sym
  puts I18n.translate :for
end
```

最後をlocale=deでも出してみたが、デフォルトにfallbackしてくれた。

```
$ bundle exec ruby app.rb 
of the people
人民の
by the people
人民による
for the people
for the people
for the people
```


## ロケールをどう判断したらいいか

実はこれを考えていなかった。ENV['LANG']でええんかいな？ Windowsもかな？
if else elseの予感がする。
