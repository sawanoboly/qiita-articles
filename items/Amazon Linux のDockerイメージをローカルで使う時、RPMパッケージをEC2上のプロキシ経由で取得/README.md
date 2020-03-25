Amazon Linuxが公式にDockerイメージを配布しだしました。

[Introducing New Amazon Linux Container Image for Cloud and On-Premises Workloads](https://aws.amazon.com/jp/about-aws/whats-new/2016/11/introducing-new-amazon-linux-container-image-for-cloud-and-on-premises-workloads/)

個人的にはなんでも良いのだけど、これをベースにして欲しいという要望がちらほらある。
で、これ結構空っぽなのよね。curlが入っているので、中身の構築自体はなんとでもなりそうだけど、ちょっと触ってみる分にはrpmのパッケージくらいは取れるとよいよね。

> Amazon Linux用RPMのリポジトリはAWSのネットワークからしか取得<del>できないので、EC2に取らせてみた。</del>できなかった気がしたが、気のせいだったかもしれない。


## 適当なEC2インスタンスでフォワードプロキシを起動

> 追記： もうプロキシなくてもつかえる模様

まずEC2のインスタンスを起動します。squidあたりでフォワードプロキシ(全開け)をして試してみます。

```
$ sudo yum install squid -y
$ sudo sed 's/http_access deny all/http_access allow all/g' /etc/squid/squid.conf -i
$ sudo service start squid
```

とりあえずこれで、デフォルトの`TCP/3128`でHTTPプロキシが起動。あとはこれを指定と。

## ローカルのDockerコンテナ内からyum

ECRからpullしておいた`amazonlinux`をrunして、yumしてみる。

```
$ docker run -it --rm 137112412989.dkr.ecr.ap-northeast-1.amazonaws.com/amazonlinux:latest 

## 以降Dockerコンテナ内

bash-4.2# http_proxy=http://54.199.xxx.xx:3128 yum install tar
Loaded plugins: priorities, update-motd, upgrade-helper
Resolving Dependencies
--> Running transaction check
---> Package tar.x86_64 2:1.26-31.22.amzn1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===============================================================================================================================================================================================================
 Package                                     Arch                                           Version                                                    Repository                                         Size
===============================================================================================================================================================================================================
Installing:
 tar                                         x86_64                                         2:1.26-31.22.amzn1                                         amzn-main                                         1.1 M

Transaction Summary
===============================================================================================================================================================================================================
Install  1 Package

Total download size: 1.1 M
Installed size: 2.7 M
Is this ok [y/d/N]: 
```

OK。tarくらいは使えるほうがいいな。
公式に持ち出し禁止だった時はしなかったけど、もうこれやってもいいってことだよねぇ？


## buildとか


ビルドの時は`build-arg`で`http_proxy`を渡してもいいし、yumの設定に書いても良い。
EC2でプロキシする以外、他にも色々やり方はありそうなんで、何が一番コストかからないか試してみるのもいいかもね。
