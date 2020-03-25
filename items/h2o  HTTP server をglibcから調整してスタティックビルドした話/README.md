
[h2oをbusyboxのコンテナに突っ込んだら軽そう - Qiita](http://qiita.com/sawanoboly/items/c24a6ad45e18f682173d "h2oをbusyboxのコンテナに突っ込んだら軽そう - Qiita") の続編。

前回、名前解決の関係で一部機能が使えてなかったところを改善したのでその記録。

<del>また、nss関連を取り込めたので、busyboxのベースを`busybox:ubuntu`から`busybox`に変更しても問題なくなった。イメージサイズも7MB => 5MBに。</del>

> 1.6.0でのTLSでperl, opensslコマンドが必要だったので、alpineに乗り換えました。 https://hub.docker.com/r/higanworks/h2o-alpine 
> 見た目のサイズは18MB ? perl, opensslパッケージの大きさと少々違う気がしますが...

![Docker_Hub.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/5eb4ca80-67fb-ec8a-a785-2456153aeed8.jpeg "Docker_Hub.jpg")


↓↓↓

![Docker_Hub.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/e5e66dd1-3673-8cbe-ac09-087ad3f4de4d.jpeg "Docker_Hub.jpg")



## glibcのdebカスタマイズ、ならば`--enable-static-nss`だ

`make h2o` 時に出ていた警告。getaddrinfoなどはこのglibcではあかんでーという話だった。

```
CMakeFiles/h2o.dir/lib/common/serverutil.c.o: In function `h2o_setuidgid':
/h2o/lib/common/serverutil.c:72: warning: Using 'initgroups' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
CMakeFiles/h2o.dir/src/main.c.o: In function `on_config_user':
/h2o/src/main.c:970: warning: Using 'getpwnam' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
CMakeFiles/h2o.dir/lib/common/serverutil.c.o: In function `h2o_setuidgid':
/h2o/lib/common/serverutil.c:60: warning: Using 'getpwnam_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
CMakeFiles/h2o.dir/deps/libyrmcds/connect.c.o: In function `connect_to_server':
/h2o/deps/libyrmcds/connect.c:36: warning: Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```

このlibnssあたりのライブラリを取り込んだbusyboxが`busybox:ubuntu`のようなんだが、結局SegFaultしていた。

元々使おうとしていた用途でなら機能的に問題なさそうではあったが、OCSPとか動かんのかしらと気になっていたのでやっぱりglibcから作り直すことにした。

雑いがこんな感じ。

```
FROM buildpack-deps
RUN echo "deb-src http://ftp.jp.debian.org/debian/ jessie main" >> /etc/apt/sources.list
RUN echo "deb-src http://security.debian.org/ jessie/updates main" >> /etc/apt/sources.list
RUN mkdir -p /glibc
WORKDIR /glibc
RUN apt-get update && apt-get install -y \
cmake dpkg-dev \
&& apt-get build-dep -y glibc --fix-missing \
&& apt-get source -y glibc \
&& rm -rf /var/lib/apt/lists/*

# CONFARGS += --enable-static-nss
WORKDIR /glibc/glibc-2.19
ENV DEB_BUILD_OPTIONS="nocheck parallel=4"
RUN sed -i '/--without-selinux/i\--enable-static-nss\ \\' debian/rules.d/build.mk
RUN dpkg-buildpackage -us -uc -rfakeroot

WORKDIR /glibc
RUN dpkg -i *.deb

WORKDIR /
RUN git clone -b v1.6.0 https://github.com/h2o/h2o --depth 1
WORKDIR /h2o
RUN git submodule update --init --recursive
RUN echo 'SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a")' >> CMakeLists.txt
RUN echo 'SET(BUILD_SHARED_LIBRARIES OFF)' >> CMakeLists.txt
RUN echo 'SET(CMAKE_EXE_LINKER_FLAGS "-static -static-libstdc++ -static-libgcc")' >> CMakeLists.txt
RUN cmake -DWITH_BUNDLED_SSL=on -DBUILD_SHARED_LIBS=no .\
&& make h2o
CMD ["/bin/bash"]
```

これでdocker buildするとイメージサイズが3GBを超える。あとはh2oをbusyboxに入れるだけなので、もはやビルドのほうは容量を気にしていない。

> 修正: べースをalpineに変更

```
FROM alpine
RUN apk --update add perl openssl \
     && rm -rf /var/cache/apk/*
ADD build/h2o /usr/local/bin/h2o
ADD build/share/* /usr/local/share/h2o/
WORKDIR /usr/local
CMD ["/bin/sh"]
```

スッキリ。

```
$ docker run -it --rm higanworks/h2o-alpine:1.6.0 h2o --version
h2o version 1.6.0
OpenSSL: LibreSSL 2.2.4
```

nobodyで動かせるし、`proxy.reverse.url`も利く。

## おわりに

staticにしたことで(ほぼ)単品で動くようになった。この時点でそもそも64bitのlinuxならなんでもいい気がするので、Dockerコンテナでなくでもいいっちゃいい。。
が、コンテナのベースはやっぱり小さい方がよいよねえと。composeとかでちょちょっと増やしたりにも。

