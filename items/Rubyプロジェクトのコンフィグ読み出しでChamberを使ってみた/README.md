<!-- permanent -->

[Settingslogicを使ってconfig.yml, config.local.yml運用してみる - Qiita](http://qiita.com/sawanoboly/items/1d776006ed24513e1b55 "Settingslogicを使ってconfig.yml, config.local.yml運用してみる - Qiita") の続編で、[binarylogic/settingslogic](https://github.com/binarylogic/settingslogic "binarylogic/settingslogic")から[m5rk/chamber](https://github.com/m5rk/chamber "m5rk/chamber")に乗り換え。


## こうしたかった

- 設定はYamlに書きたかった
- CIやPaaS(Heroku)にフィットさせるため、環境変数と併用したかった
  - ChamberはSettingsLogicと同じように、Yamlを一度ERbで処理する
- hoge.yml, hoge.local.ymlとあったら、hoge.local.ymlを優先するようにマージしたかった
  - ローカル作業時はわざわざ環境変数を変更するのが面倒なことが多い
  - test-kitchenの設定の持ち方を参考にした
  - hoge.ymlは基本的にERbのENVから取るので、センシティブな値を除去してリポジトリにADDできるのが嬉しい。



## このように書いた

```
require 'chamber'

env_confs = [
  File.expand_path('../../config/environment.yml', __FILE__),
  File.expand_path('../../config/environment.local.yml', __FILE__)
]
EnvSettings = Chamber.load files: env_confs


app_confs = [
  File.expand_path('../../config/app.yml', __FILE__),
  File.expand_path('../../config/app.local.yml', __FILE__)
]
AppSettings = Chamber.load files: app_confs
```

まあまあスッキリした。
