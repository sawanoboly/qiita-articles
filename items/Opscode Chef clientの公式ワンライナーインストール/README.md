<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

参照元：[http://www.opscode.com/chef/install/](http://www.opscode.com/chef/install/)



## 公式のinstall.sh

```
$ curl -k -L https://www.opscode.com/chef/install.sh | (sudo) bash
```

スクリプト内でプラットフォームを判別し、出来る範囲でインストールしてくれる。Rubyも組み込みで入っているのでとりあえずのchef-soloなどにちょうどいい。

もちろん`client.rb`とpemを用意すればchef-clientが動作する。


### cehf-serverを使っている場合

ホスティングでも自前でも、ServerがあるならBootstrapで作るほうがいい。

```
$ knife bootstrap FQDN (options)
```

bootstrapのテンプレートを見るとわかるが、プラットフォームによっては前述の`install.sh`が呼ばれることもある。
