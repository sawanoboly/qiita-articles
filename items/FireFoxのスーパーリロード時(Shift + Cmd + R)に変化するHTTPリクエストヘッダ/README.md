<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


ふと気になったのでブラウザのスーパーリロード時にブラウザが投げるヘッダをチェックしてみた、FireFoxでは`Shift + Cmd + R`だ。

ああ、`Pragma: no-cache`が付いて、`Cache-Control:`が `no-cache`になると。httpサーバの設定でよく見るね。

### ただの`Cmd+R`

```bash:Cmd+R
$ echo hoge | nc -l 8080
GET / HTTP/1.1
Host: xxx.xxx.xxx.xxx:8080
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:22.0) Gecko/20100101 Firefox/22.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: ja,en-us;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
Connection: keep-alive
Cache-Control: max-age=0
```

### `Shift+Cmd+R`

```bash:Shift+Cmd+R
$ echo hoge | nc -l 8080
GET / HTTP/1.1
Host: xxx.xxx.xxx.xxx:8080
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:22.0) Gecko/20100101 Firefox/22.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: ja,en-us;q=0.7,en;q=0.3
Accept-Encoding: gzip, deflate
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
```
