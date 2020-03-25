

## http-dos-detectorとは？

任意の条件でいわゆるDDos攻撃を判定し、ブロックなり当局に通報するなり、適切な処理を施すための実装だ。

https://github.com/matsumoto-r/http-dos-detector

Apache httpd用のmod_mrubyと、Nginx用のngx_mrubyはそれぞれの差異を抽象化して吸収し、同じmrubyコードを使用できるように作られている。
この`http-dos-detector`は同じコードで両方対応のデモとしてもよくできている。パラメータの意味などは作者ブログへ。

[nginxでもapacheでも使用可能なDoS的アクセスを検知して任意の制御をするWebサーバ拡張をmrubyで作った](http://hb.matsumoto-r.jp/entry/2015/06/08/224827)



## 処理性能への影響をみたよ。


|Env|Req/sec|Time per request(ms)|
|----|----|----|
|mruby_access_handlerなし |2177.34|13.778 |
|mruby_access_handlerあり、ブロックしない |1522.99|19.698 |
|mruby_access_handlerあり、ブロックする |1515.88|19.790 |

3割くらいあったね。割合よりは、リクエストあたり6msくらい時間が増える、のほうが目安にするにはよいかも。
とりあえず今日の測定結果を残しておけば、またいつか測りたくなったときに比較できる。


### 試した環境

ハードはMacBook Pro 2014特盛り x Boot2Docker。ngx_mrubyの公式においてあるDockerfileちょっといじってます。

Nginxのビルドは次のように。ngx_mrubyはタグ`v1.10.12`がついてる奴。

```
nginx version: nginx/1.8.0
built by gcc 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04) 
built with OpenSSL 1.0.1f 6 Jan 2014
TLS SNI support enabled
configure arguments: --add-module=/usr/local/src/ngx_mruby --add-module=/usr/local/src/ngx_mruby/dependence/ngx_devel_kit --with-debug --with-http_stub_status_module --with-http_ssl_module --prefix=/usr/local/nginx --with-http_realip_module --with-http_addition_module --with-http_sub_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --add-module=/usr/local/src/ngx_cache_purge-2.3
```

worker_processesはautoにしたので8個かな。

abはこのくらい。 `ab -n 6000 -c 30 http://`boot2docker ip`:8080/404.html`

## detectorなしのab結果

なしといっても、クラスのロードとかは入れており、locationにて`mruby_access_handler`を入れていない状態でかるくabしてみた。

```
Server Software:        nginx/1.8.0
Server Hostname:        192.168.59.103
Server Port:            8080

Document Path:          /404.html
Document Length:        4 bytes

Concurrency Level:      30
Time taken for tests:   2.756 seconds
Complete requests:      6000
Failed requests:        0
Total transferred:      1392000 bytes
HTML transferred:       24000 bytes
Requests per second:    2177.34 [#/sec] (mean)
Time per request:       13.778 [ms] (mean)
Time per request:       0.459 [ms] (mean, across all concurrent requests)
Transfer rate:          493.30 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    4   2.8      3      18
Processing:     1   10   4.4      9      30
Waiting:        1   10   4.4      9      30
Total:          2   14   4.6     13      33

Percentage of the requests served within a certain time (ms)
  50%     13
  66%     15
  75%     17
  80%     17
  90%     20
  95%     22
  98%     25
  99%     27
 100%     33 (longest request)
```

## detectorあり、ブロックなしのab結果

threshold_counterを十分大きな値にして、カウントはするけどブロックまではしない。
locationにて次のようにdos_detectorを通るようにした。

```
        location = /404.html {
            mruby_access_handler conf/dos_detector/dos_detector.rb cache;
            root   html;
        }
...
```

abでは性能にそれなりの変化があった。

```
Server Software:        nginx/1.8.0
Server Hostname:        192.168.59.103
Server Port:            8080

Document Path:          /404.html
Document Length:        4 bytes

Concurrency Level:      30
Time taken for tests:   3.940 seconds
Complete requests:      6000
Failed requests:        0
Total transferred:      1392000 bytes
HTML transferred:       24000 bytes
Requests per second:    1522.99 [#/sec] (mean)
Time per request:       19.698 [ms] (mean)
Time per request:       0.657 [ms] (mean, across all concurrent requests)
Transfer rate:          345.05 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    4   3.0      3      17
Processing:     2   16  15.8     12     269
Waiting:        2   16  15.8     12     269
Total:          4   19  15.6     17     270

Percentage of the requests served within a certain time (ms)
  50%     17
  66%     19
  75%     21
  80%     22
  90%     26
  95%     33
  98%     69
  99%    105
 100%    270 (longest request)
```


## detectorあり、ブロックする場合のab結果

5秒で1,000リクエストのルールでブロックしてみた。次のようなコンフィグ。

```
config = {
  :counter_key => r.hostname,
  :magic_str => "....",

  :behind_counter => -500,

  :threshold_counter => 1000,
  :threshold_time => 5,

  :expire_time => 60,
}
```


`Non-2xx responses => 2000`ってことで、しっかりブロックされているのがわかる。
ブロック回数が5,000でないのは`behind_counter`が消費されたからなのでOK。

```
Server Software:        nginx/1.8.0
Server Hostname:        192.168.59.103
Server Port:            8080

Document Path:          /404.html
Document Length:        4 bytes

Concurrency Level:      30
Time taken for tests:   3.958 seconds
Complete requests:      6000
Failed requests:        2000
   (Connect: 0, Receive: 0, Length: 2000, Exceptions: 0)
Non-2xx responses:      2000
Total transferred:      2214000 bytes
HTML transferred:       914000 bytes
Requests per second:    1515.88 [#/sec] (mean)
Time per request:       19.790 [ms] (mean)
Time per request:       0.660 [ms] (mean, across all concurrent requests)
Transfer rate:          546.25 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    4   2.9      3      18
Processing:     2   16  15.4     13     264
Waiting:        2   16  15.4     13     264
Total:          5   20  15.3     17     265

Percentage of the requests served within a certain time (ms)
  50%     17
  66%     19
  75%     21
  80%     23
  90%     27
  95%     32
  98%     69
  99%     94
 100%    265 (longest request)
```


## それじゃあつかっとく

今回採用を目論む環境ではバックエンドがわりと遠くにあるし元々のアクセスも多くはないので、このくらいの性能差は許容範囲だ。

このNginxの前には色々いるので、`counter_key`はつど判断でいーかな。

> 追記: 条件をちょっと直した。

```
config = {
  :counter_key => r.headers_in["X-Forwarded-For"] ? r.headers_in["X-Forwarded-For"].split(",").first : r.headers_in["X-Real-IP"],
  :magic_str => "....",

  :behind_counter => -5000,

  :threshold_counter => 1000,
  :threshold_time => 5,

  :expire_time => 60,
}
```
