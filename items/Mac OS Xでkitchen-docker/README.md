<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[portertech/kitchen-docker](https://github.com/portertech/kitchen-docker "portertech/kitchen-docker") がMac内だけでも使える。


## boot2dockerの準備

Macなのでboot2dockerでホストVMを作ります。

```bash:
$ boot2docker start
Waiting for VM to be started...
...........
Started.
Auto detection of the VM's IP address.
To connect the Docker client to the Docker daemon, please set:
     export DOCKER_HOST=tcp://:xxxx
```

通常のDocker使用では`export DOCKER_HOST=tcp://:xxxx`でdocekrコマンドからホストVMを意識せずに使えます。
Macでkitchen-dockerする場合、コンテナへのSSH接続に`boot2docker`で作成したホストVMを対象にとりたいので、IPアドレスを指定します。

### IPアドレスを付けてエクスポート

`boot2docker ip`でホストVMのアドレスを取得。

```bash:
$ boot2docker ip

The VM's Host only interface IP address is: 192.168.59.yyy
```


くっつけてエクスポートしましょう。

```
export DOCKER_HOST=tcp://192.168.59.yyy:xxxx
```

アドレス省略時、kitchenコマンドはssh接続時に`127.0.0.1:${sshd用にフォワードしたポート}`に行くのでコンテナ作成はできますが、続きができません。


## `.kitchen.local.yml`で上書きマージ

Test-Kitchenの設定を`.kitchen.local.yml`で差分だけ作ります。
環境変数にもあることだし、ERBとして使い回しましょう。

```yaml:.kitchen.local.yml
---
driver:
  name: docker
  binary: /usr/local/bin/docker
  socket: <%= ENV['DOCKER_HOST'] %>

platforms:
  - name: ubuntu-12.04
    driver_config:
      image: ubuntu:12.04
    run_list:
      - recipe[ubuntu]
      - recipe[apt]
```

ではconvergeと。


```bash:
$ kitchen converge

-- snip --

        "Ports": {
            "22/tcp": [
                {
                    "HostIp": "0.0.0.0",
                    "HostPort": "49153"

-- snip --

       Waiting for 192.168.59.xxx:49153...  ## ここが127.0.0.1宛になって止まる

-- snip --

$ kitchen list
Instance                  Driver  Provisioner  Last Action
default-ubuntu-1204       Docker  ChefZero     Converged

$ uname -v
Darwin Kernel Version 13.2.0: Thu Apr 17 23:03:13 PDT 2014; root:xnu-2422.100.13~1/RELEASE_X86_64
```

できました。
