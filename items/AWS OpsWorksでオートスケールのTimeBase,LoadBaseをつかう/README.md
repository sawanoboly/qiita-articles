<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[AWS OpsWorks](http://aws.amazon.com/jp/opsworks/)を使ってオートスケールを組み立てようと思い、用意されているスケール対応インスタンスタイプを調べてみました。


![inst01.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/2d9fe749-cd2b-2a90-eadc-1d58e758073f.jpeg "inst01.jpg")


ちなみにOpsWorksはCloudWatchのメトリクスと連携しているのでどのインスタンスでも`AutoHeal`オプションを使うことができます。Statusが何かおかしかったらインスタンスを作りなおしてくれるナイス機能です。


## 24/7

普通のレギュラーなインスタンスで、常時上がりっぱなしのインスタンスです。
通常はこのタイプを一つ用意して、`TimeBase`， `LoadBase`を増やしていく感じでしょう。

ただ、レイヤ内に`24/7`が１つもない状態で他のタイプがある場合は最低1台になるようにどれかが上がってくるようです。実は不要かもしれませんね。


## Time based instance

１時間単位で起動・停止の状態をスケジュールできるインスタンスです。**時間はUTC**で指定するので、お間違えの無いよう。

設定のUIはこんなかんじです、薄いのは曜日別設定アリ、濃い緑は`Everyday`のスケジュールです。

![time_base01.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/db489f61-cca5-79c9-ac76-1778c892457c.jpeg "time_base01.jpg")


設定後、UP指定の時間帯になったらStop状態だったインスタンスが勝手に上がって来ました。

![time_base02.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/1350845e-1cc3-ab0b-26f7-fb0dc5f0b44a.jpeg "time_base02.jpg")

デプロイまで完了したらELBに勝手に参加します、便利やなあ。


## LoadBased

CloudWatchとのメトリクス連携真骨頂ですね。
レイヤ内のインスタンスの**平均負荷状況**に閾値を設定し、指定時間内にその状態だったらインスタンスを起動します。

ルールはこんな感じです。

- ５分高負荷状態が続いたらインスタンス起動
- 10分低負荷になったらインスタンス停止
- 一度に任意の台数を追加・停止可能

![LBI01.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/fdb0af8a-df9b-fb28-1e05-5974abe4070b.jpeg "LBI01.jpg")




### 負荷をかけてみる

LoadBasedインスタンスを追加した後閾値を40%まで下げて、レイヤ内の1台でCPUベンチをまわしてみました。

丁度yumに`sysbench`があったのでぶっ放します。


```shell:sysbench_loop
$ while /bin/true ; do sysbench --num-threads=4 --test=cpu run ; done
```


![sysbench_agent.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/37f60b9c-597c-1497-e331-dd3bc72a3c7f.jpeg "sysbench_agent.jpg")


インスタンス2つのうち、1つが100％ベッタリになっているのでレイヤの平均負荷も良い具合に50％です。

![monitoring01.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/6901acd1-e859-438b-34e3-0d340b0ca488.jpeg "monitoring01.jpg")


では5分ほど待ってみます。

## 勝手にインスタンスが起動

![load03.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/9919cbea-558e-18f5-ed59-e337b4c085bc.jpeg "load03.jpg")


おお、レイヤの負荷に控えのインスタンスが駆けつける※様子が見られました！

※インスタンスストアでやると手軽ですが流石に遅いです、EBSにして一回くらい起動しておくとパッケージインストールなどを省けるぶんそこそこ早いかと思います。


### 平均負荷が軽減

3台目のインスタンスが起動したので、レイヤの平均負荷は狙い通り33％となりました、もちろんELBに参加してくれています。

![load04.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/4ee7db4a-7931-bbb5-28f4-c851ca33ff08.jpeg "load04.jpg")



## 終わりに

こういったオートスケールに対応するには、アプリケーション側の対応はもちろん、アプリケーションが動作するプラットフォームを、サーバに一切触らず構築する仕組みへの理解が大事になってきます。

今回試したのはStaticなWebサイトの配信だったので全く考える余地は無かったのですが、起動・デプロイ・後片付け・停止のサイクルをしっかり管理できるようなレシピを作るスキルを磨いていかないといけないですね。

