<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[Settingslogic](https://github.com/binarylogic/settingslogic) は設定ファイルを一回ERbで処理してから使えるステキなRubygem。


CIやherokuなどは`config.yml`を読んで環境変数を使いつつ、ローカル環境では`config.local.yml`で上書きしたいと思ったので試してみた。

ちなみに`*.local.yml`を使うという運用方はtest-kitchenから拝借。


```ruby:app.rb
require 'settingslogic'

class EnvSettings < Settingslogic
  source File.expand_path('../config.yml', __FILE__)
end

if File.exist?(File.expand_path('../config.local.yml', __FILE__))
  class EnvLocalSettings < Settingslogic
    source File.expand_path('../config.local.yml', __FILE__)
  end
  EnvSettings.merge!(EnvLocalSettings)
end

puts EnvSettings[:hoge]
```

多少冗長のような気もするが、既存のコードも`if`ブロックを一個追加で同じ対応ができるしOK。

とりあえず`config.yml`のみで実行。

```yaml:config.yml
hoge: <%= ENV['HOGEHOGE'] %>
```


```shell:
 $ HOGEHOGE=moge ruby ./app.rb 
moge
```

`config.local.yml`を置いて実行。

```yaml:config.local.yml
hoge: mogemoge
```

```
$ HOGEHOGE=moge ruby ./app.rb 
mogemoge
```

local側がENVを読んでもいいんだけど、使い勝手を考えるとこの順番かな。
