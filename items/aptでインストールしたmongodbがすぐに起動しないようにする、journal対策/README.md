<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


10genが配布しているmongodb、[aptなどで導入](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-debian-or-ubuntu-linux/)できて楽なんだが`Ver.2.0`以降ではjournalがデフォルトで有効になっている。

journal有効『= aptで入れた瞬間**3GBのDisk確保**に走る』ため(`/var/lib/mongodb`以下)、導入対象によっては結構な地雷となるのでこれを防ぐ。


## 起動スクリプトを確認する
Upstart用の起動スクリプトを見ると、`ENABLE_MONGODB`が文字列"yes"ならば起動するとあり、直後に`default`から設定を取得しようとしている。

```shell"/etc/init/mongodb.conf
-- snip --
script
  ENABLE_MONGODB="yes"
  if [ -f /etc/default/mongodb ]; then . /etc/default/mongodb; fi
  if [ "x$ENABLE_MONGODB" = "xyes" ]; then exec start-stop-daemon --start --quiet --chuid mongodb --exec  /usr/bin/mongod -- --config /etc/mongodb.conf; fi
end script
-- snip --
```

## 導入直後の起動を制限する
読むふりをしつつも`/etc/default/mongodb`はインストールパッケージに含まれていないので、なんでも良いので用意する。

```shell:/etc/default/mongodb 
ENABLE_MONGODB=断る
```

上記ファイル設置の後、`apt-get install mongodb-10gen`としてインストールすれば勝手には起動してこない。

