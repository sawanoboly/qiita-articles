
Knife-Zeroを試してみようとして、Vagrantでconverge(chef_client)を挫折するのをよく見るので、手順を追ってみた。

> 追記: knife-zero 1.8でbootstrap時のattributeを自動追加するように変更した。

## 素のVagrant VM(VBox)を作成する。

とりあえず何も設定しないでVMを起動してみることにする。

```Vagrantfile
Vagrant.configure(2) do |config|
  config.vm.box = "opscode-ubuntu-14.04"
end
```

up。

```
$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'opscode-ubuntu-14.04'...
....


 SSH address: 127.0.0.1:2201

...

```

途中でSSHの接続情報出てくるよね。後でも取れるけど。


## knife zero bootstrapする。

さっきのSSH情報でBootstrapはできる。

```
$ knife zero bootstrap 127.0.0.1 --ssh-port 2201 --ssh-user vagrant --node-name test -i ./.vagrant/machines/default/virtualbox/private_key --sudo

Doing old-style registration with the validation key at ...
Delete your validation key in order to use your user credentials instead

Connecting to 127.0.0.1
127.0.0.1 Installing Chef Client...
127.0.0.1 --2015-06-18 09:25:51--  https://www.opscode.com/chef/install.sh
127.0.0.1 Resolving www.opscode.com (www.opscode.com)... 166.78.227.233
127.0.0.1 Connecting to www.opscode.com (www.opscode.com)|166.78.227.233|:443... connected.
127.0.0.1 HTTP request sent, awaiting response... 200 OK
127.0.0.1 Length: 18736 (18K) [application/x-sh]
127.0.0.1 Saving to: ‘STDOUT’
127.0.0.1 
100%[======================================>] 18,736      --.-K/s   in 0.003s  
127.0.0.1 
127.0.0.1 2015-06-18 09:25:58 (6.14 MB/s) - written to stdout [18736/18736]
127.0.0.1 
127.0.0.1 Downloading Chef 12 for ubuntu...

...

127.0.0.1 Converging 0 resources
127.0.0.1 
127.0.0.1 Running handlers:
127.0.0.1 Running handlers complete
127.0.0.1 Chef Client finished, 0/0 resources updated in 6.751803888 seconds
```

うむ、`nodes/test.json`がつくられる。


## Nodeで自動取得したIPアドレス(10.0.〜とか)とBootstrapで使用(SSH接続できる先)したIPアドレス(127.0.0.1)が違う

たいていここで詰まる。
NWが少し入り組んだ環境ではゲートウェイに繋がるNICとSSHの接続に使うIPアドレスが異なることもあるので、knife-zeroではbootstrap時に仕込みを入れることにした。

`nodes/test.json`を見ると、Bootstrap時に使用した接続名がnodeのattributeとして保持されている。


```nodes/test.json
{
  "name": "test",
  "normal": {
    "knife_zero": {
      "host": "127.0.0.1"
    },
    "tags": [

    ]
  },
  "automatic": {

-- 以下省略
```


## 接続用のIPを`--attribute`で渡してconverge

保持された接続先を指定するには、`--attribute knife_zero.host`を渡してconverge(chef_client)でOK。


```
$ knife zero converge name:test --attribute knife_zero.host --ssh-port 2201 --ssh-user vagrant  -i ./.vagrant/machines/default/virtualbox/private_key
127.0.0.1 Starting Chef Client, version 12.3.0
127.0.0.1 resolving cookbooks for run list: []
127.0.0.1 Synchronizing Cookbooks:
127.0.0.1 Compiling Cookbooks...
127.0.0.1 [2015-06-18T09:33:49+00:00] WARN: Node test has an empty run list.
127.0.0.1 Converging 0 resources
127.0.0.1 
127.0.0.1 Running handlers:
127.0.0.1 Running handlers complete
127.0.0.1 Chef Client finished, 0/0 resources updated in 6.403417422 seconds
```

これは`knife.rb`に`knife[:ssh_attribute] = 'knife_zero.host'`として省略もできます。
Vagrantでknife-zeroを"試してみるだけ"ならこれで十分。

----

複数のVMをつかう場合や、private_network/hostonly_networkを使うならREADMEに説明を書いています。
