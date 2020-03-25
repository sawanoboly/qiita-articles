<!-- permanent -->


Amazon EC2でCentOSを使いたい場合、CentOSコミュニティがAWS MarketPlaceに公開しているイメージが一応ある。


![CentOS_6__x86_64__-_with_Updates_on_AWS_Marketplace.png](https://qiita-image-store.s3.amazonaws.com/0/7454/30cb71f0-b992-8ca7-1eb2-75afea9ffb68.png "CentOS_6__x86_64__-_with_Updates_on_AWS_Marketplace.png")


かなり素の状態で置いてあり、例えば次のような点でAmazon Linuxなどと異なっている。

- ログインユーザがroot
    - ついでにランダムパスワード
- Iptables, Selinuxが有効
- `ec2metadata`や`aws`コマンドがない (curlで取るのはOK)
- cloud-initが入ってない

## Packerで調整して保存

詳しくはこちら。

[https://github.com/OpsRockin/centos6_ami_from_official](https://github.com/OpsRockin/centos6_ami_from_official)

これにより、次のように調整したAMIを作れる。

- - Iptables, Selinux無効
- ログインユーザは`cloud-user`に、sudo対応
    - `Please login as the user "cloud-user" rather than the user "root".`
- `aws`コマンド標準装備、Roleも対応

PackerのShell-Provisionerはこんな感じ。

```shell:bootstrap.sh
#!/usr/bin/env bash

set -ex

# Disable Selinux and Iptables
sed s/SELINUX=enforcing/SELINUX=disabled/ /etc/selinux/config -i
chkconfig iptables off
chkconfig ip6tables off

yum update -y


## Install AWS Tools and Git
rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum install -y cloud-init git jq cloud-utils python-pip
pip install awscli


## Remove Password from root
passwd -d root
sed -e '9,$d' /etc/rc.local -i


## Set sudo rule for cloud-user
cat <<EOL > cloud-init
cloud-user ALL = NOPASSWD: ALL

# User rules for cloud-user
cloud-user ALL=(ALL) NOPASSWD:ALL
Defaults:cloud-user !requiretty
EOL

install -o root -g root -m 0440 cloud-init /etc/sudoers.d/cloud-init
rm -f cloud-init
```

これで作っておいたAMIに対しては、通常のAmazon公式イメージとほぼ同様の扱いが可能になる。

```
$ ssh cloud-user@ec2-xx-xx-xxx-xxx.compute-1.amazonaws.com sudo echo 'hogehoge'
hogehoge
```

元がAWS MarketPlaceのイメージなので、これを更にPublicにする事はできないけど。
