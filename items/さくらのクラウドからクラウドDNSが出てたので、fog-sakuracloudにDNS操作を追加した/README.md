
[クラウドDNS | さくらのクラウドニュース](http://cloud-news.sakura.ad.jp/cloud_dns/)

クラウド共用ライブラリの[fog](http://fog.io)に、なんだったかのイベント向けで作ったさくらのクラウド向け実装[fog-sakuracloud](https://github.com/fog/fog-sakuracloud)というGemを趣味で保守してます。
(当時目線で)世界のクラウド見本市のようなfog対応プロバイダ一覧に、日本のが無いなーという程度の理由で追加したはず。

## fogでクラウドDNS

この記事で、fog-sakuracloudはv1.3.1です。クラウドDNSが無料のうちに追加しました。

### ゾーン一覧

クラウドDNSは必要な操作が少ないので、fogでまあまあ楽に管理できますね。

```Ruby:
require 'fog/sakuracloud'

## FogのDNSサービスを初期化
dns = Fog::DNS::SakuraCloud.new(
  :sakuracloud_api_token => 'YOUR_API_TOKEN',
  :sakuracloud_api_token_secret => 'YOUR_TOKEN_SECRET'
)

----

## ゾーン一覧を取得
> dns.zones

=> [  <Fog::DNS::SakuraCloud::Zone
    id="112700801016",
    name="sawanoboly4.net",
    description=nil,
    status={"Zone"=>"sawanoboly4.net", "NS"=>["ns1.gslb2.sakura.ne.jp", "ns2.gslb2.sakura.ne.jp"]},
    settings={"DNS"=>{"ResourceRecordSets"=>nil}},
    tags=[]
  >,
   <Fog::DNS::SakuraCloud::Zone
    id="112700801019",
    name="sawanoboly5.net",
    description=nil,
    status={"Zone"=>"sawanoboly5.net", "NS"=>["ns1.gslb2.sakura.ne.jp", "ns2.gslb2.sakura.ne.jp"]},
    settings={"DNS"=>{"ResourceRecordSets"=>nil}},
    tags=[]
  >]

```

(このstatusやsettingsのネスト、fogではわりと邪魔なので、ホンマに階層いるんかいなと思いつつ。。)ゾーン一覧が取れました。

### ゾーン単品取得して、更新

zonesはArrayなので、selectやらfindも使えます。一応getでID指定できるのでそれで単品を取得します。

```Ruby:
> zone = dns.zones.get('112700801016')
=>   <Fog::DNS::SakuraCloud::Zone
    id="112700801016",
    name="sawanoboly4.net",
    description=nil,
    status={"Zone"=>"sawanoboly4.net", "NS"=>["ns1.gslb2.sakura.ne.jp", "ns2.gslb2.sakura.ne.jp"]},
    settings={"DNS"=>{"ResourceRecordSets"=>nil}},
    tags=[]
  >
```

ZoneやNS、ResourceRecordSetsにはちょっと便利なゲッターが使えます。

```
> zone.zone
=> "sawanoboly4.net"

> zone.nameservers
=> ["ns1.gslb2.sakura.ne.jp", "ns2.gslb2.sakura.ne.jp"]
```

ではResourceRecordSetsを登録してみましょ。

```Ruby:
> zone.rr_sets = [{"Name"=>"mail", "Type"=>"A", "RData"=>"192.168.1.5"}]
=> [{"Name"=>"mail", "Type"=>"A", "RData"=>"192.168.1.5"}]

> zone.save
[fog][WARNING] Update DNS Zone 112700801016
=> true
```

設定が保存できているか確認します。

```Ruby:
> zone.reload # reloadはAPIから再取得。
=>   <Fog::DNS::SakuraCloud::Zone
    id="112700801016",
    name="sawanoboly4.net",
    description=nil,
    status={"Zone"=>"sawanoboly4.net", "NS"=>["ns1.gslb2.sakura.ne.jp", "ns2.gslb2.sakura.ne.jp"]},
    settings={"DNS"=>{"ResourceRecordSets"=>[{"Name"=>"mail", "Type"=>"A", "RData"=>"192.168.1.5"}]}},
    tags=[]
  >
```


rr_setの変更用ヘルパーは`Zone#rr_sets=`しか実装していないので、Arrayらしく操作するなら`settings['DNS']['ResourceRecordSets']`を直接触ってOK。

```Ruby:
> zone.settings['DNS']['ResourceRecordSets'] << {"Name"=>"www", "Type"=>"A", "RData"=>"192.168.1.2"}
=> [{"Name"=>"mail", "Type"=>"A", "RData"=>"192.168.1.5"}, {"Name"=>"www", "Type"=>"A", "RData"=>"192.168.1.2"}]

> zone.save
[fog][WARNING] Update DNS Zone 112700801016
=> true
```


その他の操作は[fogサイト](http://fog.io/about/provider_documentation.html)のDocumentationにまとめるようにしています。
