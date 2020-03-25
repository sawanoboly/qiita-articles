
以前HHVM3.6がリリースされた頃、こんなことをやった。[Loader.ioを使ってWordPressのバックエンド構成を変えつつ性能を測ってみる、php-fpmとHHVM](http://qiita.com/sawanoboly/items/e422b366439f67bfb01c)

で、HHVMのLTSとされる3.12.0がリリースされたので、またベンチを取ってみようという試み。前回とはWordPressおよびMySQLのバージョンも違うので、直接参考にはならないけど。


## まず通常のベンチマーク

AMIはこちら。[WordPress Powered by AMIMOTO (HHVM)](https://aws.amazon.com/marketplace/pp/B00V5JYXTO) インスタンスはc3.large。


ルールは前回と一緒。トップページと、管理パネルの記事一覧表示。

### トップページ

![normal_top.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/bdc9370d-4e24-b575-458d-6b8b80098922.jpeg "normal_top.jpg")

前回と比較。

| - |HHVM3.6|HHVM3.12|(参考)Nginx Cache|
| ---- | ---- | ---- | ---- |
|平均レスポンスタイム | 1566ms | 1405ms | 25ms |
|リクエスト処理数 | 5387 | 5995 | 346791 |

ちょっとだけ速い。。？ 誤差程度かな。そもそもキャッシュするしこんなもんで。


### 管理パネル

ログイン状態でも比較してみる。これも条件は前回と同じ。

![Admin.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/2eed8313-9f71-71f0-20f5-1fd261d07ac1.jpeg "Admin.jpg")


| - |HHVM3.6|HHVM3.12|
| ---- | ---- | ---- |
|平均レスポンスタイム | 8866ms | 4652ms |
|リクエスト処理数 | 1429 | 3273 |

おや、倍くらい速いぞこれ。何かしら効率が上がったとか、WordPressがそもそも早くなったのかは不明だが良いことだ。
Failも無いしね。


## 補足:RepoAuthoritative

HHVM3.8以降でPHPのコードを事前にバイトコードに変換するRepoAuthoritativeという機能がついた。

- [Use RepoAuthoritative](https://github.com/facebook/hhvm/wiki/performance-tuning)

> Enabling RepoAuthoritative mode involves pre-compiling all of your source code into a bytecode repo.

通常はリクエストに応じて生成するバイトコードのを、ウォームアップ代わりに事前に生成したい時に使う。

変換には`hhvm-repo-mode`コマンドをつかう。

※ `hhvm-repo-mode`コマンド、ソースでは`hphp/tools/oss-repo-mode`に入っている。binに無い場合でも、大抵パッケージのどこかに入っています。

これ、HHVMを止めないといけないので注意。

```
$ time sudo hhvm-repo-mode enable /var/www/vhosts/
Stopping hhvm:                                             [  OK  ]
running hphp...
parsing inputs...

...中略

analyzeProgram...
analyzeProgram took 0'00" (786762 us) wall time
parsing inputs took 0'02" (2681877 us) wall time
pre-optimizing...
pre-optimizing took 0'00" (614471 us) wall time
creating binary HHBC files...
creating binary HHBC files took 0'17" (17448057 us) wall time
all files saved in /tmp ...
running hphp took 0'21" (21824534 us) wall time
Starting hhvm:                                             [  OK  ]

Success!

real	0m21.997s
user	0m36.008s
sys	0m1.400s
```

先日microでやったらいつまでも終わらなかったが、c3.largeでは21秒で終わった。リソースはに余裕が必要そうだ。


## おわりに

HHVM3.12は、以前からかわらない安定度が保たれていた。どうやら高速になった部分もあるのかなー。

