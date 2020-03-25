<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


ChefServerがあると、Nodeの構成情報を管理・利用できる。

構成情報の利用方法は幾つかかあるがレシピ内で使う場合、Searchメソッドが使える。


## searchメソッド

任意のRoleに宛てられたノード達をArrayで返してもらう例。

```ruby
chef > search(:node, "roles:zookeeper_servers")
 => [node[zk01.example.com], node[zk02.example.com], node[zk03.example.com]] 
```

それぞれのアイテムは`Chef::Node`クラス。

```ruby
chef > search(:node, "roles: zookeeper_servers")[0].class
 => Chef::Node 
```

`Chef::Node`クラスということは、Attributesを持っているということなので、下記のようにレシピ内で利用したい情報を抽出できる。

```ruby
chef > search(:node, "roles: zookeeper_servers").map {|node| node.ipaddress}
 => ["192.168.1.10", "192.168.1.20", "192.168.1.45"] 
```

## Knifeから使う例

knife ssh はSearchAPIのクエリを使い、対象ノードにssh越しのコマンド実行が出来る。

```bash
$ knife ssh "roles:zookeeper_servers"  date
zk01.example.com Sun Apr  7 01:17:28 JST 2013
zk02.example.com Sun Apr  7 01:17:28 JST 2013
zk03.example.com Sun Apr  7 01:17:28 JST 2013

```

SearchはChefの奥義の一つだが、最近のChef対応を謳うサービスはSolo互換が主なので使えないのが残念。


