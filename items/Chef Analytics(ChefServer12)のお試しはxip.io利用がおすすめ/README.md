<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


[Chef Analytics](https://www.getchef.com/blog/2014/07/15/meet-the-chef-analytics-platform/)はChefServer上のオブジェクト変更や、Chef-ManageやKnifeといったクライアント達が何をしたか見やすく記録する、情シス臭あふれるコンポーネントです。

![Chef_Analytics.png](https://qiita-image-store.s3.amazonaws.com/0/7454/a5b26475-6c97-547f-bdb1-23b0f47cf3de.png "Chef_Analytics.png")


基本はChef-Serverに到達可能な別の場所に構築すべし(Standalone)なのですが、既存Chef-Server12と同じサーバにインストールする(Combined)ことも可能です。

[Install Chef Analytics — Chef Docs](https://docs.getchef.com/install_analytics.html "Install Chef Analytics — Chef Docs")


ただしChef-Server12と共存させる場合、次の仕様になります。
> Chef-Server12がRC3の時点での仕様なので、このへんは変わるかもしれません。

- Chef-Server APIと同じIP/PortでListen
- ネームベースでChef-ServerAPIとChef Analyticsが振り分けられる
    - Servernameを一緒にしちゃうと、Chef Analyticsが優先

この仕様から、うかつにCombinedでインストールすると、Chef-Server本体のエンドポイントが乗っ取られる悲劇が待っています。


## 安全にCombinedでインストールしよう

関連パッケージの`chef-server-core`, `opscode-manage`, `opscode-analytics`はrpm/dpkgでインストール済みとします。

まずは公式の案内通りに`/etc/opscode/private-chef.rb`を編集します。

```/etc/opscode/private-chef.rb
dark_launch['actions'] = true
```


analyticsとインテグレーションするため、reconfigure時に`private-chef/recipes/actions.rb`レシピを呼びます。
※ 一応案内通り、デフォルトで`true`な気がするんだけど...

で、一回`chef-server-ctl reconfigure`しておきます。


```
# chef-server-ctl reconfigure

...

Chef Client finished, 31/384 resources updated in 19.674226684 seconds
opscode Reconfigured!
```

### ホスト名にxip.ioを利用しよう

次に`/etc/opscode-analytics/opscode-analytics.rb`を作成します。

ここで名前ベースのホスト名に、[xip.io](http://xip.io/ "xip.io: wildcard DNS for everyone")を使います。

```/etc/opscode-analytics/opscode-analytics.rb
analytics_fqdn "54.199.xx.xxx.xip.io"
```

ちなみに、このファイルもRubyで書いてOKなうえ、Ohaiのアトリビュートもそのまあ使えるので、次のように書いてもOK。

```/etc/opscode-analytics/opscode-analytics.rb
analytics_fqdn "#{node[:cloud_v2][:public_ipv4]}.xip.io"
```

IaaSクラウドでEIPとか付け外ししても`reconfigure`すれば済むのでおすすめ。

編集したら **opscode-analytics-ctl** で`reconfigure`します。

```
# opscode-analytics-ctl reconfigure

...


Chef Client finished, 104/114 resources updated in 36.144560204 seconds
opscode-analytics Reconfigured!
```

あとはパブリックIPとxip.ioでブラウザからアクセスすればChef Analyticsを使って見る事ができます。
xip.ioなので、オンプレミスでChef-Server12にプライベートIPしか付与してないという状態でも特に問題ないのはいいですね。

![Chef_Analytics.png](https://qiita-image-store.s3.amazonaws.com/0/7454/4ab317fc-0acc-bec7-a1be-d833d07df243.png "Chef_Analytics.png")

ログインはChef-Serverに登録済みのユーザで、関連付けられたOrgに関する情報を見ることができます。
Analyticsを有効にした後で色々と操作をしていくと、アクティビティが表示されていきます。

![Chef_Analytics.png](https://qiita-image-store.s3.amazonaws.com/0/7454/e408ad6f-1da4-ceed-56a8-e3c70762faef.png "Chef_Analytics.png")



止めたくなったら`opscode-analytics-ctl stop`で止めときましょう。
本格運用したくなったらStandAloneを検討するか、自ドメインのホスト名くらいは付けてあげたほうが良いかも。

## さいごに

Node内のChef-Clientの挙動はReporting, 操作ログ(オペログ)はChef Analyticsと役割が分かれていますね。

