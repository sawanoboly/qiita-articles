<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Chef-Server12(RC2)の配布がはじまったので内容をチェックしてみた。 [Chef Releases Chef 12 to Power DevOps Practices in the Enterprise | Chef Blog](http://www.getchef.com/blog/2014/09/08/chef-releases-chef-12-to-power-devops-practices-in-the-enterprise/ "Chef Releases Chef 12 to Power DevOps Practices in the Enterprise | Chef Blog")

![Chef_Server_12___Chef_Downloads___Chef.png](https://qiita-image-store.s3.amazonaws.com/0/7454/96e06620-a56c-69a0-66bd-3440fba27996.png "Chef_Server_12___Chef_Downloads___Chef.png")


今までEnterprise Chefとして提供されていたものが、ほぼそのままOSS版12になりました。
OrgnaizationやUserもそのままなので、1台あれば対象システムの管理区分けもできる。


## 構築環境

- Amazon EC2のUbuntu14.04
    - ↑Ubuntu12用のパッケージで問題なく動く


## Web管理の大幅変更 

### Management Console(WebのUI)はアドオンに。

[![Chef_Manage___Chef_Downloads___Chef.png](https://qiita-image-store.s3.amazonaws.com/0/7454/535a6d4d-7d36-b311-7e01-35c6418b347b.png "Chef_Manage___Chef_Downloads___Chef.png")](http://downloads.getchef.com/chef-manage/)



従来のWeb-UIは廃止になり、アドオンパッケージのインストールになる。Chef-Serverのコアと切り離して使いやすい。

パッケージは上図のリンクからダウンロードして、あとは次のリンクに従って導入。

- [Chef Manage — Chef Docs](http://docs.getchef.com/manager.html "Chef Manage — Chef Docs")
- [Install Chef Manage — Chef Docs](http://docs.getchef.com/install_manager.html "Install Chef Manage — Chef Docs")

操作コマンドが`opscode-manage-ctl`のままなのは置いておこう。

### Reporting

[![Reporting___Chef_Downloads___Chef.png](https://qiita-image-store.s3.amazonaws.com/0/7454/39fcd446-808f-33f1-8b3a-3ae6b280b383.png "Reporting___Chef_Downloads___Chef.png")](http://downloads.getchef.com/reporting/)

パッケージは上の図リンクから、インストールなどは次を参照。

- [Reporting — Chef Docs](http://docs.getchef.com/reporting.html "Reporting — Chef Docs")
- [Install Reporting — Chef Docs](http://docs.getchef.com/install_reporting.html "Install Reporting — Chef Docs")


レポーティングが有効になったManagement Consoleを確認するため、ちょこちょことChef-Clientを実行してみる。

![Chef_Manage_と_lw1-chef-server_—_bash_—_120×30.png](https://qiita-image-store.s3.amazonaws.com/0/7454/e5c660d3-f83b-f77f-ac6b-2f5753eb0b2b.png "Chef_Manage_と_lw1-chef-server_—_bash_—_120×30.png")

ナイスレポート、ヒストリーも悪くない。

![Chef_Manage.png](https://qiita-image-store.s3.amazonaws.com/0/7454/30b89316-9610-1a24-9ce7-afd949347f51.png "Chef_Manage.png")


## CLI拡張と非同期のPush型Job

とりあえずジョブキューみたいなことができる(サーバいるけど)Push jobsも解禁になった。
これも適当に分散してしまって大丈夫なコンポーネントだ。

[![Push_Jobs_Server___Chef_Downloads___Chef.png](https://qiita-image-store.s3.amazonaws.com/0/7454/86c11408-108e-d407-07c5-6938dd88ee1d.png "Push_Jobs_Server___Chef_Downloads___Chef.png")](http://downloads.getchef.com/push-jobs-server/)

パッケージは上の図リンクから、インストールなどは次を参照。

- [Push Jobs — Chef Docs](http://docs.getchef.com/push_jobs.html "Push Jobs — Chef Docs")
- [Install Push Jobs — Chef Docs](http://docs.getchef.com/install_push_jobs.html "Install Push Jobs — Chef Docs")


ただ、RC2の現状インストールにはコツがいる。

- リンクからダウンロードできる`opscode-push-jobs-server_1.1.1-1_amd64.deb`は、`private-chef`パッケージに依存しているので`--force`がいる。

- 管理用の`opscode-push-jobs-server-ctl`にもパスが通らないので、ダイレクトに叩く。
    - `/opt/opscode-push-jobs-server/bin/opscode-push-jobs-server-ctl reconfigure`

- クライアント(ジョブを購読して実行する側)のパッケージは案内が無い
    - ただ推測可能になっているので探す
- knifeコマンドの拡張は`knife-push`gemを入れる

以上のことを踏まえれば、強引に使える状態にできる。


### ジョブを登録

knifeから、ノードに実行してもらいたいジョブをpush-serverに登録する。
ジョブの名前はコマンドを直接でなく、事前登録済みのキーバリュー。設定に記載されていないジョブ名は`quorum_failed`となり実行されない。

```shell:
# knife job start chef-client ip-10-172-xxx-xxx.ap-northeast-1.compute.internal
Started.  Job ID: 6b4dcd19a746b97aecbb30a0906b7778
Running (1/1 in progress) ...
Complete.

command:     chef-client
created_at:  Tue, 09 Sep 2014 08:37:04 GMT
id:          6b4dcd19a746b97aecbb30a0906b7778
nodes:
  succeeded: ip-10-172-xxx-xxx.ap-northeast-1.compute.internal
run_timeout: 3600
status:      complete
updated_at:  Tue, 09 Sep 2014 08:37:12 GMT

```

ジョブIDから実行状況を確認したり、

```
# knife job status 6b4dcd19a746b97aecbb30a0906b7778
command:     chef-client
created_at:  Tue, 09 Sep 2014 08:37:04 GMT
id:          6b4dcd19a746b97aecbb30a0906b7778
nodes:
  succeeded: ip-10-172-xxx-xxx.ap-northeast-1.compute.internal
run_timeout: 3600
status:      complete
updated_at:  Tue, 09 Sep 2014 08:37:12 GMT
```

全体のジョブ一覧を確認したり。


```shell:
# knife job list
command:     date
created_at:  Tue, 09 Sep 2014 08:35:21 GMT
id:          6b4dcd19a7466b04fbe530a83bebd0c2
run_timeout: 3600
status:      quorum_failed
updated_at:  Tue, 09 Sep 2014 08:35:21 GMT

command:     chef-client
created_at:  Tue, 09 Sep 2014 08:37:04 GMT
id:          6b4dcd19a746b97aecbb30a0906b7778
run_timeout: 3600
status:      complete
updated_at:  Tue, 09 Sep 2014 08:37:12 GMT
```


## 追記： Analytics Platform

忘れていたので追記。

[![Analytics___Chef_Downloads___Chef.png](https://qiita-image-store.s3.amazonaws.com/0/7454/ea780abc-f539-77b1-8fc5-f04e67e1b897.png "Analytics___Chef_Downloads___Chef.png")](http://downloads.getchef.com/analytics/ubuntu/#/)

パッケージは上の図リンクから、インストールなどは次を参照。

[Install Chef Analytics — Chef Docs](http://docs.getchef.com/install_analytics.html "Install Chef Analytics — Chef Docs")

何が表示されるかはこちらが詳しい。
[Chef Actions — Chef Docs](http://docs.getchef.com/actions.html "Chef Actions — Chef Docs")


これはできれば別のサーバに入れましょう、通常のWebUIと競合しがち。



## まとめ

OSS版はもはや結構しょぼかったので、これらの機能に対する敷居を下げて無償にすることでClient/Server構成の価値が少し回復しました。
個人的にはもう一年くらい早かったらよかったのにと思いました。

