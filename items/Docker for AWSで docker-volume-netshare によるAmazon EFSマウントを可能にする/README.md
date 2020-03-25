> <del>これ失敗。</del> からやっぱり成功。

少々義理人情的によくないやりかたなので、詳細は省きます。

https://github.com/sawanoboly/docker-efs-mounter-for-docker-for-aws

Docker for AWSのCFnテンプレートに2箇所、これを入れます。

```
"mkdir -p /var/lib/docker-volumes/netshare/efs\n",
"mount --bind /var/lib/docker-volumes/netshare/efs /var/lib/docker-volumes/netshare/efs\n",
"mount --make-shared /var/lib/docker-volumes/netshare/efs\n",
"docker run --restart=always -d ",
"--name=netshare-efs-for-d4aws ",
"--privileged ",
"-v /var/run/docker/plugins:/var/run/docker/plugins ",
"-v /etc/resolv.conf:/etc/resolv.conf ",
"-v /var/lib/docker-volumes/netshare/efs:/var/lib/docker-volumes/netshare/efs:shared ",
"sawanoboly/docker-efs-mounter-for-docker-for-aws:0.20-1\n"

```

システムコンテナを増やしちゃうわけですね。2箇所入れれば全部のノードで起動するはずです。
ついでにD4xのフィードバックにはこれがデフォルトだったらうれしいなーと送信しました。


## Managerで様子を確認

`docker@{ELB endpoint}` でログインしたら、勝手に追加したDockerコンテナを確認します。ありますね。

```
$ docker ps
CONTAINER ID        IMAGE                                       COMMAND                  CREATED             STATUS              PORTS                NAMES
9700c6b85cc6        sawanoboly/netshare-efs-for-d4aws:latest    "/usr/bin/docker-volu"   7 minutes ago       Up 7 minutes                             netshare-efs-for-d4aws
de98d7bb17ad        docker4x/controller:aws-v1.12.1-rc1-beta5   "loadbalancer run --l"   7 minutes ago       Up 7 minutes                             editions_controller
61e479c6aa9b        docker4x/shell-aws:aws-v1.12.1-rc1-beta5    "/entry.sh /usr/sbin/"   7 minutes ago       Up 7 minutes        0.0.0.0:22->22/tcp   evil_turing
385d6257417e        docker4x/guide-aws:aws-v1.12.1-rc1-beta5    "/entry.sh"              7 minutes ago       Up 7 minutes                             determined_mahavira
```

## docker-volume-netshareによるvolume-driver=efsの動作確認

> EFSのSGは`SwarmWideSG`をつけたらOK。

docker-volume-netshareのログをみます、起動してるねー。

```
$ docker logs 9700c6b85cc6
time="2016-08-16T10:29:31Z" level=info msg="== docker-volume-netshare :: Version: 0.17 - Built: 2016-05-18T09:50:31-07:00 ==" 
time="2016-08-16T10:29:31Z" level=info msg="Starting EFS :: availability-zone: , resolve: false, ns: " 
```

serviceは書式が面倒なので、取り急ぎubuntuのコンテナをRUNして終わらせます。EFSのIDだけで良いのが楽だよね。

``
docker pull ubuntu:latest
docker run -it --rm --volume-driver=efs -v fs-xxxxxx:/mount ubuntu /bin/true
```

もう一度docker-volume-netshareのログをみます、バッチリMount/Unmountされてますわ。

```
$ docker  logs 9700c6b85cc6
time="2016-08-16T10:29:31Z" level=info msg="== docker-volume-netshare :: Version: 0.17 - Built: 2016-05-18T09:50:31-07:00 ==" 
time="2016-08-16T10:29:31Z" level=info msg="Starting EFS :: availability-zone: , resolve: false, ns: " 
time="2016-08-16T10:39:25Z" level=info msg="Mounting EFS volume 192.168.34.130:/ on /var/lib/docker-volumes/netshare/efs/fs-xxxxxxxx" 
time="2016-08-16T10:39:37Z" level=info msg="Unmounting volume 192.168.34.130:/ from /var/lib/docker-volumes/netshare/efs/fs-xxxxxxxx" 
```

## 一言

テンプレート改変はねー。。おすすめしたくないので公式に送ったフィードバックに期待しよう。

----

## 追記

`docker service`でもこんな感じの指定で使えました。


```
docker service create --name mysite0003 -p 10002:8080 \
--mount type=volume,src=fs-xxxx/mysite0003,dst=/var/www/html,volume-driver=efs \
--with-registry-auth \
xxxxxxxxxxxxxxx.dkr.ecr.us-west-2.amazonaws.com/static-site-hosting
```

マウントソースのEFSに指定したディレクトリがなくてもマウントする際に作ってくれるのでわりとハッピー。
