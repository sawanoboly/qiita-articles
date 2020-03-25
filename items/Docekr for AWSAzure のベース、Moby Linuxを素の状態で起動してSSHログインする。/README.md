
Docker for AWS(以下D4a)はDocker社が用意したAMIを含む、Swarmクラスタを作るためのCloudFormationテンプレートです。

ここで使われるAMIはAlpineがベースのMoby Linuxというやつで、D4aのテンプレートを無改造で使用すると、インスタンスへのSSHログインは専用のDockerコンテナに入ります。
この方式では、ホスト(manager/worker)に適用しておきたいカスタマイズがあるケースでは、テンプレートに直接試すのが少々面倒でした。

ちょっと試したい追加設定があるたびに、CloudFormationのテンプレートを改造していてはラチがあかないので、あたりをつけるまでは直接触って影響などみたいなーという時に、AMIを単品で起動してしまうこともできます。

ちなみに、beta10までは申請済みのAWSアカウントにSharedの状態でアクセス権が与えられていましたが、beta11以降は気付いたら`Community AMIs`に公開しているようなので、一応誰でも起動してよくなったみたいです。


![EC2_Management_Console.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/6108eb38-7c22-ac50-cde1-7afc7c118aab.jpeg "EC2_Management_Console.jpg")

(なんかbeta15、2つある。。)

> 名前をよく間違えるとかで、割と適当にアップしている模様です。`ec2 describe-images`でオーナーIDと作成日時を一応チェックしてから使いましょう。

## 単品起動にUserScriptを適用する


※ Boot2Dockerも一応Mobyでしたが、EC2で直接触って試したいこともあったりします。

D4aでログイン用のDockerイメージのHistoryを参考に、次のような感じでUserScriptを突っ込んで起動します。

```
#!/bin/sh
docker kill shell-aws
apk add --update bash openssh ca-certificates sudo curl jq
echo -e "Port 22\nClientAliveInterval 180\nPasswordAuthentication no\n" >> /etc/ssh/sshd_config

rc-update add sshd
/etc/init.d/sshd start
echo -e "\ndocker ALL=(ALL) NOPASSWD: ALL\n" >> /etc/sudoers 
```

> 追記2017.9： いつのまにやらshell-awsコンテナが起動するようになっているので、 `docker kill shell-aws` で止めます。
 
あとはストレージを適当に増やしておきましょう。

## SSHでMobyにログイン

ログインユーザは`docker`です。

```
$ ssh docker@52.91.xxx.xxx

...

Welcome to Moby, based on Alpine Linux.
ip-10-0-11-114:~$ id
uid=1001(docker) gid=50(docker) groups=50(docker),50(docker)
```

SSHログインできました。

## D4aのMobyカスタマイズ例

例えば、DockerコンテナからEC2のメタデータを取れないようにしたいとか、そういう設定を試してみるのにつかえます。

通常は次のように、EC2のインスタンス上でDockerコンテナを稼働させると、いつものメタデータAPIを参照することができます。

```
$ docker run -it --rm alpine wget http://169.254.169.254/latest/meta-data/ -O - | head
Connecting to 169.254.169.254 (169.254.169.254:80)
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
hostname
instance-action
instance-id
instance-type
local-hostname
```

えーと、これが見れるようでは困るケースもあるわけです。CI用に貸し出してるとかで。

いくつか情報を拾ったりつつ(どこだったか忘れた。。)、とりあえずPREROUTINGでごまかすことが可能ぽかったので試してみます。到達しないようにしちゃうと。

```
$ sudo iptables -t nat -I PREROUTING -i docker0 -p tcp -d 169.254.169.254 --dport 80 -j DNAT --to-destination 1.1.1.1
$ sudo iptables -t nat -I PREROUTING -i docker_gwbridge -p tcp -d 169.254.169.254 --dport 80 -j DNAT --to-destination 1.1.1.1
```

無事、DockerコンテナからEC2メタデータを隠蔽することができたようです。

```
$ docker run -it --rm alpine wget http://169.254.169.254/latest/meta-data/ -T 10 -O -
Connecting to 169.254.169.254 (169.254.169.254:80)
wget: download timed out
```

あとは、今回成功した`iptables`のコマンドをテンプレートのUserScriptに仕込めばよいはず、と。

----

以下おまけ。


何かの拍子で再起動した時にも適用されるよう、一応`local.d`にも仕込んでおく場合、こんな感じですかね。

```
$ sudo rc-update add local default

$ cat <<"EOL" | sudo tee -a /etc/local.d/block_metadata.start
iptables -t nat -I PREROUTING -i docker0 -p tcp -d 169.254.169.254 --dport 80 -j DNAT --to-destination 1.1.1.1
iptables -t nat -I PREROUTING -i docker_gwbridge -p tcp -d 169.254.169.254 --dport 80 -j DNAT --to-destination 1.1.1.1
EOL

$ sudo chmod +x /etc/local.d/block_metadata.start
```

ただ、今回試したbeta15用のAMIは(AutoScaleGroupで捨てるのが前提だから？)、そもそもrebootしたらSTOPのまま上がってこないので、自動適用の成否はわからんです。
