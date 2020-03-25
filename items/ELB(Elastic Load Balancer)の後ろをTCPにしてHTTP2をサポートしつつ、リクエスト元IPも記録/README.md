
Nginxが1.9.5で[HTTP2モジュールをやや試験的ながら同梱](https://www.nginx.com/blog/nginx-1-9-5/)するようになりました。つかおう。

しかし、例えばNginxがELB(TLS終端)の後ろにいる場合、バックエンドへの接続をTCPにする必要があります。
で、TCPにするとアクセス元IPを通常ではログに記録できない。

ELBでは一応そういう時のために、Proxy Protocolを差し込んでくれる。NginxもProxy Protocolを取れるように設定できる。

- [ロードバランサーの Proxy Protocol のサポートを設定する - Elastic Load Balancing](http://docs.aws.amazon.com/ja_jp/ElasticLoadBalancing/latest/DeveloperGuide/enable-proxy-protocol.html "ロードバランサーの Proxy Protocol のサポートを設定する - Elastic Load Balancing")


## Nginxで普通のTLSとProxy Protocolを両方Listenする

通常の443ポートをProxy Protocolにすると動作確認とか面倒になるので、別ポートで設定する。
サンプルはこんな感じ。

```
    server {
        listen       443 ssl http2;
        listen       8443 ssl http2 proxy_protocol;
        server_name  _;  

        ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on; 
        ssl_ciphers AESGCM:HIGH:!aNULL:!MD5;
        ssl_session_cache   shared:SSL:10m;
        ssl_session_timeout 5m; 
        ssl_certificate /path/to/cert;
        ssl_certificate_key /path/to/key;
...
```

ログのフォーマットには`$proxy_protocol_addr`を追加しておく。ELB経由で渡ってきたクライアントIPがあれば記録される。

```
    log_format pp_combined '$proxy_protocol_addr $remote_addr - $remote_user [$time_local] '
                               '"$request" $status $body_bytes_sent '
                               '"$http_referer" "$http_user_agent"';
```


## Nginx直接、443ポートのHTTP/2をチェック

これは普通のクライアントから開ける。

```
$ curl -kI --http2 https://ec2-54-238-xxx-xxx.ap-northeast-1.compute.amazonaws.com/
HTTP/2.0 200
server:nginx/1.9.5
date:Fri, 16 Oct 2015 09:58:08 GMT
content-type:text/html
content-length:612
last-modified:Wed, 14 Oct 2015 06:23:09 GMT
vary:Accept-Encoding
etag:"561df4cd-264"
accept-ranges:bytes
```

アクセスログには次のように記録されたね。

```
119.104.xxx.xxx - - [16/Oct/2015:09:58:08 +0000] "HEAD / HTTP/2.0" 200 160 "-" "curl/7.44.0"
```

## Nginx直接、8443ポートをチェック

cURLついでに8443も叩いてみよう。`Unknown SSL protocol error`と言われる。

```
$ curl -kI --http2 https://ec2-54-238-xxx-xxx.ap-northeast-1.compute.amazonaws.com:8443
curl: (35) Unknown SSL protocol error in connection to ec2-54-238-xxx-xxx.ap-northeast-1.compute.amazonaws.com:8443
```

サーバのログでは、`broken header`とされちゃう。

```
2015/10/16 09:59:24 [error] 11#0: *1 broken header: "?????,????F??Ŧ?)`t???B?N???0?,?(?$??
????kjih9876?????2?.?*?&???=5" while reading PROXY protocol, client: 119.104.xxx.xxx, server: 0.0.0.0:8443
```

## ELBを 443=>8443としつつ、Proxy Protocolを有効に

じゃあ TCP/443=>TCP/8443 のELBを用意しよう。したね。

```
Port Configuration: 443 (TCP) forwarding to 8443 (TCP)
```

8443へのリクエストに対して、Proxy Protocolを使うポリシーを当てよう。

- [ロードバランサーの Proxy Protocol のサポートを設定する - Elastic Load Balancing](http://docs.aws.amazon.com/ja_jp/ElasticLoadBalancing/latest/DeveloperGuide/enable-proxy-protocol.html "ロードバランサーの Proxy Protocol のサポートを設定する - Elastic Load Balancing")

TLSをほどかないのでとりあえずProxyProtocol用の1つがついた。

```
...
"BackendServerDescriptions": [
     {
         "InstancePort": 8443, 
         "PolicyNames": [
             "my-ProxyProtocol-policy"
         ]
     }
 ], 

...
```

## ELB経由、443ポートのHTTP/2をチェック

ではNginxにしたように、ELBにcURLしよう。ちゃんとHTTP/2.0だ。

```
$ curl -kI --http2 https://http2-test-1298257833.ap-northeast-1.elb.amazonaws.com
HTTP/2.0 200
server:nginx/1.9.5
date:Fri, 16 Oct 2015 10:08:00 GMT
content-type:text/html
content-length:612
last-modified:Wed, 14 Oct 2015 06:23:09 GMT
vary:Accept-Encoding
etag:"561df4cd-264"
accept-ranges:bytes
```

アクセスログには、リクエスト元のクライアントとELBの内側アドレスが両方記録される(※ログフォーマット次第)。

```
119.104.xxx.xxx 10.134.140.5 - - [16/Oct/2015:10:08:00 +0000] "HEAD / HTTP/2.0" 200 160 "-" "curl/7.44.0"
```

これでよし。


## ついでにngx_mrubyでの`proxy_protocol_addr`の取り方

`Nginx::Request#var`経由でとれる。

```
location /hello {
  mruby_content_handler_code '
    r = Nginx::Request.new
    Nginx.echo "Your IP is #{r.var.proxy_protocol_addr}"
  ';
}  
```

ELB経由で確認っと。

```
$ curl -k --http2 https://http2-test-1298257833.ap-northeast-1.elb.amazonaws.com/hello
Your IP is 119.104.xxx.xxx
```



## おわりに

ELBにTLS終端を任せるのは色々と都合がよいので、それを諦めてでも俺はWebサイトをHTTP/2にしたいぜ！ って時に。
まあHTTP/2自体、いつのまにかELBがサポートしてそう。
