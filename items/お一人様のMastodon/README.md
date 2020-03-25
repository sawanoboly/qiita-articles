
[Mastodon](https://github.com/tootsuite/mastodon)を見ていたら、`SINGLE_USER_MODE`という設定項目があった。

- トップページは`アカウントID=1`のストリームに固定(リダイレクト)される
- `/about`からサインアップできない。(入力はできるけど、トップに戻り何も起きない)
- Adminの管理はできる

公開インスタンスリストにも3つばかりSingleな奴がいますし、作ってもいいんじゃないかと。

[List-of-Mastodon-instances](https://github.com/tootsuite/documentation/blob/master/Using-Mastodon/List-of-Mastodon-instances.md)


## SINGLE_USER_MODEをセットアップする

まずは普通に立ち上げよう。準備。

- cp .env.production.sample .env.production
- `vim` .env.production
  - `rake secret`を埋めたり。
  - 手元でやってみる場合、LOCAL_DOMAINはとりあえず`localhost:3000`を入れとくとよいです。

Rails的な色々をしつつ起動。

- docker-compose build/up
- rake db:migrate
- rake assets:precompile
  - Webにスタイルが当たってなかったらVolume失敗してるのでdocker-compose upしなおせばOK

サインアップします。

http://localhost:3000/about で適当に入力。メールは送信できなくてもよい。

### メール強制confirmationと管理者に昇格

サインアップ後のメール認証はタスクでもできるのでします。お一人なので管理者にもしておきましょう。

```
docker-compose run --rm web rails mastodon:confirm_email USER_EMAIL=sawanoboly@example.com
docker-compose run --rm web rails mastodon:make_admin USERNAME=sawanoboly
```

一旦railsのコンテナを止めて、

設定から`SINGLE_USER_MODE=true`とします。

これでもう一度起動しすれば、たいていのアクセスは`http://localhost.local:3000/@sawanoboly`にリダイレクトされるようになります。

![sawanoboly_-_Mastodon.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/2e727edc-4228-6993-a527-2177fb4ea335.jpeg "sawanoboly_-_Mastodon.jpg")


Tootするときには、 http://localhost:3000/auth/sign_in に直接訪れれば自分はログインできるので`/web/*`でやりましょう。

![Mastodon.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/1350b70d-3e00-c22f-e07d-211beea489a6.jpeg "Mastodon.jpg")

- 管理画面には`Open registration`の項目があるので、`Disabled`にすれば登録フォームが消えます。

フェデレーション用のTCP/4000はどうなるのかまでを見てないですが(試すにはもう1セット立ち上げればよいのかな？)、シングル運用のインスタンスでもフォロー・フォロワーの数はあるしなんとかなるんでしょう。



## 運用


スキーマ定義に`enable_extension "plpgsql"`ってあったから、RDBはPostgreSQLしかダメなのかなってなると、ホスティングしてるPaaSがよいなあと思う次第。
SideKiq用のRedisもいるし、公式デモに習ってscalingoやらherokuが楽かなって思いますが、web+workerになるのはなーと。

画像などはローカルのほかにAmazon S3が使えます。S3のほうは互換性があれば他でもよさそう(rubygemsのPaperclipを使ってた。)。`S3_ENDPOINT(API),S3_HOSTNAME(配信ドメイン)`が環境変数から指定できるようになっているので、国内でも色々選択肢はあったはず。
手間とお金が安く運用できそうな組み合わせを考えてみるのもよいね。

ちなみにメールはサンプル設定にもあるように、Mailgunで不自由しないと思います。それか場所によってはSMTP_METHODをsendmailにしときゃ十分。


> 意外と反応が多いので追記: 
>
> docker-composeでそのまま起動して運用もそこそこできますが、補足しておきます。
>
> - docker-compose.ymlでは、volumeのコメントアウトを外して、(少なくとも)volumeをローカルのホストに書き出しておくと`docker-compose rm`しても残ります。
> - 3000, 4000はこのようにnginxでリバースプロキシをつけましょう。
>   - https://github.com/tootsuite/documentation/blob/master/Running-Mastodon/Production-guide.md
>   - 証明書はサンプルにもあるようにLetsEncryptでやるとよいです。
> - この例そのまま、localhostでやっては意味がないので、それがよくわからなかったらやめとく方が無難です。
> 

なお、データをエクスポートできるようなので飽きたら止めて移住できそう。フォロワーから見てドメインがどうなるんだろうという点は気になりますけど。
