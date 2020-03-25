
[Kitchen::Driver::Sakuracloud](https://github.com/higanworks/kitchen-driver-sakuracloud)がひっそりと公開されている。


[Test-Kitchen](http://kitchen.ci) のバージョン1.4以降ではVMの調達に関連する処理を、独立したモジュールとして扱えるようになりました。

- Test-Kitchenについては： [test-kitchenのつかいかた](http://qiita.com/sawanoboly/items/9f560bd63ad0712b17ba)

先日、所用で作業用のDebianサーバがそこそこの性能で1つ必要になったので、さくらのクラウドに一瞬だけ上げようと思いました。
コンパネ開くのが面倒くさかったのでTest-Kitchenのドライバを書くことにしました。


## Test-KitchenのDriver(VMの作成・削除を担当)

v1.4以降のDriverは、(調達したサーバへの接続方法がSSH or WinRMに限れば)

- 作る時: 作って、ID(クラウドベンダ側で発行)とSSH or WinRMで接続するためのアドレスをstateに入れる
- 消す時: IDを控えてあるので、それに従って削除する


今回作成したぶんで、該当するのはこの辺。


```ruby:lib/kitchen/driver/sakuracloud.rb
...

      def create(state)
        server = create_server
        state[:id] = server.id
        state[:hostname] = server.interfaces.first["IPAddress"]
      end

      def destroy(state)
        destroy_server(state[:id]) if state[:id]
      end

...

```

Driverの役割ってこれだけです。[fogにしてある](https://github.com/fog/fog-sakuracloud)から、こういう時に楽ちん。


## 使う時

`SAKURACLOUD_API_TOKEN`と`SAKURACLOUD_API_TOKEN_SECRET`は環境変数にあれば勝手に使うので、このようにYAMLに書いてしまう。

```yaml:.kitchen.yml
---
driver:
  name: sakuracloud
  sshkey_id: <%= ENV['SAKURA_SSHKEY_ID'] %>
  api_zone: tk1a

transport:
  username: root
  ssh_key: <%= ENV['SAKURA_SSHKEY_PATH'] %>

provisioner:
  name: shell

verifier:
  name: shell

platforms:
  - name: debian8
    driver:
      sourcearchive: 112700890487
      serverplan: 4004
      diskplan: 4
      size_mb: 20480
  - name: ubuntu14
    driver:
      serverplan: 4004
      diskplan: 4
      size_mb: 20480
    transport:
      username: ubuntu


suites:
  - name: default
    run_list:
    attributes:
```

`provisioner=>shell`、はカレントにあるbootstrap.shを転送して叩く。
`verifier =>shell`、は省略するとローカルの`/bin/true`で終わる。
とりあえずログイン状態で対話的に試したい事があったので、これでOK。


listをみれば、Driverに`Sakuracloud`となっているのがわかる。

```
$ kitchen list
Instance          Driver       Provisioner  Verifier  Transport  Last Action
default-debian8   Sakuracloud  Shell        Shell     Ssh        <Not Created>
default-ubuntu14  Sakuracloud  Shell        Shell     Ssh        <Not Created>
```

### VMを起動

そうそう、このときはglibcのリビルドをオプション変えつつ試したかったのです。なのでとりあえず準備だけしてくれればよいので、convergeで実行するスクリプトを置きます。

```bootstrap.sh
sudo mkdir -p /usr/local/deb/glibc
cd /usr/local/deb/glibc

sudo apt-get update && apt-get install -y \
cmake dpkg-dev bison libbison-dev ruby \
&& apt-get build-dep -y glibc --fix-missing \
&& apt-get source -y glibc
```




```
$ kitchen converge debian
-----> Starting Kitchen (v1.4.2)
-----> Creating <default-debian8>...
[fog][WARNING] Create Server
[fog][WARNING] Create Volume
[fog][WARNING] Waiting disk until available
.[fog][WARNING] Modifing disk
       Finished creating <default-debian8> (1m31.15s).
-----> Converging <default-debian8>...
       Preparing files for transfer
       Preparing script
       [SSH] connection failed, retrying in 1 seconds (#<Timeout::Error: execution expired>)
       Transferring files to <default-debian8>
Get:1 http://security.debian.org jessie/updates InRelease [63.1 kB]
Get:2 http://security.debian.org jessie/updates/main Sources [109 kB]          
Ign http://dennou-h.gfd-dennou.org jessie InRelease     

....

-----> Kitchen is finished. (2m58.76s)
```

ではログインして色々と試す。

```
$ kitchen login debian

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

...

root@localhost:~#
```

### VMを廃棄

じゃあ消そう。

```
$ kitchen destroy debian
-----> Starting Kitchen (v1.4.2)
-----> Destroying <default-debian8>...
       Finished destroying <default-debian8> (0m19.32s).
-----> Kitchen is finished. (0m19.40s)
```

はい消えたー。

普通に各種Provisionerも使えます。
