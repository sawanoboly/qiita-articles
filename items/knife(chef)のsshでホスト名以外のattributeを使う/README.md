<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

適当なサーバに`knife bootstrap`したあと、`knife ssh`などでなにかコマンド実行させよう。


`knife ssh`はデフォルトではホスト名で接続に行くので、IPが引けないとこうなる。

```
knife ssh "hostname:hogehoge" "uptime"
FATAL: 1 node found, but do not have the required attribute to stablish the connection. Try setting another attribute to open the connection using --attribute.
```

この場合、ohaiで取れる別の`attribute`を使うことが出来る。

```
knife ssh "hostname:hogehoge" "uptime" --attribute ipaddress
210.xxx.xxx.xxx  02:03:11 up 33 min,  2 users,  load average: 0.00, 0.00, 0.00
```

OK.
