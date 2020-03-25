
Amazon Elastic File System (以下EFS)はざっくりいうとマネージドのNFSだ。
このEFSとDocker Swarmの新機能(1.12〜)、ingressとELBを組み合わせてみよう。

なんかユルいクラスタがつくれます。


記事を書いてる時点では、Dockerの1.12はrc4が出たとこですね。



事前に必要なもの。

- Amazon EFSボリューム
    - インスタンスからマウント可能なSGの設定


## Docker-MachineからEC2インスタンスを作成する。

EFSと同じVPCに作ってね。EFSがあるリージョンでよろしく。２個作るか。

```shell:(local_machine)
$ docker-machine create --driver amazonec2 \
  --amazonec2-region us-west-2 \
  --amazonec2-instance-type t2.small \
  --amazonec2-vpc-id vpc-xxxxxxxx \
  --amazonec2-subnet-id subnet-xxxxxxxx \
  aws-efs-sandbox01

....


$ docker-machine create --driver amazonec2 \
  --amazonec2-region us-west-2 \
  --amazonec2-instance-type t2.small \
  --amazonec2-vpc-id vpc-xxxxxxxx \
  --amazonec2-subnet-id subnet-xxxxxxxx \
  aws-efs-sandbox02

...
```

amazonec2ドライバは(この記事時点で)Ubuntuの15.10でインスタンスを作成します。
15.10のままでも良いし、`do-release-upgrade`で16.04にしちゃっても動いてましたのでお好みで。

### Docker-Engineを1.12.RCx以降にする

それぞれのインスタンスに`docker-machine ssh`で侵入します。

```shell:(local_machine)
$ docker-machine ssh aws-efs-sandbox01
```

ログインできたら`test.docker.com`をぶっこみましょう。

```shell:(remote_user)
$ curl -fsSL https://test.docker.com/ | sh


Warning: the "docker" command appears to already exist on this system.

If you already have Docker installed, this script can cause trouble, which is
why we're displaying this warning and provide the opportunity to cancel the
installation.


```shell:(remote_user)
Client:
 Version:      1.12.0-rc4
 API version:  1.24
 Go version:   go1.6.2
 Git commit:   e4a0dbc
 Built:        Wed Jul 13 04:00:11 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.0-rc4
 API version:  1.24
 Go version:   go1.6.2
 Git commit:   e4a0dbc
 Built:        Wed Jul 13 04:00:11 2016
 OS/Arch:      linux/amd64

If you would like to use Docker as a non-root user, you should now consider
adding your user to the "docker" group with something like:

  sudo usermod -aG docker ubuntu

Remember that you will have to log out and back in for this to take effect!

```

OKでーす。

```shell:
$ sudo docker -v
Docker version 1.12.0-rc4, build e4a0dbc
```

この時点で多分一回rebootしとくほうが良いと思います。


## Swarmを組みます

1.12からSwarmが組みやすい、組もうぜみんな。

aws-efs-sandbox01からinit。

```shell:(remote_user_and_root)
$ sudo -s # rootのほうが補完が利いて楽ちん

# (aws-efs-sandbox01) 
$ docker swarm init 
No --secret provided. Generated random secret:
	xxxxxxxxxxxxxxxxxxxxxxx

Swarm initialized: current node (xxxxxxxxxxxxxxxxxxxxxxxxxxxx) is now a manager.

To add a worker to this swarm, run the following command:
	docker swarm join --secret xxxxxxxxxxxxxxxxxxxxxxxxxxx \
	--ca-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
   xx.xx.xx.xx:2377
```

aws-efs-sandbox02からはjoin。

```shell:(remote_root)
# (aws-efs-sandbox02)
$ docker swarm join --secret xxxxxxxxxxxxxxxxxxxxxxxxxxx \
	--ca-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
   xx.xx.xx.xx:2377

This node joined a Swarm as a worker.
```

Docker nodeを確認しましょう。いるわね。

```shell:(remote_root)
# (aws-efs-sandbox01)
$ docker node ls
ID                           HOSTNAME           MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
xxxxxxxxxxxxxxxxxxxxxxxxx *  aws-efs-sandbox01  Accepted    Ready   Active        Leader
xxxxxxxxxxxxxxxxxxxxxxxxx    aws-efs-sandbox02  Accepted    Ready   Active   
```

`aws-efs-sandbox02`もついでにマネージャーにします。

```shell:(remote_root)
# (aws-efs-sandbox01)
$ docker node promote  aws-efs-sandbox02
Node aws-efs-sandbox02 promoted to a manager in the swarm.
```

ヒラのWorkerから、`MANAGER STATUS` => `Reachable` になったね。これでどちらからでもSwarmをコントロールできる。

```shell:(remote_root)
# (aws-efs-sandbox01)
$ docker node ls
ID                           HOSTNAME           MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
xxxxxxxxxxxxxxxxxxxxxxxxx *  aws-efs-sandbox01  Accepted    Ready   Active        Leader
xxxxxxxxxxxxxxxxxxxxxxxxx    aws-efs-sandbox02  Accepted    Ready   Active        Reachable


# (aws-efs-sandbox02)
$ docker node ls
ID                           HOSTNAME           MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
xxxxxxxxxxxxxxxxxxxxxxxxx    aws-efs-sandbox01  Accepted    Ready   Active        Leader
xxxxxxxxxxxxxxxxxxxxxxxxx *  aws-efs-sandbox02  Accepted    Ready   Active        Reachable
```


### ついでにELBを

ELBを作って、起動したインスタンスをぶら下げます。

- とりあえずTCPの2376をヘルスチェックついでにふっちゃえばよいかと思います。
- SGのinboundはDockerコンテナが使う範囲を許可(例えば10000-11000とか)
- Docker-MachinesなインスタンスはELBからのアクセスを許可。



## EFSをホストでNFSマウントして、Dockerコンテナにmountする

さて、EFSについて。NFSなのでまずはホストからマウントして、Dockerコンテナの起動時にそのパスをマウントしてみます。
両方のインスタンスで、同じパスにマウントします。

```shell:(remote_user)
$ sudo apt-get -y install nfs-common 
$ sudo mkdir -p /mnt/testefs


$ sudo mount -t nfs4 -o nfsvers=4.1,relatime,hard $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone).fs-xxxxxxxx.efs.us-west-2.amazonaws.com:/ /mnt/testefs
```

ここでマウントできない場合は多分SGです。

`aws-efs-sandbox01`で書いて、`aws-efs-sandbox02`で読んでみる。

```shell:(remote_root)
# (aws-efs-sandbox01)
$ echo "hello" > /mnt/testefs/index.html


# (aws-efs-sandbox02)
$ cat /mnt/testefs/index.html 
hello
```

よろしいね。


### docker run with nfs(from host)

ホストにマウントしているので、`docker run`はまあ普通にこのようになる。

```shell:(remote_root)
$ docker run -it --rm -v /mnt/testefs:/usr/share/nginx/html nginx  cat /usr/share/nginx/html/index.html
hello
```

まあこりゃ普通だね。どのインスタンスでも同じコンテンツになるのはよいけども。

### docker service with nfs(from host)

さて、Swarmのingressにぶら下げるべく`service create`で起動。

```shell:(remote_root)
# (aws-efs-sandbox01)
$ docker service create --name nginx \
    -p 10080:80 \
    --mount target=/usr/share/nginx/html,source=/mnt/testefs,type=bind,readonly=true \
    nginx

> 44fzmwtypbw501rxwo8he54nx

$ docker service tasks nginx

ID                         NAME     SERVICE  IMAGE  LAST STATE              DESIRED STATE  NODE
44fzmwtypbw501rxwo8he54nx  nginx.1  nginx    nginx  Running 14 minutes ago  Running        aws-efs-sandbox02
```

`aws-efs-sandbox01`でcreateしたけど、`aws-efs-sandbox02`でDockerコンテナが起動した。

で、両方のインスタンスで`TCP/10080`がListenされました。これがあれだね、ingress。

```shell:(remote_user)
# (aws-efs-sandbox01)
$ sudo netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
...

tcp6       0      0 :::10080                :::*                    LISTEN     
...


$ curl 127.0.0.1:10080
hello


# (aws-efs-sandbox02)
$ sudo netstat -tnl
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
...

tcp6       0      0 :::10080                :::*                    LISTEN     
...


$ curl 127.0.0.1:10080
hello

```


どちらでも`curl 127.0.0.1:10080`が同じコンテンツを返してる。


## ELBにリスナを追加して、アクセスを振ってみる

今回Nginxだし、HTTPでリスナ追加してみようか。10080=>10080でいいや。

![EC2_Management_Console.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/fe16a892-da0f-4582-207a-a5c713a7fb11.jpeg "EC2_Management_Console.jpg")

ブラウザでみます。

![swarm-test-1588239295_us-west-2_elb_amazonaws_com_10080_と_EC2_Management_Console.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/15a18041-4308-e72e-34ee-5f3eeeda7864.jpeg "swarm-test-1588239295_us-west-2_elb_amazonaws_com_10080_と_EC2_Management_Console.jpg")

どっちを経由したかわかりませんけど、みえますね。NATのテーブルでは`docker_gwbridge`ってのを通っていました。

スケールもOKです。

```shell:(remote_root)
$ docker service scale nginx=2
nginx scaled to 2
```

スケールをインスタンス数以上(3とか)にしても、ラウンドロビンでアクセスを振り分けてるのを確認できました。



## (おまけ)docker-volume-netshareで、Dockerコンテナに直接EFSをNFSマウントさせる

DockerのVolumeドライバnetshareはNFSに対応してます。 [ContainX/docker-volume-netshare](https://github.com/ContainX/docker-volume-netshare)


インストールはこんな感じで。

```shell:(remote_user)
$ wget https://github.com/ContainX/docker-volume-netshare/releases/download/v0.18/docker-volume-netshare_0.18_amd64.deb
$ sudo dpkg -i docker-volume-netshare_0.18_amd64.deb
```

両方のノードでNFSモード起動しておきます。とりあえずフォアグラウンドでいいっしょ。

```shell:(remote_user)
$ sudo docker-volume-netshare nfs
INFO[0000] == docker-volume-netshare :: Version: 0.18 - Built: 2016-05-27T20:14:07-07:00 == 
INFO[0000] Starting NFS Version 4 :: options: '' 
```



### docker run with nfs(via nfs volume-driver)

この状態なら`--volume-driver=nfs`でDockerコンテナがNFSマウントっぽいことになります。


```shell:(remote_root)
$ docker run -it --rm --volume-driver=nfs \
  -p 10080:80 \
  -v $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone).fs-xxxxxx.efs.us-west-2.amazonaws.com/:/usr/share/nginx/html \
  nginx
```

とりあえずEFSのコンテンツであることを確認。

```shell:(remote_user)
$ curl 127.0.0.1:10080
hello
```

Dockerコンテナ内でmount状況をみると、次のようになってます。

```shell:(remote_root)
$ docker exec 5df1f86f8c64 mount | grep nfs
us-west-2a.fs-xxxxxx.efs.us-west-2.amazonaws.com:/ on /usr/share/nginx/html type nfs4 (rw,relatime,vers=4.0,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.0.0.212,local_lock=none,addr=10.0.0.135)
```

実体は`docker volume`。


### docker service with nfs(via nfs volume-driver)

で。
これができる事を期待したんだけど、ダイレクトにマウントするのはまだ無理っぽい？このへんオプションがカオスなのよね。
ホストのNFSマウントが不要になるから使い勝手がいいんだけどなあ。

Issueで相談中のようなので、なんとかなるだろう。

[netshare volume plugin not working in "swarm mode" v1.12.0-rc3 with docker volume create, docker service create --mount · Issue #48 · ContainX/docker-volume-netshare](https://github.com/ContainX/docker-volume-netshare/issues/48)

普通にdokcer runした時のinspectを見ると、なんか色々とオプションの変換が必要そうでややこしい。今回はちょっとスルー。



## このあと

一応、仕組み的にはListen出来る限り(IPアドレス * 60,000程度)のSwarm Serviceを公開できるってことになりますね。

ただ、ELBのリスナーは1つあたり同時に100までという制限があったり、ついでにいうと名前ベースによる振り分けも無かったりです。Webアプリを使うならばngx_mrubyとかで動的リバースプロキシを作るとよさそう。

Google Cloud PlatformのGKEならばもっと簡単にLBをつけられるんだけど、Googleだと今なら同じことする場合、GlusterFSのサービスをGCEかk8s上に作ることになるんだよね。
