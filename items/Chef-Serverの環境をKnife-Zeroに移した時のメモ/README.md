
古くなってきたあるサービスの環境を縮小することにしたので、Chef-Server11配下にあった環境を適当にKnife-Zeroに移した。

[knife-zero](http://higanworks.com/knife-zero/)。


## 準備段階

- 定期実行のChef-Clientを止めた。
    - Cronのやつをリソースのdeleteで。
    - Monit配下のデーモンにしてあるやつは`knife ssh`などで。
- Chef-Clientを最新に更新した。
    - `knife ssh`などで。
    - ※あとの`zero bootstrap`の時点でもやろうと思えばできます。


## リソースオブジェクトのダウンロード

`knife download`でChef-Serverから色々ダウンロード。

- environments, roles, data_bagsとnodesが対象

要はcookbooks以外は全部downloadしただけで使い回しできる。vaultを使ってないところだったので、clients, usersも省きました。


### knife.rbに追記その1

Knife-Zero用に`chef_repo_path`を追記した。

```.chef/knife.rb
chef_repo_path File.expand_path("../../", __FILE__)
```

この状態でもう、knifeに`-z`をつけたり外したりでServerとローカルChef-Repoの切り替えが効きます。


## Cookbookの集約

ファイルベースでChef-Zeroを使うと、Cookbook Versionを複数同時には扱えません。
この環境ではBerkshelfだったので、`berks vendor`で'./berks-cookbooks'に一旦集めました。

Chef-ServerにしかCookbookがない場合などは、一回それをバックエンド指定したBerksAPIを上げるといいかも。


### knife.rbに追記その2

集約したのでcookbook_pathを変更。もうChef-Serverにアップロードしないしね。

```
cookbook_path [ './berks-cookbooks' ]
```

幸いCookbookバージョン指定に関しては、更新前のチェックくらいにしか使ってなかったのと、混在させないように分岐で吸収しておいたので、機能が無くても大丈夫でした。
もし複数バージョンのCookbookを切り替えるならば、専用のBerksfileを用意してcookbook_pathを都度変更しかないかな。

## zero bootstrap (with `--no-converge`) でclient.rbだけ更新

まだノード側のclient.rbは次の状態

- まだChef-Serverを向いている。
- git管理向けのattributeホワイトリストを持ってない。


それらを解消するため、`knife zero bootstrap`を`--no-converge`オプションで実行し、client.rbだけを更新しました。


### knife.rbに追記その3

chef-clientを最新にしたのはホワイトリストのため。bootstrap再実行時には次のような設定を入れました。

```
knife[:automatic_attribute_whitelist] = %w[
fqdn
network
platform_build
virtualization
dmi
domain
ip6address
platform_version
platform
recipes
os
macaddress
chef_packages
hostname
roles
os_version
languages
ipaddress
platform_family
]
```

時間経過とともに変更があるファイルシステムなどのリソースを排除しました。そういうメトリクスはDatadogとかマカレルとかに上げりゃいいしね。

## zero convergeでローカルのnodeオブジェクトを更新

ホワイトリストが適用された状態のnodeオブジェクトにするため、一回convergeしておいた。roleやrun_listも保持されているのでサーバ側には変更もない。
さすがに`-W`のWhy-runで影響を事前に確認したけども、Search依存のレシピもちゃんと反映できて問題なかった。



### knife.rbに追記その4

仕上げにローカルモードをデフォルトへ。

```
local_mode true
```


最後にgitにコミットしておしまい。

