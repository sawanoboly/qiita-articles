
HTTP/2のServer Pushって、Linkヘッダに対象のURIを入れたら[h2o](https://h2o.examp1e.net)とかがよろしくやってくれるそうだよ。

じゃあRails4.2でLinkヘッダを差し込んで見る。


## Precompiled Assets

さて、renderに応じて必要なassetsの実際のパスが必要だね。`Rails.application.assets_manifest`に格納されているみたい。

ちなみにフロント側のmrubyなどで現役Assetsのパスを拾いたい場合、パスがそれぞれ`.sprockets-manifest-(MD5ハッシュ).json`に書いてあるので、それを使うとよさそう。


## Linkヘッダの付与

とりあえずコントローラがこうだとします。

```
class ResourceController < ApplicationController

  def index
  end

end
```

やってみたアプリではフロントをRiotで書いたSPAにしてあるので、実際のコードも上記のまんまだったりします。

とくにいじってないRailsなので`application.css`,`application.js`に対応するファイルをpush対象とすればよさげ。


```
class ResourceController < ApplicationController
  before_action :prepare_link_header # for server push

  def index
  end

  protected

  def prepare_link_header
    push_keys = ['application.css', 'application.js']
    prefix = String.new('/assets/')
    push_paths = push_keys.map { |k| prefix + Rails.application.assets_manifest.assets[k] }

    response.headers["Link"] = push_paths.map{|p| "<#{p}>; rel=preload"}.join("\n")
  end

end
```

一旦全部コントローラに書いてみた。これで一旦動作を確認しよう。


![pushed.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/b735b894-f66e-e551-4246-b5ec9a7f9549.jpeg "pushed.jpg")


pushedきたわ。
