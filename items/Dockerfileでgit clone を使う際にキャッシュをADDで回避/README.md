

Dockerfile内で`RUN git clone`な行を入れておいて、『ああキャッシュ使われた』ってなるのをどうしよう。
`--no-cache`もあるけど、毎回全ビルドはさすがになあと。

- 参考
    - [New feature request: Selectively disable caching for specific RUN commands in Dockerfile · Issue #1996 · docker/docker](https://github.com/docker/docker/issues/1996 "New feature request: Selectively disable caching for specific RUN commands in Dockerfile · Issue #1996 · docker/docker")
    - [Dockerfile cache on a git repo - Google グループ](https://groups.google.com/forum/#!topic/docker-user/FQ5RufTfG-k "Dockerfile cache on a git repo - Google グループ")


## サンプルDockerイメージ

適当なsinatraアプリでDockerfileのサンプルを作った。

- rubyコンテナベース
- Githubからコードを落とす
- bundlerでライブラリ(Rubygems)を入れる
- 公開先は[ココ](https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB)

```Dockerfile:Dockerfile
FROM ruby
MAINTAINER SAWANOBORI Yukihiko <sawanoboriyu@higanworks.com>

RUN git clone --depth 1 https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB.git
WORKDIR eb_nginxBrockRubyviaELB
RUN bundle install

EXPOSE 9292
ENTRYPOINT ["bundle", "exec", "rackup"]
```

ビルドしてみる。

```shell:docker_build
$ docker build --rm -t ebn:latest .
Sending build context to Docker daemon 3.906 MB
Sending build context to Docker daemon 
Step 0 : FROM ruby
 ---> 51473a2975de
Step 1 : MAINTAINER SAWANOBORI Yukihiko <sawanoboriyu@higanworks.com>
 ---> Using cache
 ---> a1814396dbf3
Step 2 : RUN git clone --depth 1 https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB.git
 ---> Using cache
 ---> 272d8729ebbe
Step 3 : WORKDIR eb_nginxBrockRubyviaELB
 ---> Using cache
 ---> 33929ab00ff0
Step 4 : RUN bundle install
 ---> Using cache
 ---> 08d07d381eb6
Step 5 : EXPOSE 9292
 ---> Using cache
 ---> f5f072ffc73e
Step 6 : ENTRYPOINT bundle exec rackup -o 0.0.0.0
 ---> Using cache
 ---> b39745296053
Successfully built b39745296053
```

ここまでは普通。


### キャッシュ効いたら困る時がある行

これだ。

- `RUN git clone --depth 1 https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB.git`

リモート側の更新があっても、buildの際に調べたりはしない。
そもそもDockerのイメージにgitとか入れるべきではないという話もあるけど、選択肢の１つとしてね。


## ADD

冒頭の参考リンクでは色々議論があって、`ADD`も奨められていた。

[Best practices for writing Dockerfiles - Docker Documentation](https://docs.docker.com/articles/dockerfile_best-practices/#build-cache "Best practices for writing Dockerfiles - Docker Documentation")


じゃあ`RUN git clone`の前にADDを入れてみよう。

```Dockerfile:Dockerfile
FROM ruby
MAINTAINER SAWANOBORI Yukihiko <sawanoboriyu@higanworks.com>

ADD dummyfile /data/   ## これを追加。
RUN git clone --depth 1 https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB.git
WORKDIR eb_nginxBrockRubyviaELB
RUN bundle install

EXPOSE 9292
ENTRYPOINT ["bundle", "exec", "rackup"]
```


ADDにおけるキャッシュ条件判断では[FileInfo](http://golang.org/pkg/os/#FileInfo)のオブジェクト全体からsumを生成してる？っぽかった。
ModTimeを含むので、例えばtouchすればキャッシュと違うという判断をされるのかなと。


```shell:docker_build_with_add
$ touch dummyfile && docker build --rm -t ebn:latest .
Sending build context to Docker daemon 
Step 0 : FROM ruby
 ---> 51473a2975de
Step 1 : MAINTAINER SAWANOBORI Yukihiko <sawanoboriyu@higanworks.com>
 ---> Using cache
 ---> a1814396dbf3
Step 2 : ADD dummyfile /data/       ## ここからキャッシュじゃなくなる
 ---> 0572e2f222c5
Removing intermediate container f2b365e473ad
Step 3 : RUN git clone --depth 1 https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB.git
 ---> Running in 35f2bc462d1e
Cloning into 'eb_nginxBrockRubyviaELB'...
 ---> 2bfaa75326d2
Removing intermediate container 35f2bc462d1e
Step 4 : WORKDIR eb_nginxBrockRubyviaELB
 ---> Running in b3781281f1f7
 ---> a8fb39ad4ca0
Removing intermediate container b3781281f1f7
Step 5 : RUN bundle install
 ---> Running in b51e6bc8f546
Don't run Bundler as root. Bundler can ask for sudo if it is needed, and
installing your bundle as root will break this application for all non-root
users on this machine.
Fetching gem metadata from https://rubygems.org/..........
Installing rack 1.6.0

-- 以下省略
```

`toucn && build`は何度繰り返してもADD以降をビルドするようになった。


### ADDの対象

Docker buildを起動するワーキングディレクトリ次第で選ぶ。
gitリポジトリの場合はこのへんのファイルをADDにすると、更新の必要がないときはキャッシュも効くし丁度いいかもしれない。

- .git/index
- .git/logs/HEAD


他のケースでは、たとえばElasticBeanstalkのDockerタイプにebコマンドでデプロイするなら、ワーキングディレクトリが毎回作られるので`ADD dummyfile /data/`は毎回別ハッシュになる。要はtouchの必要が無い。
逆に`.git/`が無くなるのでそこは使えない。

ADDはfromに適当なURLも指定できるので、[Issueにあったサンプル](https://github.com/docker/docker/issues/1996#issuecomment-51292997)ではランダム文字列を生成するWEBサービスを呼んでいた。



## ADD以降

ADD以降全部やり直し、になるのもそれはそれで勿体無い。
アプリの場合大抵後にライブラリのインストール、`bundler`(Ruby)や`npm install`(node.js)が必要で、場合によっては結構時間がかかる。


### ライブラリ用に親ホスト(または別コンテナ)を使う

とりあえずbundlerならgemを置くパスが選べるので、Rubygemsのインストール先をマウントしてみることにした。Capistranoの要領だ。
条件が限定されるものの時間が短縮できそう。


`bundle install`の先がbuild時には使えないため、Dockerfileから出してスクリプトに。


```bin/run_on_docker.sh
#!/usr/bin/env bash
bundle install --path=/data/shared/tmp_myapp/vendor
bundle exec rackup -o 0.0.0.0
```

`ENTRYPOINT`も追って変更。

```Dockerfile:Dockerfile
FROM ruby
MAINTAINER SAWANOBORI Yukihiko <sawanoboriyu@higanworks.com>

ADD dummyfile /data/
RUN git clone --depth 1 https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB.git
WORKDIR eb_nginxBrockRubyviaELB

EXPOSE 9292
ENTRYPOINT ["./bin/run_on_docker.sh"]
```

そして`docker run`すると初回は`Installing ...`でbundle.

```shell:docker_run1
$ docker run -v /tmp:/data/shared --rm -p 9292:9292 -ti ebn:latest 
Don't run Bundler as root. Bundler can ask for sudo if it is needed, and installing your bundle as root will break this application for all non-root users on this machine.
Fetching gem metadata from https://rubygems.org/..........
Installing rack 1.6.0
Installing puma 2.11.0
Installing rack-protection 1.5.3
Installing tilt 1.4.1
Installing sinatra 1.4.5
Using bundler 1.7.12
Your bundle is complete!
It was installed into /data/shared/tmp_myapp/vendor
Puma 2.11.0 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:9292
```


止めてもう一度`docker run`するとGemfile.lockに沿って`Using ...`でbundle.


```shell:docker_run2
$ docker run -v /tmp:/data/shared --rm -p 9292:9292 -ti ebn:latest 
Don't run Bundler as root. Bundler can ask for sudo if it is needed, and installing your bundle as root will break this application for all non-root users on this machine.
Using rack 1.6.0
Using puma 2.11.0
Using rack-protection 1.5.3
Using tilt 1.4.1
Using sinatra 1.4.5
Using bundler 1.7.12
Your bundle is complete!
It was installed into /data/shared/tmp_myapp/vendor
Puma 2.11.0 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:9292
```

コンテナを止めた後、親(boot2docker)からマウントしていた所にはRubygemsが残されている。

```shell:show_cached_gems
(docker@boot2docker) $ ll /tmp/tmp_myapp/vendor/ruby/2.2.0/

total 28
drwxr-xr-x    2 root     root          4096 Feb  2 07:03 bin/
drwxr-xr-x    2 root     root          4096 Feb  2 07:03 build_info/
drwxr-xr-x    2 root     root          4096 Feb  2 07:03 cache/
drwxr-xr-x    2 root     root          4096 Feb  2 07:03 doc/
drwxr-xr-x    3 root     root          4096 Feb  2 07:03 extensions/
drwxr-xr-x    7 root     root          4096 Feb  2 07:03 gems/
drwxr-xr-x    2 root     root          4096 Feb  2 07:03 specifications/
```


ベストとまでは思わないけど、色々と回避することができた。
