

ドメインそのものを表す Zone Apex (Naked DomainとかRoot Domainとかも) は[色々あって](https://tools.ietf.org/html/rfc1912#section-2.4)基本的にはAレコードでの運用です。

最近はNameServerとWebサイト(公開URL)を預けるプロバイダが同一ならば対応しやすくはなっていますが、次のようなケースでは悩みどころです。


- ゾーンは自分の管理下
- Webサイトはホスト名を払い出すタイプ(CNAME推奨)のどこかのサービスを利用

この組み合わせでZone apexを利用できるNameServerプロバイダを並べてみます。 ほか、ここでもできるよーという情報があれば追記するのでコメントしてもらえると嬉しいです。



## 国内

新鋭のサービス中心、って感じですね。

### [Dozens（ダズンズ）](http://dozens.jp/)

- [DNSレコード設定の手順 - Dozens ヘルプセンター](http://help.dozens.jp/post/help/help_03/)
  - `ALIASレコード` を参照

### [Gehirn DNS](https://www.gehirn.jp/gis/dns.html)

- [ALIAS（エイリアス）レコードの追加 | ゲヒルンサポートセンター](https://support.gehirn.jp/manual/gis/dns/add-aliasrecord/)


## 海外

特に区別する必要もない気はしますがなんとなく。

### [DNSimple](https://dnsimple.com/)

- [ALIAS Records - DNSimple Help](https://support.dnsimple.com/articles/alias-record/)
- 設定例： [Pointing the Domain Apex to Heroku - DNSimple Help](https://support.dnsimple.com/articles/domain-apex-heroku/)
  - heroku向けの案内ですが、どこでも一緒です。

ここ、個人的にしばらく使っていましたね。


### [DNS Made Easy](https://dnsmadeeasy.com/)

ここのは`ANAME`ですね。

- [ANAME Records | DNS Made Easy](https://dnsmadeeasy.com/services/anamerecords/)


### [easyDNS](https://www.easydns.com/)

こちらも`ANAME`ですね。

- [ANAME Records - easyDNS](https://fusion.easydns.com/index.php?/Knowledgebase/Article/View/190/7/aname-records/)



## その他、やや特殊・制限つき

### [Cloudflare](https://www.cloudflare.com/)

CNAMEを設定しても、基本的にAに展開してあつかう、という機能でApexに他のホスト名を割り当てられます。(alias of 〜 などと表記される)

- [CNAME Flattening: RFC-compliant support for CNAME at the root – Cloudflare Support](https://support.cloudflare.com/hc/en-us/articles/200169056-CNAME-Flattening-RFC-compliant-support-for-CNAME-at-the-root)

もともとCDNの補佐的位置で、ApexにCNAMEをつかうと他のレコードとの兼ね合いで挙動が怪しい、などとどこかで見たのでこの位置。一応自分も使ってます。


### Route53とCloudFrontでそれぞれAWSアカウントが違うケース

サービスを申し込んだらCloudFrontで配信してくれた、自分のドメインのApexを割り当てたい。というケース。
アカウントが同一ならプルダウンで選べるけども、別だとさすがに一覧に出て来ません。一応AWSのリソースならApexにALIASを直接指定すれば使えます。

- [エイリアス先 - Amazon Route 53](http://docs.aws.amazon.com/ja_jp/Route53/latest/DeveloperGuide/resource-record-sets-values-alias.html##rrsets-values-alias-alias-target)
  - `Amazon Route 53 ホストゾーンとディストリビューションを作成する際に異なるアカウントを使用している場合`  を参照

ELBなんかも同様。


### [PointDNS](https://pointhq.com/) (with heoku?)

[herokuのアドオン経由でALIASを設定](https://devcenter.heroku.com/articles/pointdns) できるようなんだけど、PointDNS側に何のドキュメントもない。
APIでもALIASというタイプのレコードを作れるように見えないのでheroku限定かも。
