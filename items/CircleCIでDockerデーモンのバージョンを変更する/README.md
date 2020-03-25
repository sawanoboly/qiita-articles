
この記事時点でCircleCIのDockerは1.6.2のcircleciエディション(Docker version 1.6.2-circleci, build 2f3236d)が標準で動きます。
ただ、諸事情でもうすこし新しいDocker使いたいので差し替えの方法。


CircleCIがフォークしているDockerはこちらにありました。 [circleci/docker](https://github.com/circleci/docker)

## バージョン差し替えのcircle.yml

じゃあこれをビルドすればイイのかなーとgit cloneしてきた所、`circle.yml`を発見。

https://github.com/circleci/docker/blob/docker-1.8.2/circle.yml

```circle.yml
machine:
  pre:
    - echo 'DOCKER_OPTS="-s btrfs -e lxc -D --userland-proxy=false"' | sudo tee -a /etc/default/docker
    - sudo curl -L -o /usr/bin/docker 'https://s3-external-1.amazonaws.com/circle-downloads/docker-1.8.2-circleci'
    - sudo chmod 0755 /usr/bin/docker
  services:
    - docker
```

おお、結合テスト用に差し替えの手順が。

ありがとう。

```
ubuntu@box124:~$ docker -v
Docker version 1.8.2-circleci, build a8b52f5
```

----

追記：

CicleCIの中の人に『(さっきの問い合わせの件は)フォークで動いたッス。Thanks』 などと伝えた所、それはテスト中なんでお客さん向けにはお知らせはしてないのですわと。

> Glad it worked!
> We haven't notified docker1.8.2 to customers because we are still testing but it should work for you. 

言うまでもないけど、正式にリリースされないうちは自己責任でどーぞ。
