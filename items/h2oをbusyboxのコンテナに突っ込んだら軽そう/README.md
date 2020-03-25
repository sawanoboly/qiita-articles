
HTTPサーバの[h2o](https://h2o.examp1e.net)を単品のバイナリにしてbusyboxのコンテナで動くかなという実験。

![Quick_Start_-_Configure_-_H2O.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/9b8836d0-e4c9-89fb-847b-f27465e74a8c.jpeg "Quick_Start_-_Configure_-_H2O.jpg")


結果、普通のファイル配信はいけそう。
1.5.4(と現時点のmasterである1.6.0Bata)でリバースプロキシにホスト名を使ううと名前解決でSegFauiltしたので、一部機能が怪しくはある。

作ったイメージはDockerhubに置いた。 https://hub.docker.com/r/higanworks/h2o-busybox/

## h2o-busyboxの準備

手元がOSXなので、ビルドもDockerでいいや。

h2oを強引に単品バイナリとするため、StackOverflowさんから軽く調べた範囲で`CMakeLists.txt`にてオプションを追加する。
綺麗ではないがまずはお試し。


```Dockerfile(抜粋)
...
RUN echo 'SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a")' >> CMakeLists.txt
RUN echo 'SET(BUILD_SHARED_LIBRARIES OFF)' >> CMakeLists.txt
RUN echo 'SET(CMAKE_EXE_LINKER_FLAGS "-static")' >> CMakeLists.txt
RUN cmake -DWITH_BUNDLED_SSL=on -DBUILD_SHARED_LIBS=no .\
&& make h2o
...
```

Busybox戦略は、準備段階のDockerイメージについてサイズがいくらデカくても気にしなくていいのが良いね。

できたh2oバイナリをチェック。

```
root@66e2f5bd84a4:/h2o# ./h2o --version
h2o version 1.5.4
OpenSSL: LibreSSL 2.2.4

root@66e2f5bd84a4:/h2o# ldd ./h2o 
	not a dynamic executable
```

外部にリンクしてないようには見える。


## 取り出した`h2o`を`busybox`ベースのイメージに入れる

公式のイメージにADDするだけのお仕事です。

```Dockerfile(busybox-h2o)
FROM busybox:ubuntu-14.04
ADD build/h2o /bin/h2o
CMD ["/bin/sh"]
```

hubにpush。サイズちっさ。。ちっさ。。

![Docker_Hub.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/a066549e-7416-9ea2-4f0c-7b89c0717903.jpeg "Docker_Hub.jpg")


## 適当なWebサイトをコンテナに突っ込む。

ちょうどhugoで作ってるサイトがあったので、ビルドしたファイルを突っ込んでみよう。

```
sawanoboly.github.io/public/
├── 2015
│   └── 11
│       └── blog-hugo
├── css
├── js
├── post
└── tags
    ├── blog
    └── hugo

9 directories
```


マウントでもいいんやけどさ。昔の雑誌付録CD-R(※)みたいにサイトごと焼きこんでもいいかなと思って入れてみた。

(※あったよね？)

```Dockerfile(Webサイト用サンプル)
FROM higanworks/h2o-busybox:1.5.4

COPY h2o /h2o
COPY public /public
ENTRYPOINT ["/sbin/h2o", "-c", "/h2o/h2o.conf"]
```


```h2o.conf
listen:
  port: 8080
user: root
hosts:
  "default":
    paths:
      /:
        file.dir: /public
access-log: /dev/stdout
```

これでサイト入りh2o(配信機能付きサイトアーカイブ？)の準備は整った。

## ビルドして、起動してみる。

```
$ docker build -t sawanoboly.github.io .

...

Successfully built 5631bf462823
```

サイズ16MBしか無いぜ。なかなかポータビってるぞ。

```
$ docker images | grep sawanoboly.github.io 
sawanoboly.github.io              latest               5631bf462823        36 seconds ago      16.74 MB
```

早速起動だ。

```
$ docker run --rm -it -p 8080:8080 sawanoboly.github.io
[INFO] raised RLIMIT_NOFILE to 1048576
h2o server (pid:1) is ready to serve requests

# ブラウザでアクセス。

192.168.99.1 - - [01/Dec/2015:04:29:52 +0000] "GET / HTTP/1.1" 200 3725 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_1) AppleWebKit/601.2.7 (KHTML, like Gecko) Version/9.0.1 Safari/601.2.7"
192.168.99.1 - - [01/Dec/2015:04:29:52 +0000] "GET /css/blog.css HTTP/1.1" 200 2664 "http://192.168.99.100:8080/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_1) AppleWebKit/601.2.7 (KHTML, like Gecko) Version/9.0.1 Safari/601.2.7"
192.168.99.1 - - [01/Dec/2015:04:29:52 +0000] "GET /css/prism.css HTTP/1.1" 200 2356 "http://192.168.99.100:8080/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_1) AppleWebKit/601.2.7 (KHTML, like Gecko) Version/9.0.1 Safari/601.2.7"
192.168.99.1 - - [01/Dec/2015:04:29:52 +0000] "GET /js/prism.js HTTP/1.1" 200 40574 "http://192.168.99.100:8080/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_1) AppleWebKit/601.2.7 (KHTML, like Gecko) Version/9.0.1 Safari/601.2.7"
192.168.99.1 - - [01/Dec/2015:04:29:52 +0000] "GET /css/custom.css HTTP/1.1" 200 141 "http://192.168.99.100:8080/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_1) AppleWebKit/601.2.7 (KHTML, like Gecko) Version/9.0.1 Safari/601.2.7"
```

うむ、ちゃんと表示された。

![Sawanoboly_net_-_Sawanoboly_net.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/5d28ebff-67d5-5d99-8275-1a8ea904992c.jpeg "Sawanoboly_net_-_Sawanoboly_net.jpg")


このほか、次の機能についてはこのバイナリで問題なく動作することを確認。

- TLS&HTTP/2
- スレッド数指定



## この利用方法で既知の問題:`proxy.reverse`

順風満帆に見えたが、設定によって止まることがあった。

```
proxy.reverse.url: "http://127.0.0.1:8080/" # これは動く
proxy.reverse.url: "http://www.example.com:8080/" # これでSegFault
```

名前解決が必要なバックエンドだと、このコンテナは止まる。

```
h2o server (pid:1) is ready to serve requests
received fatal signal 11; backtrace follows
[0x445575]
[0x531070]
/lib/libnss_files.so.2(+0x4b2c)[0x7f5b1d094b2c]
/lib/libnss_files.so.2(_nss_files_gethostbyname4_r+0xd3)[0x7f5b1d095ca3]
[0x586033]
[0x58818b]
[0x407e51]
[0x52db84]
[0x58be19]
```

環境のせいだとは思う。
フル機能完全に使うには、Busyboxのベースイメージから調整するほうがいいのかな。。

さらにこの辺をStaticにするには、GCCから調整しないといけないのね。

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

## 動機の片鱗

急にアドベントカレンダーが来たのですこし関係有ることを追記してしめよう。

[SliceMediaっちゅーサービス](http://www.slice.media) で作れるサイトは一部をNginx+mruby配信のHTTP/2にしてある。
しかしスタイル等の共通パーツを[Zenlogic](https://zenlogic.jp)に置いてるのでHTTP/1.1になってる。

...HTTP/2は快適で一部やったら全部やりたくなる魅力にあふれている、特に装飾が多めのCMSとかで。スタイル系の部品を、h2oコンテナとVolumeの組み合わせで他所に移そっかなと思ってちょっと試している最中。

転送制限ないレンサバって(同一国内向けなら)Static配信役として向いてるので、HTTP/2が使えるようになるといいね。
