
サーバに来ているSSL/TLSコネクションのクライアントバージョンを確認したい時、`ssldump`コマンドが使える。

Debian系ならaptで`apt-get install ssldump`、結構古い。


## クライアントバージョンをClientHelloでチェック

ヘッダ表示と逆引きしないオプションあたりを付けて、`ssldump`を実行してみた。
ClientHelloのバージョンから判断できる。

```
# ssldump -v
ssldump 0.9b3


# ssldump -n -H -i eth0 


## SSLV3の時はVersion 3.０
New TCP connection #1: xxx.xxx.xxx.xxx(64235) <-> xxx.xxx.xxx.xxx(443)
1 1  0.0651 (0.0651)  C>S  Handshake
      ClientHello
        Version 3.0 
        cipher suites

-- snip --

### TLS1の時はVersion 3.1
New TCP connection #3: xxx.xxx.xxx.xxx(64288) <-> xxx.xxx.xxx.xxx(443)
3 1  0.0651 (0.0651)  C>S  Handshake
      ClientHello
        Version 3.1 
        cipher suites
```

あと、`SSLv2 compatible client hello`を送ってくるのもいるので、そういう手合はその後のVersionで判断。



## opensslコマンドでクライアント役

挙動確認のクライアント役にはopensslコマンドをつかってもOK。

```
openssl s_client -connect xxx.example.com:443 -ssl3  ## ClientHello 3.0
openssl s_client -connect xxx.example.com:443 -tls1  ## ClientHello 3.1
openssl s_client -connect xxx.example.com:443        ## SSLv2 compatible Hello
```

