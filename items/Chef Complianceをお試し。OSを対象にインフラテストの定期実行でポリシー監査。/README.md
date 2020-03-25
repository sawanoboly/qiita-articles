
[Auditing and Compliance with Chef](https://www.chef.io/solutions/audit-compliance/)


![スクリーンショット_11_10_15__1_12_PM.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/fe8acd3a-b763-5051-60ab-af78e8486ec6.jpeg "スクリーンショット_11_10_15__1_12_PM.jpg")

先日Chef IncがVulcanoSecっちゅーところを買収し、Chef Complianceをリリースしました。うん触ってみよう。

## Chef Compliance

ざっとこんなツールです。

- OSのパッチ状況検出 (Linux/Windows)
- テストスイートInSpecのリモート実行
    - リモート系はSSHまたはWinRMを経由、エージェント不要
- レポート作成
- WebUIとAPI
- スケジュール

適当に作成したUbuntu14にプリセットの監査をかけると、SSH越しで色々と調べてレポートを作ってくれました。

![Chef_Compliance_Dashboard.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/1744c967-ea71-dc3e-ceea-b5b831c5f334.jpeg "Chef_Compliance_Dashboard.jpg")

なるほどねー。OSの自前管理を含むシステムを運用する場合の監査タスクに良さそう。


## インストール＆セットアップ

RPMまたはDEBがこちらで配布されています(0.9.1)。

https://downloads.chef.io/compliance/

> 注： ノードの数についてChef Incに質問してもらったら、RPM, DEBで配布しているローカルセットアップ版は、25ノードまでで使ってほしいなーとのことです。コード側では特に制限かかっては無さそうなので、自主的に問い合わせ系。
> 
> https://twitter.com/dai_lxr/status/664445935036928000



とりあえずVagrantのVMにセットアップしてみます。private_networkを当てておくとよいです。

セットアップは次の手順。これだけでHTTP/HTTPSの

```
$ dpkg -i chef-compliance_0.9.1-1_amd64.deb 
$ chef-compliance-ctl reconfigure
$ chef-compliance-ctl user-create [username] [password]
```

Chef-Serverやその他アドオンたちと同じですね。
nginx/postgresなども/opt以下にまとめて入るので、他のサービスが動いていないサーバにスタンドアローンで入れるのがよいです。

できたらHTTPSで開きます。ログインユーザはさっき作ったやつ。

## コンプライアンスプロファイルとInSpec

InSpecについてはまずここを一読。(単品で別のエントリは書くかも。)

[The Road to InSpec | Chef Blog](https://www.chef.io/blog/2015/11/04/the-road-to-inspec/ "The Road to InSpec | Chef Blog")

InSpecは単品でも使えますが、Chef Complianceのポリシーとして使うには、それぞれに影響度合いを数値でつけます。

プリセットから1つ拾ってみます。

- TCPのSYN cookieが有効だったら影響度1.0

なポリシー記述とテストケースは次のようになってます。

```sysctl_ipv4_spec.rb
...
control 'sysctl-ipv4-9.2' do
  impact 1.0
  title 'Enable TCP syn-cookies'
  desc '
    Avoid DoS TCP SYN attacks, by enabling SYN cookies.
  '
  describe kernel_parameter('net.ipv4.tcp_syncookies') do
    its(:value) { should eq 1 }
  end
end
...
```

UIではこんな感じで。

![スクリーンショット_11_10_15__1_24_PM.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/dba8fdd2-c6b9-395c-c5f2-8cd83421097d.jpeg "スクリーンショット_11_10_15__1_24_PM.jpg")

作成したポリシー(InSpecテストケース+重み付け)は、ZIPかTARのアーカイブにしてUIまたはAPIからChef Complianceに登録します。これをまとめる単位がプロファイルかな。


### セキュリティベースライン重視

Chef Complianceを触ってまずギョッとしたのが、Center for Internet Security(以下CIS)の配布するCIS Security Benchmarksを同梱していたこと。

[SECURE CONFIGURATION BENCHMARKS | CIS](https://benchmarks.cisecurity.org/ "Center for Internet Security")

PDFで案内される内容の一例がこちら。

> ## 2.2 Set nodev option for /tmp Partition (Scored)
> The nodev mount option specifies that the filesystem cannot contain special devices.

上記のような指針をspecに起こして収録されていました。

- CIS Ubuntu 14.04 LTS Server Benchmark Level 1
- CIS Ubuntu 14.04 LTS Server Benchmark Level 2

画面でみるとこちら。

![スクリーンショット_11_10_15__1_46_PM.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/3cbb656d-d286-b528-f8a4-db030ca1f687.jpeg "スクリーンショット_11_10_15__1_46_PM.jpg")

実際のポリシーファイル群。

```
$ tree /opt/chef-compliance/embedded/service/compliance-profiles/cis/cis-ubuntu-level1/
/opt/chef-compliance/embedded/service/compliance-profiles/cis/cis-ubuntu-level1/
├── metadata.rb
├── README.md
└── test
    ├── 01_patches_spec.rb
    ├── 02_filesystem_spec.rb
    ├── 03_secure_boot_spec.rb
    ├── 04_process_hardening_spec.rb
    ├── 05_os_services_spec.rb
    ├── 06_special_services_spec.rb
    ├── 07_network_firewall_spec.rb
    ├── 08_logging_auditing_spec.rb
    ├── 09_system_access_spec.rb
    ├── 10_user_accounts_spec.rb
    ├── 11_warning_banner_spec.rb
    ├── 12_file_system_permission_spec.rb
    └── 13_user_group_settings_spec.rb

1 directory, 15 files
```

おお、よくコードに起こしたなあ。。

## API

APIはありますが、Chef Docsではドキュメントがまだありません。

しかし [VulcanoSec API Reference](http://docs.vulcanosec.com/#api-reference)

現状のChef Complianceアプリケーションは[VulcanoSec](http://vulcanosec.com)のロゴ差し替え＆機能ちょっと減らした版なので、このドキュメントの内容がそのまま使えました。

Basic認証でトークンを発行して、期限内持ちまわるタイプ。
プロファイルを取るならこんな感じ。

```
$ curl -k -X POST https://127.0.0.1/api/oauth/token   -u testuser:testpass   -d "grant_type=client_credentials"
{"access_token":"y0IPr/YwUmKJJzKFfOTgd4gOYLQQrInh6wM7X9cl8YWgui02lutqFnJ16BGvnf4j4sbXZ/A2GecCwdmvF94kEg==","expires_in":16556,"token_type":"vulcanosec token"}


$ curl -k -X GET https://127.0.0.1/api/user/compliance -u "y0IPr/YwUmKJJzKFfOTgd4gOYLQQrInh6wM7X9cl8YWgui02lutqFnJ16BGvnf4j4sbXZ/A2GecCwdmvF94kEg==:" | jq .
{
  "cis": {
    "cis-ubuntu-level2": {
      "copyright_email": "support@chef.io",
      "copyright": "Chef Software, Inc.",
      "id": "cis-ubuntu-level2",
      "owner": "cis",
      "name": "cis/cis-ubuntu-level2",
      "title": "CIS Ubuntu 14.04 LTS Server Benchmark Level 2",
      "version": "1.0.1",
      "summary": "CIS Ubuntu 14.04 LTS Server Benchmark",
      "description": "# CIS Ubuntu 14.04 LTS Server Benchmark\n\ncopyright: 2015, Vulcano Security GmbH\nlicense: All rights reserved\n",
      "license": "Proprietary, All rights reserved"
    },
    "cis-ubuntu-level1": {
      "copyright_email": "support@chef.io",
      "copyright": "Chef Software, Inc.",
      "id": "cis-ubuntu-level1",
      "owner": "cis",
      "name": "cis/cis-ubuntu-level1",

...
```

認証がちょっと不便かな。。でもNodeの登録削除ができるので十分でしょう。

## おわりに

インフラにあるマネージドOSにテストをかけたいが、なにからやったらいいのかしら？ とか、ここは見とけっていうオススメテストケースが配布されていたらいいのになー、とかって割と聞く話です。
そういう場合に、こういったデフォルトでベースラインセキュリティの監査をタップリめにやってくれるプロダクトもアリですね。

もちろんテストケースのカスタマイズもすぐ書けますよね。Serverspecで慣れてるでしょう。
