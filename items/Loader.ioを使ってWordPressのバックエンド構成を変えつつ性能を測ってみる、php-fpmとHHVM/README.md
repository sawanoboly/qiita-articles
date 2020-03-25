
[Loader.io](https://loader.io/)をそのうち試そうとしていたところに、丁度いい相談があったのでやってみた。
プラットフォームの要素を変更したWordPressでパフォーマンスを比較したいとのことだ。

## HipHopVM(HHVM)のAmazon Linux用RPMを作った

> [HHVM](http://hhvm.com/ "HHVM")はFacebookの言語Hackの実行エンジン。細かい話は置いといて、HHVMのバージョン3.6は5.6相当のphp実装でもある。

Amazon Linuxをベースにしつつ、HHVMでWordPressを動かしたいと言われ、RPMでだれでも使えるようにしていいならしばらく面倒見てもいーよと。

で、test-kitchenでRPM作れるようにして、walter-cdでポイっとリリースできるようにしたのが次のリンク。

[OpsRockin/hhvm-for-amazon-linux](https://github.com/OpsRockin/hhvm-for-amazon-linux "OpsRockin/hhvm-for-amazon-linux")


できたものはpackagecloudで配布中、と。

[opsrock-hhvm/hhvm-stable - Packages](https://packagecloud.io/opsrock-hhvm/hhvm-stable "opsrock-hhvm/hhvm-stable - Packages")


## HHVMのRPMを元に、AMIMOTOシリーズにHHVMをラインナップしてもらった

少し前からHHVMに興味津々だったWordPressのAMIMOTOにRPMを渡すと、ほどなくHHVMを使ったWordPressがAmazon MacketPlaceに並びました。
既存のHVM版との要素はこんな感じで違います。 HVMとHHVMとタイトルが付いておりすこし紛らわしい。

| - |[AMIMOTO HVM](https://aws.amazon.com/marketplace/pp/B00LWHVJH8/)|[AMIMOTO HHVM](https://aws.amazon.com/marketplace/pp/B00V5JYXTO)|
| ---- | ---- | ---- |
|Virtualization Modes| HVM | HVM |
|HTTPエントリ | nginx | nginx |
|upstream | fastcgi |fastcgi |
|PHPランタイム| **php-fpm** (5.5) | **hhvm** (5.6相当) |
|DB| Percona | Percona |


これをそれぞれインスタンスタイプ`c3.large`で起動し、WordPressのインストールとloader.ioのverifyまで済ませる。
nginxのキャシュプラグインが入っているので、これはOFF。

リージョンはloader.ioに近そうなので`us-east-1`を選択。


## Loader.ioのリクエストセッティング

普段こういうのは大抵httperfあたりを使うんですが、Loader.ioは結果をシェアしやすいのと、地味に性能が必要なクライアントマシンと回線の用意が無いのでやってみた。

無料プランなのでテストは1分間、上限は10000クライアント。ホストを変えると結果のアーカイブがクリアされちゃうのは仕方ないところか。

- Duration: 1分
- Type: 期間中2クライアント数を0-300まで増加(Maintain client load)

比較は2つの指標で。

### トップページ表示

とりあえずトップページに対してクライアントを増やしながら全力アクセス。これは特に説明不要か。

![スクリーンショット_4_1_15__1_41_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/88cc59ba-1373-faca-ab63-44cfc30a75c0.png "スクリーンショット_4_1_15__1_41_AM.png")

nginxキャッシュを切っているので、毎度毎度トップページを生成させます。


#### php-fpm版に対して実行

まずはphp-fpmのAMIMOTOに対して実行。

![スクリーンショット_4_1_15__1_51_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/1375bc7b-56ad-0583-8e99-db21927c79bf.png "スクリーンショット_4_1_15__1_51_AM.png")

クライアントの増加に対し、ほぼ線形に平均のレスポンスタイムも遅延していくのがよく分かる。


#### hhvmに対して実行

つづいてhhvmのAMIMOTOに対して実行。

![スクリーンショット_4_1_15__1_55_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/e570d436-c4ca-e944-ec92-604ccaae619b.png "スクリーンショット_4_1_15__1_55_AM.png")

同じように見えるが、青軸のスケールが違うね。次で比較しよう。


#### トップページ簡易比較

分かりやすい所をピックアップしてみた。

| - |[AMIMOTO HVM](https://aws.amazon.com/marketplace/pp/B00LWHVJH8/)|[AMIMOTO HHVM](https://aws.amazon.com/marketplace/pp/B00V5JYXTO)| (参考)nginxキャッシュ有効 |
| ---- | ---- | ---- | ---- |
|平均レスポンスタイム | 3108ms | 1566ms | 15ms |
|リクエスト処理数 | 2637 | 5387 | 386830 |

同じインスタンスタイプだけども、処理数が倍違う。


### WordPressの管理画面の記事一覧を表示

Loader.ioでも複数のリクエスト間でサーバレスポンスの値を使えます。WordPressはCookieが使えればいいみたいなので、次の2つ

1. 管理画面ログイン (POST /wp-login.php )
2. 記事一覧表示    (GET /wp-admin/edit.php )

これならphpの処理系としての実力がさらに顕著に出るかも？

厳密には1で返されるリダイレクト先(`/wp-admin/`)にもリクエストを送るようなので3回づつリクエストが発生していたけど、リダイレクト時にCookieは持ち回ってくれないみたい。

ツールにWordPressでログインの処理をさせる時は、Post用のパラメータとして次の項目を埋めればOKだった。

- log : ユーザ名
- pwd : パスワード
- wp-submit : `Log+In`
- testcookie : 1 (いらないかも)

Cookieの使い回し方はLoader.ioのヘルプで。

[Variables - Loader.io Help Desk](http://support.loader.io/article/18-variables "Variables - Loader.io Help Desk")

![スクリーンショット_4_1_15__2_45_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/6e6c19b7-4287-d2f4-2bec-0b00977cf453.png "スクリーンショット_4_1_15__2_45_AM.png")



ついでに、管理画面は重たい事がわかっているのでレスポンスが50xになってしまってもベンチマークが中断しないようにエラー許容レートとタイムアウトを伸ばしておく。

![スクリーンショット_4_1_15__2_32_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/24a01498-5859-52f6-4ed8-5c23d50e115c.png "スクリーンショット_4_1_15__2_32_AM.png")

増やした理由はすぐに。


#### php-fpm版に対して実行

では、ログインと管理用データ取得という重たげな処理をやってみよう。まずphp-fpm。


![スクリーンショット_4_1_15__2_52_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/b0ecddbe-7556-3376-b2d4-93831c859024.png "スクリーンショット_4_1_15__2_52_AM.png")

レスポンスタイムが途中から劇的に早くなった？？ のではなく、150−200クライアントのあたりでエラーで落ちている。

次の図、レスポンスコードの推移が分かりやすい。

![スクリーンショット_4_1_15__2_52_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/0aed2e24-9881-6985-8a8f-9e6e06297139.png "スクリーンショット_4_1_15__2_52_AM.png")

php-fpmのエラーログには幾らか原因が出ていたので、チューニングの余地はあるかもしれないが。。まあ大抵こうなるよねと。


#### hhvmに対して実行

同じことをhhvmに対しても実行した。

![スクリーンショット_4_1_15__2_48_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/f7c15905-5e64-78b7-5632-1bd93739f7d9.png "スクリーンショット_4_1_15__2_48_AM.png")

レスポンスタイムこそ伸びた(defaultの10秒ではタイムアウト扱い)ものの、エラー無しで完走。ちょっとこれ内部の取り回しどうなってんだろと思いました。

hhvmはこの後さらに数千までクライアントを増やしたが、先にnginxのパラメータチューニングが必要だったほど安定していました。


### さいごに

hhvmは早い上に頑丈だった。PHPと完全に互換があるわけではないらしいということだけど、採用できるPHPアプリケーションなら検討もしくは対応していいとおもう。
Amazon Linux用のHHVM-RPMはビルド関連もオープンなので、オプションなどリクエストがあればフィードバックもらえると多分対応します。

ベンチマークの裏でサーバにログインしてモニタリングしていると、どっちもCPUコア2つのうち片方しか使えてなかった感じなので、そこらもっと改良できるかも。

Loader.ioは定期的に実行しつつSlackに結果を通知してくれたりするので、たまに本番環境の健康診断とか、ちょっとした環境の評価に使うのもいいですね。

![スクリーンショット_4_1_15__3_40_AM.png](https://qiita-image-store.s3.amazonaws.com/0/7454/b700fe0e-f95d-c70b-10de-b921e04efcd0.png "スクリーンショット_4_1_15__3_40_AM.png")

----

追記：

本記事の画像はこちらにも提供しました。 >> [網元HHVMの提供開始 | 超高速 WordPress AMI 網元](http://ja.amimoto-ami.com/2015/05/07/amimoto-hhvm/)
