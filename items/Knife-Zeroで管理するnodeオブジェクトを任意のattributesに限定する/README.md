
Chef-Clientには[Whitelist Attributes](https://docs.chef.io/attributes.html#whitelist-attributes)という機能があって、Ohaiが収集した(Automatic)Attributesから、任意のAttributeのみ保存対象することができます。

使い方は`client.rb`に書けばOK。


## Knife Zero Bootstrap で普通にnodeオブジェクトを作成するとでかい

適当なAmazon EC2インスタンスにbootstrapを仕掛けてみます。

```shell:
$ knife zero bootstrap 52.69.40.215 -x ec2-user -i ~/.ssh/privatekey --hint ec2 -N test-server
Doing old-style registration with the validation key at ...
Delete your validation key in order to use your user credentials instead

Connecting to 52.69.40.215
52.69.40.215 -----> Installing Chef Omnibus (-v 12)
52.69.40.215 downloading https://www.opscode.com/chef/install.sh
52.69.40.215   to file /tmp/install.sh.2281/install.sh
52.69.40.215 trying wget...
52.69.40.215 Downloading Chef 12 for el...
52.69.40.215 downloading https://www.opscode.com/chef/metadata?v=12&prerelease=false&nightlies=false&p=el&pv=6&m=x86_64
52.69.40.215   to file /tmp/install.sh.2286/metadata.txt
52.69.40.215 trying wget...
52.69.40.215 url	https://opscode-omnibus-packages.s3.amazonaws.com/el/6/x86_64/chef-12.3.0-1.el6.x86_64.rpm
52.69.40.215 md5	c19fefcb3d033107e9fbdb3839312584
52.69.40.215 sha256	4b7c846a9ad93564cc203a5ac99890431f7d6ad159c424aa89827fd772c9881d
52.69.40.215 downloaded metadata file looks valid...
52.69.40.215 downloading https://opscode-omnibus-packages.s3.amazonaws.com/el/6/x86_64/chef-12.3.0-1.el6.x86_64.rpm
52.69.40.215   to file /tmp/install.sh.2286/chef-12.3.0-1.el6.x86_64.rpm
52.69.40.215 trying wget...
52.69.40.215 Comparing checksum with sha256sum...
52.69.40.215 Installing Chef 12
52.69.40.215 installing with rpm...
52.69.40.215 warning: /tmp/install.sh.2286/chef-12.3.0-1.el6.x86_64.rpm: Header V4 DSA/SHA1 Signature, key ID 83ef826a: NOKEY
52.69.40.215 Preparing...                          ################################# [100%]
52.69.40.215 Updating / installing...
52.69.40.215    1:chef-12.3.0-1.el6                ################################# [100%]
52.69.40.215 Thank you for installing Chef!
52.69.40.215 Starting first Chef Client run...
52.69.40.215 Starting Chef Client, version 12.3.0
52.69.40.215 Creating a new client identity for test-server using the validator key.
52.69.40.215 resolving cookbooks for run list: []
52.69.40.215 Synchronizing Cookbooks:
52.69.40.215 Compiling Cookbooks...
52.69.40.215 [2015-06-15T08:00:47+00:00] WARN: Node test-server has an empty run list.
52.69.40.215 Converging 0 resources
52.69.40.215 
52.69.40.215 Running handlers:
52.69.40.215 Running handlers complete
52.69.40.215 Chef Client finished, 0/0 resources updated in 2.739626435 seconds
```

### Whitelistなしで採集したNodeオブジェクトのキーを確認

無策でやるとタップリとれました。固定的な値と状況による値が混在しているので、例えばnodeオブジェクトもgitの管理対象にしたいなーというケースで特に面倒です。

```
$ knife exec -E "puts nodes.all.first.keys"
tags
filesystem
network
counters
ipaddress
macaddress
ip6address
memory
kernel
lsb
os
os_version
platform
platform_version
platform_family
uptime_seconds
uptime
idletime_seconds
idletime
cpu
virtualization
root_group
block_device
ec2
cloud
ohai_time
languages
command
chef_packages
etc
current_user
init_package
keys
cloud_v2
dmi
hostname
machinename
domain
recipes
roles
```


## Knife Zero Bootstrap でWhitelistを指定する。

Chefに元々ある機能ということで、knife-zeroのv1.7でBootstrap時にWhitelistも使うように機能を追加しました。

たとえばknife.rbにこの様に書いて、あらためてBootstrapしてみます。

```Ruby:knife.rb
knife[:automatic_attribute_whitelist] = [
  "fqdn/",
  "os/",
  "os_version/",
  "hostname",
  "ipaddress/",
  "roles/",
  "recipes/",
  "ipaddress/",
  "platform/",
  "platform_version/",
  "platform_version/",
  "cloud/",
  "cloud_v2/",
  "ec2/ami_id/",
  "ec2/instance_id/",
  "ec2/instance_type/",
  "ec2/placement_availability_zone/",
  "chef_packages/"
]
```


Bootstrapされたインスタンスの`/etc/chef/client.rb`には、`automatic_attribute_whitelist`が追加されました。

```/etc/chef/client.rb 
log_location     STDOUT
chef_server_url  "chefzero://localhost:8889"
validation_client_name "chef-validator"
node_name "test-server"
ssl_verify_mode :none
automatic_attribute_whitelist ["fqdn/", "os/", "os_version/", "hostname", "ipaddress/", "roles/", "recipes/", "ipaddress/", "platform/", "platform_version/", "platform_version/", "cloud/", "cloud_v2/", "ec2/ami_id/", "ec2/instance_id/", "ec2/instance_type/", "ec2/placement_availability_zone/", "chef_packages/"]
```


### Whitelistを指定したNodeオブジェクトの様子

ローカルのnode.json(`nodes/test-server.json`)にあるキーはこれだけになりました。

```
$ knife exec -E "puts nodes.all.first.keys"
tags
os
os_version
hostname
ipaddress
roles
recipes
platform
platform_version
cloud
cloud_v2
ec2
chef_packages
```

Vimで直接開いても画面に収まるくらいですね。

```nodes/test-server.json
{
  "name": "test-server",
  "normal": {
    "tags": [

    ]
  },
  "automatic": {
    "os": "linux",
    "os_version": "3.14.35-28.38.amzn1.x86_64",
    "hostname": "test-server",
    "ipaddress": "10.0.1.122",
    "roles": [

    ],
    "recipes": [

    ],
    "platform": "amazon",
    "platform_version": "2015.03",
    "cloud": {
      "public_ips": [
        "52.69.40.215"
      ],
      "private_ips": [
        "10.0.1.122"
      ],
      "public_ipv4": "52.69.40.215",
      "public_hostname": "",
      "local_ipv4": "10.0.1.122",
      "local_hostname": "test-server.ap-northeast-1.compute.internal",
      "provider": "ec2"
    },
    "cloud_v2": {
      "public_ipv4_addrs": [
        "52.69.40.215"
      ],
      "local_ipv4_addrs": [
        "10.0.1.122"
      ],
      "provider": "ec2",
      "public_hostname": "",
      "local_hostname": "test-server.ap-northeast-1.compute.internal",
      "public_ipv4": "52.69.40.215",
      "local_ipv4": "10.0.1.122"
    },
    "ec2": {
      "ami_id": "ami-cbf90ecb",
      "instance_id": "i-0ef561fb",
      "instance_type": "t2.micro",
      "placement_availability_zone": "ap-northeast-1c"
    },
    "chef_packages": {
      "chef": {
        "version": "12.3.0",
        "chef_root": "/opt/chef/embedded/apps/chef/lib"
      },
      "ohai": {
        "version": "8.3.0",
        "ohai_root": "/opt/chef/embedded/lib/ruby/gems/2.1.0/gems/ohai-8.3.0/lib/ohai"
      }
    }
  }
}
```

これならあまりじゃんじゃんと変更はされません。


## もうBootstrapしちゃったホストの`client.rb`だけを更新して、Whitelist対応したい？

あら古いknife-zeroでBootstrapしちゃったわよ。という環境は、`--no-converge`でbootstrapするやり方が用意されています。

```
--[no-]converge              Bootstrap without Chef-Client Run.(for only update client.rb)
```

通常のBootstrapは一度Chef-Clientが走るため、再度実行する際には気を使いましたが、`--no-converge`オプションで実行しないという選択肢を追加。


```shell:
$ knife zero bootstrap 52.69.40.215 -x ec2-user -i ~/.ssh/private_key --hint ec2 -N test-server --no-converge
Doing old-style registration with the validation key at ...
Delete your validation key in order to use your user credentials instead

Connecting to 52.69.40.215
52.69.40.215 -----> Existing Chef installation detected
52.69.40.215 Starting first Chef Client run...
52.69.40.215 Execution of Chef-Client has been canceled due to bootstrap_converge is false. <= Chef-Client実行をとりやめて終了
```

client.rbが書き換わったので、zero converge(chef_client)を実行すればスッキリNodeになります。

```
$ knife zero converge name:test-server -x ec2-user -i ~/.ssh/private_key -a cloud_v2.public_ipv4 
52.69.40.215 Starting Chef Client, version 12.3.0
52.69.40.215 resolving cookbooks for run list: []
52.69.40.215 Synchronizing Cookbooks:
52.69.40.215 Compiling Cookbooks...
52.69.40.215 [2015-06-15T08:27:08+00:00] WARN: Node test-server has an empty run list.
52.69.40.215 Converging 0 resources
52.69.40.215 [2015-06-15T08:27:08+00:00] WARN: Could not find whitelist attribute fqdn/.
52.69.40.215 
52.69.40.215 Running handlers:
52.69.40.215 Running handlers complete
52.69.40.215 Chef Client finished, 0/0 resources updated in 1.669752105 seconds
```

`--[no-]converge`のフラグは`knife[:bootstrap_converge]`としてknife.rbでも指定OK、CLIオプション優先です。

```
knife[:bootstrap_converge] = true/false
```

ちなみに初回のBootstrapで`--no-converge`しちゃうと、そもそもnodeオブジェクトができないため、その後convergeができません。やり直しです。

----

Twitterで拾った意見と、Github Issueに突貫してきたどこか異国の兄さん達のフィードバックが反映されました。
