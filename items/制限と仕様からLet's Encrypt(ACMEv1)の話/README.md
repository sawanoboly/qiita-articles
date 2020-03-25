
現行のACMEv1を使ったLet's Encryptのお話。

([https://letsencrypt.org](https://letsencrypt.org))

> V1は終わりましたが、V2でも概ね同じです。一応V2はひとつ制限が追加されてます、追記の3を参照。

個人が手持ちのドメインで利用するにはあまり気にすることもないですが、何度も証明書を発行しようとすると制限に引っかかってくることがあります。

- https://letsencrypt.org/docs/rate-limits/

先日[Encryptを少し多めにLet'sした機会](https://tech.pepabo.com/amp/2017/08/22/lolipop-free-ssl/)があったので、その時に色々気を使ったことをまとめておきます。


## Let's Encryptにかかる制限(rate-limits)

といっても、(ドメインの所有さえ確認できれば)ACMEの仕様としてかかる制限はありません。
ほとんどはACMEのプロバイダによる、証明書の発行やそれにまつわるリクエストへの量的な制限となります。


例えば、Let's Encryptが証明書を管理するAPIの実装、[boulder](https://github.com/letsencrypt/boulder)では、次のように設定ファイルで制限ルールを管理しています。

[boulder/test/rate-limit-policies.yml](https://github.com/letsencrypt/boulder/blob/master/test/rate-limit-policies.yml)

ほとんどが `期間(window)` X `回数による閾値(threshold)` です。

ちなみにこちら、実際にLet's Encryptの本番(有効なCA)で利用されている設定値とは違います。実際の値を確認することはできないので、公表されている文書から判断しましょう。
制限はわりとよく変更されていますので、この記事の内容は公開時点の制限がベースになります。

以下、 **Let's EncryptはLEと略して表記** します。

## 制限のかかるドメインの単位 - public suffix

LEでは、

- 同じ`ドメイン`での証明書発行は週あたり20証明書まで

という制限があります。さて、このドメインとはどこまでを対象とするのか。

たとえばFQDNではラベルを複数つけて、`www.sawanoboly.net`とか`xxx.yyy.sawanoboly.net`とか、それなりに長いドメイン(いわゆるサブドメイン)を付与できて、それぞれにLEで証明書を発行することができます。

LEではこの判断に、Mozilla主導の **public suffix** をつかっているようです。

- https://publicsuffix.org

**public suffix** の詳細は省きます。(私があまり詳しくない)

LEでは、rate-limits対象の割り出しと、そのドメインが個人の持ちものとしておかしくないか？ という判断基準のような使いかたをしているという感じで理解しておけば十分そうです。
さらに大雑把に言うとレジストラ直轄のドメイン部分までが対象ってとこでしょうか。


**public suffix** を利用するためのライブラリもあります。

- [Goのライブラリ | weppos/publicsuffix-go](https://github.com/weppos/publicsuffix-go)
- [Rubyのライブラリ | weppos/publicsuffix-ruby](https://github.com/weppos/publicsuffix-ruby)

ちょっとRubyのやつを使ってみましょう。


```ruby
pry> require 'public_suffix'

pry> PublicSuffix.domain('sawanoboly.net')
=> "sawanoboly.net"

pry> PublicSuffix.domain('www.sawanoboly.net')
=> "sawanoboly.net"

pry> PublicSuffix.domain('sub.www.sawanoboly.net')
=> "sawanoboly.net"

pry> PublicSuffix.domain('kosub.sub.www.sawanoboly.net')
=> "sawanoboly.net"
```

この例では`sawanoboly.net`が **public suffix** なので、サブドメインの証明書を含めて`sawanoboly.net`が週あたり20件のrate-limits対象です。


### 余談: Not Public のドメインを判断する

PublicSuffixのライブラリは『こんなもん認証するまでもねーだろ』っていう申請を蹴るのにつかえたりします。

たとえばAmazonEC2でインスタンスに払い出されるFQDNは`Not Public`として判断されます。

```ruby
pry> PublicSuffix.domain('ec2-34-193-82-xxx.compute-1.amazonaws.com')
=> nil
```

Rubyライブラリだと、`#parse`を使うと例外を出してくれます。

```ruby
pry> PublicSuffix.parse('ec2-34-193-82-xxx.compute-1.amazonaws.com')
PublicSuffix::DomainNotAllowed: `ec2-34-193-82-xxx.compute-1.amazonaws.com` is not allowed according to Registry policy
```

対象のリストはちょくちょく人力でアップデートしています。

- https://github.com/publicsuffix/list/commits/master

最新リストの追従度はgoのライブラリの方が自動でやっているぶん良い感じです。


### PublicSuffixとrate-limits

ちなみにこのドメインであとどのくらい証明書を作成できるのか？という、要はrate-limitsを問い合わせるインターフェースはありません。
後述するクライアント識別のゆるやかさ維持にも関わってくるので、そんな野暮ったい機能はつけないのかもしれません。

何かしらシステム化した証明書管理を行いつつ、rate-limitsを事前に知りたい場合は **public suffix** をベースに発行履歴を残しておき、履歴を元に計算するほかなさそうです。


## Common Name(CN)とSubject Alternative Name(s)(SANs)

前述の『週あたり20証明書』までという制限では、サブドメインを沢山使用している環境では一斉に証明書を発行できなくて不便です。
LEの証明書はCSRにSANsを入れて、`sawanoboly.net`と`www.sawanoboly.net`を1つの証明書で両方有効にするやつです。通称で2wayとかも呼ばれるやつですね。

LEではFQDN100個まで証明書に含めてOKとしています。個別に発行して20件の制限にかかることがないよう、積極的につかうと良いと思います。


### SANsも個別にドメイン所有の確認(Authorizations)は必須

ACMEにおけるDNSやHTTPによるドメイン所有の確認(以降Authorizationsまたはチャレンジと呼びます。)はCN、SANsとわず全てのFQDNに対して必要です。
CNさえチャレンジとおればSANs付け放題、ということはないので地道に`.well-known/acme-challenge/`しましょう。


### CSRに含めるSANsは99件? 100件？

LEでは一度の証明書に含めるFQDNの上限を100件としています。
これ、CSR視点ではぱっと見では迷うんですよね。

- CN + SANs で合計100？
- SANsに100入れていいの？

こちら、LEのboulderではCSRでのCNもSANsに入れてから証明書を発行するので、申請に使用できるCSRはCNと違うFQDNのSANsを99件まで(※)です。

boulderでは設定によってCNをSANsに含めるかを決めています。ソースを見たら次のような処理をしていました。


```go:boulder/csr/csr.go
...

func normalizeCSR(csr *x509.CertificateRequest, forceCNFromSAN bool) {
	if forceCNFromSAN && csr.Subject.CommonName == "" {
		if len(csr.DNSNames) > 0 {
			csr.Subject.CommonName = csr.DNSNames[0]
		}
	} else if csr.Subject.CommonName != "" {
		csr.DNSNames = append(csr.DNSNames, csr.Subject.CommonName)
	}
	csr.Subject.CommonName = strings.ToLower(csr.Subject.CommonName)
	csr.DNSNames = core.UniqueLowerNames(csr.DNSNames)
}

...
```

発行された証明書を確認してみるとよいでしょう。CNのFQDNはSANsにも入っているはずです。

> ※) 実際に運用されているboulderでmaxNamesのパラメータが100であれば。

あと、全く同じSANsの組み合わせでは週あたり5件という制限もあり、これはテストで作成する時とかに引っかかることがあるかな？というくらいです。一応SANsの組み合わせを変えれば無視(※20の方のルール適用)できます。


### 理論上はドメインあたり2万超のFQDNをTLS保護が可能

これもほぼ余談になります。これまで紹介した証明書発行数の制限を鑑みますと、

- 1週間あたり20の証明書が発行できる。
- それぞれSANsとして100のFQDNを含めることができる。
- 証明書は3ヶ月有効。

12週で申請をローテーションすれば、2,000(週あたり) x 12週分で24,000のFQDNを有効な状態に保つことができる気がします。

2018年にはACMEv2・ワイルドカード対応がありますが、気合い次第では手持ちサブドメインの全対応も大抵の環境では可能ですね。


## Authorizations(チャレンジ)の制限 - Pending放置ダメ絶対

ドメイン所有確認のAuthorizationsは、以下のフローでおこないます。まあ大体知っているかと思います。

1. LEにFQDNを申請する => 認証に必要な情報を受け取る
2. 認証に必要な情報を元にセットアップ
3. **認証実行** を申請する
4. LEが認証してみた結果。。。
  - 成功で`valid`
  - 失敗で`invalid`、 **同じAuthorizationのリトライは不可能**

ここで2の状態(`Pending`)で放置してはダメです。たとえばスクリプトにしているとして、2の処理中に例外で落ちるとかのケースで。
この`Pending`状態にあるAuthorizationは週あたり300までという制限があります。

> (昔はもうちょっと回数少なかった覚えがあります、5件くらい貯めたらアウトだったような...)

この制限が曲者なのは`Pending`から他の状態に遷移する手段が **認証実行** を依頼する以外にないことです。
キャンセルは仕様になく、今自分がいくつのAuthorizationを申請したかを取得するインターフェースもありません。


なので、1のFQDN申請をした場合、セットアップに失敗したとしてもステータスを `Pending` => `valid/invalid` に進めるため、3の **認証実行** までは必ず行いましょう。


### Authorizationsちょっと補足

この制限300は多いような気がしますが、`Pending`は放っておくと1週間カウントされるので、SANsで数件ずつとかの処理をする場合にクライアントの処理がまずいといつのまにか到達している数です。

複数のSANsを含む証明書を作る場合、それぞれのFQDNでAuthorizationを **認証実行** までいくようにループするのをお勧めしています。
Authorization件数分作成してまとめて **認証実行** だと、やはりなんらかの異常があった時にまとめて`Pending`が残ってしまいます。

なお、このPending Limitのカウントは申請時の認証に使うアカウントが基準です。
つまりアカウントを新しく作ってしまえばとりあえずなかったことにもできるので。緊急の手段として覚えておくとよいかも。


### 検証用にチャレンジ無視したboulderを使いたい

LEのboulderはソースが丸ごとGithubで公開されているので、各自の手元などで簡単に動かすことができます。
そのまま使用すればCAがダミーなだけのLEテスト環境として開発用途などにも使えますし、CA証明書と秘密鍵を差し替えればACME対応プロバイダにすぐなれます。

で、せっかく手元で動かすのだから、Authorizationsをやらずに証明書発行してしまいたいこともあります。
そういうときはva(ValidationAuthorityなのかな?)にちょいと手をいれるとノーチャレンジ(実際はチャレンジ失敗を無視)のboulderを稼働できます。

```go:boulder/va/va.go
...

	// Check for malformed ValidationRecords
	if !challenge.RecordsSane() && prob == nil {
		prob = probs.ServerInternal("Records for validation failed sanity check")
	}

	// 問題を無かったことにする
	prob = nil

...
```

これでじゃんじゃんと好きなFQDNで証明書を作ることができるので、LEを大規模に扱うサービスの開発が捗るかも。
また、組織用CAによる証明書をACMEで発行しまくる、などの用途にもアリかなと思ってます。


## クライアントの鍵(≒アカウント)の取り扱い

ここは制限というよりは、むしろゆるいよという話。

LEのアカウントは秘密鍵を使って作成します。
通常の有料サービスでは作成したアカウントを大事に使い回すことになりますが、LEでは次のような仕様であり、その存在感はなかなか希薄です。

- メールアドレス被りOK
- ドメイン認証の申請はどのアカウントからでもOK
  - 証明書発行のrate-limitはあくまでドメイン(public suffix)がベース
- アカウントの申請履歴など、紐づいた情報などは全く取れない

もはやTOSに同意するためだけに存在するような気がします。

### なんでこんなに緩いのか

とにかくドメインの所有を確認できればOKという姿勢からくる仕様だと思います。
煩わしい手順の省略や、管理の手間の軽減につながり、個人的には大歓迎です。


例えば新規にサーバを構築するケースで、すでに何件か発行済みのドメイン配下のFQDNを割り当てるとします。
LEの証明書をHTTPの認証で発行して割り当てようとする際、もしアカウントとドメイン(public suffix)の紐付けが厳密な場合は次のいずれかの手段をとらなくてはいけません。

- アカウントの秘密鍵をサーバにアップロードして、certbotなどでAuthorizationを認証。
- 共有端末などでAuthorizationを作成、サーバに必要情報をアップロードしてドメイン認証 -> 証明書取得。
  - これには色々な手段があります。
  - (注) DNS認証なら、どちらかというとこの手順が正攻法です。

アカウントとドメインの紐付きがないため、実際にはサーバでcertbotを実行するだけで済みます(※)。サーバ上で作成したアカウントの秘密鍵はそのまま置いといても良いし、別に捨ててもよいので非常に楽なもんです。

※ もちろん上記の参考手段で同一のアカウントをつかってもよいです。

### Revokeとアカウント

アカウント秘密鍵のほぼ唯一といっていい再利用目的は証明書の失効(Revoke)です。
しかし、Revokeには次の2通りの手段があります。

- 証明書と、LE作成時のアカウント秘密鍵
- 証明書と、そのペアになる秘密鍵

後者がつかえるため、結局アカウント秘密鍵は管理不要です。

### 証明書の更新とアカウント

更新、といいますが、もともとX.509の証明書に期限変更みたいな仕様があるわけでもなく、新規作成と手順はなんら変わりません。

先にあげた次の仕様、

- ドメイン認証の申請はどのアカウントからでもOK

からして、結局は前回作成した時のアカウントはなんの関係もないままに新しい証明書を作成することができます。


### アカウント作成の制限 - 10 Accounts per IP Address per 3 hours

では、アカウントは証明書を作成するたびに毎回新規でよいのか？ というと、大量に作成する場合には注意が必要なこともあります。

- 同一のIPアドレスから、3時間あたり10アカウントまで作成可

この制限を避けるために、少なくともある程度の期間は使い回しをする必要があります。通常は秘密鍵をファイルに書き出しておけば良いので気にすることはないかもしれません。

しかし例えば、 **例外があったらとりあえずクライアントの作成からリトライしときゃいいっしょー** みたいな安直な処理をするとこの制限に引っかかってお祭りになったりするので気をつけようぜ！

...とにかく、LEな世界ではアカウントなんてとくに重要でなく、ドメインの所有だけが証明書発行への道なのですよということです。

アカウント関連はもうひとつ制限がありますが、よっぼどでないと引っかかるようなものでもないので割愛します。

## 終わりに

TLSで暗号化！ってよりは、HTTP/2に対応できるのが素敵だと思います。
エンドユーザのドメインをたくさんあつかうような事業者さんは、早速寄付しつつ全部のドメインでLEが使えるようにしていきましょう。

----

## <del>追記1: Google Safe Browsingに引っかかったらダメ。</del>

> 追記の追記: これは2019年2月くらいから制限されなくなっています。こういう視点でブロックするのはやっぱ駄目ということでしょう。

ドメイン所有確認の際、申請するドメインは一定のルールによって拒否されることがあります。

そのルールの1つとして、Google Safe Browsingによる`Unsafe`扱いされていないかがありました。

- [Google Safe Browsing  |  Google Developers](https://developers.google.com/safe-browsing/)


これに引っかかると次のようなレスポンスでAuthorizationがハネられます。

> "${domain}" was considered an unsafe domain by a third-party API


CLIの`sblookup`でドメインがどう扱われているか、手元で確認することができます。

```
$ echo 'http://sawanoboly.net' | sblookup -apikey=$APIKEY 2> /dev/null 
Safe URL: http://sawanoboly.net

$ echo 'http://malware.testing.google.test/testing/malware/' | sblookup -apikey=$APIKEY 2>/dev/null 
Unsafe URL: [{malware.testing.google.test/testing/malware/ {MALWARE ANY_PLATFORM URL}} {malware.testing.google.test/testing/malware/ {SOCIAL_ENGINEERING ANY_PLATFORM URL}}]
```

もちろんこういったポリシーに関しては[賛否両論ある](https://community.letsencrypt.org/t/certbot-not-able-to-issue-certificate-site-marked-as-unsafe-google-safe-browsing/33011/2)でますよね。


## 追記2: デコード出来ないPunycode風文字列はダメ

ドメインに`xn--`のプレフィクスをつけると、いわゆるPunycodeとなってUnicodeの文字を使うことができますが、これがデコード出来ないとboulderがエラーを返してきます。

> Error creating new authz :: DNS label contains malformed punycode

ただの文字列として扱ってはくれないということですね。

例えば、`日本語.jp`のPunycodeから一文字削ると不正になります。 (例はRubyのSimpleIDNを使用)

```ruby:
(pry)> SimpleIDN.to_unicode('xn--wgv71a119e.jp')
=> "日本語.jp"
(pry)> SimpleIDN.to_unicode('xn--wgv71a119.jp')
SimpleIDN::ConversionError: punycode_bad_input(1)
```

このケースでも証明書は発行できません。


## 追記3: ACMEv2で追加された制限

> 300 New Orders per account per 3 hours.

ひとつのアカウントで3時間あたり300件の新規証明書発行でLimitとなってます。
