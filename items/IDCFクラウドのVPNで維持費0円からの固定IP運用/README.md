理由は様々ですが、アクセス元のIPアドレスを固定したいことがあります。

基本的にどこかに所属してオフィスで仕事、であればあまり気にしなくてよいのですが、それ以外の場所でも固定IPを利用する場合ではちょっとした環境のセットアップが必要です。

- オフィスまたは自宅に固定IP、VPNで一旦そこを経由
- 適当なIaaSクラウドサービスでVPSや固定IPを取得して踏み台作成

さて、固定IPアドレス、たいていは色々とケチっても1,000円弱くらいかかります。

それに加えて、拠点に用意する場合はレンタル機材やらVPNエンドポイントのメンテナンス、物理的引っ越しへの対応などが必要ですし、クラウドサービスを利用する場合は維持に費用がかかったり、何かの拍子にアドレスをリリースして変更を余儀なくされたり、ということもあるので、安定運用するにはそれなりの手間を掛ける必要がでてきます。


で、たまにしか固定IPを使わない私が色々検討して 、[IDCFクラウド](https://www.idcf.jp/cloud/)でいいやとなっています。

- [IDCFクラウド](https://www.idcf.jp/cloud/)

運用方針は以下の通り。

- デフォルトルータにVPN
- RootディスクのみのVirtualMachine(VM)をDeploy/Destroy
- Socksプロキシ / SSHのTCPフォワードでVM経由のNAPTでインターネットへ

これで、以下の金額でかつ、寝かせていてもアドレスを変更しなくてよい環境ができました。

- 使わない月は0円
- 使った月も上限500円 (500時間以上のVPN接続)
  - VM + Rootディスクの料金
  - ※ネットワークも一応無料枠があり、月あたり3TB以上をどこかに送信するなら超過分がかかります。

> 料金計算はこちら => [IDCFクラウド 料金表](https://www.idcf.jp/cloud/price.html)

運用方針をみて、ああなるほどと思った方はこの後の説明は特に必要ないかと思います。


## IDCFクラウドのサブネットとルータ(IPアドレス)

ここからは補足の解説。まずIDCFクラウドのアカウントでつかえるネットワークについて。

- 基本はプライベートサブネット
- インターネットアクセス用にデフォルトルータ(※表示上はIPアドレス)が標準搭載
  - プライベートサブネットのNAPT
  - VPN機能付きで、プライベートサブネットと相互通信可能
  - 有料で追加も可能


このルータがアカウント払い出し時に全リージョンに無料でついているのがポイントで、今回の用途に都合がよい特性は以下の2つ。

- アドレスが変わらない
- 放置でも無料

## ルータVPNと踏み台ホスト

ルータのVPN機能はL2TPで、クライアント側の設定によりすべてのトラフィックをVPN経由とすることが可能です。

しかし、さすがにサブネット以外へのアクセス(インターネット側など)は制限されています。

![idcf_fixip01.png](https://qiita-image-store.s3.amazonaws.com/0/7454/34198568-15a7-7039-91ae-330679d11aef.png)

そのかわりプライベートサブネットには直接アクセスできるので、VMを作り、それを経由すればルータの外側IPアドレスをアクセス元のIPとして利用することができます。

![idcf_fixip02.png](https://qiita-image-store.s3.amazonaws.com/0/7454/ae66f62d-f550-c1fe-9c0c-5e4106a5e64a.png)

## VM作成削除をスクリプトにしておく

流石にVPNをつなぐたびにWebのコントロールパネル(以下コンパネ)からVMを作成削除するのは面倒なので、スクリプトにしておきます。

CLIがあったので、これを使っています。

- [idcf/idcfcloud-cli: A CLI client for IDCF Cloud.](https://github.com/idcf/idcfcloud-cli)

導入後、`idcfcloud init`してAPIキーなどの設定をしておきましょう。

### VM作成

idcfcloud-cliはjsonでパラメータを渡します。

APIでつくると、コンパネでは不可能な**追加ボリュームなし**のVMを作成することができます。少しでもケチりたいので

VM作成には次のようなJSONを用意します。各種IDは、一つコンパネからVMを作成して`idcfcloud compute listVirtualMachines`で取れる値を使うのが簡単です。

`ipaddress`はDHCPまかせでなく、作成後にSSHのDynamicForwardなどをスクリプトにしておく際に都合がよくなるのでプライベートサブネット内のIPを指定します。

```json:instance.json
{
  "name": "fixed-ip-bastion",
  "zoneid": "ゾーンのID",
  "serviceofferingid": "VMサイズのID",
  "templateid": "OSイメージのID",
  "ipaddress": "10.xx.0.xxx",
  "keypair": "SSHキーの登録名",
  "startvm": true,
  "passwordenabled": false
}
```

JSONを用意したら、`idcfcloud compute deployVirtualMachine`でVMを作成しましょう。出力を再利用して破棄用のJSONもついでに作成。

```shell:deployVirtualMachine
$ idcfcloud compute deployVirtualMachine "`cat instance.json`" | jq '{id: .data.id, expunge: true}' > id.json
```

利用できるようになるまでは1分もかかりません。

### VM破棄

作成時に作っていおいたJSONをそのまま利用します。即破棄の`expunge: true`がないと、リストア待機で15分ほど同じIPのVMがつくれなくなります。今回用途では指定しておくほうが無難。

```json:id.json
{
  "id": "作成済みVMのIP",
  "expunge": true
}
```

固定IP利用の用事が済んだら`idcfcloud compute destroyVirtualMachine`で破棄しましょう。

```shell:destroyVirtualMachine
$ idcfcloud compute destroyVirtualMachine "`cat id.json`" && rm id.json
```

### VM起動、停止のみ

連日で何度もVPN接続・切断する場合は、VM作成の待ち(といっても数十秒ほどですが。。)が少々かったるいこともあるので、VMを維持しておいてもよいです。停止時はRootボリュームの料金がかかります。

その場合、`startVirtualMachine`, `stopVirtualMachine`で起動・停止です。

```
# VM起動
$ idcfcloud compute startVirtualMachine "`cat id.json`"

# VM停止
$ idcfcloud compute stopVirtualMachine "`cat id.json`"
```


## ShimoでVPN接続/切断時にHook

こうなったらVM作成・破棄をVPN接続のタイミングで自働化したくなります。

ちょうど自分はmacosがPPTPサポートを打ち切ったタイミングで有料のVPN管理ソフトウェアの[Shimo](https://www.shimovpn.com)を導入していたのでそれを利用します。

- [Shimo | VPN Client for Mac – for Everyone](https://www.shimovpn.com/)

![Shimo.png](https://qiita-image-store.s3.amazonaws.com/0/7454/e4fa4be2-aab3-cbe2-16f2-aa65dbe9c0f8.png)


これが割と便利で、以下の仕組みをGUIで設定・管理できるので買っといてよかったなあと思っています。(※無料のVPN管理ソフトウェアでも設定すればできるはずです。Shimoはそれが楽というだけで。)

- StaticRouteの add/remove
- connection時スクリプト
- 任意ドメインのLookup先NS指定

このような感じで登録しています。

![Shimo_actions.png](https://qiita-image-store.s3.amazonaws.com/0/7454/d5ae6cf3-eb60-421e-e9d9-0a71576cff99.png)


ただ、切断時スクリプトは状況によって必ず実行されるとは限らないので、残ることはあります。その時は手動で`destroyVirtualMachine`のスクリプトを実行すればよいですね。

## おわりに

メンテフリーでロケーションフリーで従量課金で(IDCFクラウドが健在である限り)アドレスが変わらない固定IP、だいたい良い感じです。

すべてのトラフィックをVPN経由にはできないことは、むしろup方向3TB以上で料金がかかることを考えると良い方向に作用しています。(つなげっぱなしを忘れてISOを上げまくるなどの事故防止の観点)

SOCKSに対応していないクライアントを使う場合はそれぞれにTCPForwardのSSHトンネルがいるのがちょっとだけ面倒ですが、極端な場合はVMの中からやってしまえばよさそう。
