<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Redisのsetというデータタイプは、メンバーからランダムに値を抽出可能ということだ。

適当に4つのメンバを入れます。

```
$ redis-cli sadd daizi:sore makenai
$ redis-cli sadd daizi:sore nagedasanai
$ redis-cli sadd daizi:sore nigedasanai
$ redis-cli sadd daizi:sore sinzinuku

$ redis-cli smembers daizi:sore
1) "sinzinuku"
2) "nigedasanai"
3) "nagedasanai"
4) "makenai"

```

`srandmember x 10,000回の試行`で結果を表示する。


```
$ redis-server -v
Redis server version 2.4.15 (00000000:0)
$ for x in {1..10000}; do redis-cli srandmember daizi:sore; done | sort | uniq -c
4975 makenai
1697 nagedasanai
1681 nigedasanai
1647 sinzinuku
```

結構偏ってんな！？ どゆこと。。

----
追記：特定のケースで偏る模様？
