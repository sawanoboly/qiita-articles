

ちまたで話題のCircle CI https://circleci.com/ 。
やってみようと、あるRubyGemsのプロジエクトをCircle CIに突っ込みました。

とりあえず回したところ、テスト用のRubyバージョンは`circle.yml`によって1つを指定するタイプだった。

Travis CIはこんな感じで複数のRubyバージョンを指定できるんですよねー。

```yaml:.travis.yml
rvm:
- 2.0.0
- 1.9.3 
```

早速サポートに聞いてみました。

> We don't have very good support for build matrices at the moment. If you set up a build with 2x parallelization, you might be able to do something like:
circle.yml


と、すぐに案内してくれたワークアラウンドがこちら。

```yaml:circle.yml
dependencies:
  pre:
    - case $CIRCLE_NODE_INDEX in 0) rvm use 2.0.0 --default ;; 1) rvm use 1.9.3 --default ;; esac
test:
  override:
    - ruby -v ; bundle exec rspec: {parallel: true}
```

うおお。。そうきたか。
ちなみに、ここで外部シェルを呼ぶのはアカン模様。
テストの`ruby -v`部分は私が確認のため追加しました。なくてもOK。


> (中略)
> The {parallel: true} modifier at the end tells us to run the command on every node.
> Let me know if you need any help with that, and sorry about the hacky workaround.


『正直ハックですまんかった』とサポートの弁ですが、いやいや十分じゃあないですか。


## やってみた

コンテナをパラレルで起動して、コンテナ番号によってバージョンが別れる様子をレポート。


複数のコンテナを同時に起動するには、`Project Setting` > `Parallelism` の設定を変更します。


![Edit_settings_-_giraffi_ruby-orchestrate_io_-_CircleCI-3.png](https://qiita-image-store.s3.amazonaws.com/0/7454/e48a225d-72c1-c499-a63e-a430467de75c.png "Edit_settings_-_giraffi_ruby-orchestrate_io_-_CircleCI-3.png")


これでビルドすると。


![_33_-_giraffi_ruby-orchestrate_io_-_CircleCI.png](https://qiita-image-store.s3.amazonaws.com/0/7454/828a5842-cb84-4fed-15fd-7483f4bbc8f7.png "_33_-_giraffi_ruby-orchestrate_io_-_CircleCI.png")


おー、2コンテナで起動しました。

テストも別々の表示で、平行に進みます。

![_34_-_giraffi_ruby-orchestrate_io_-_CircleCI-4.png](https://qiita-image-store.s3.amazonaws.com/0/7454/3eef26c9-2d9e-e3b2-ac02-5f0dcd93aa7a.png "_34_-_giraffi_ruby-orchestrate_io_-_CircleCI-4.png")


コンテナ(0)はRuby2.0.0, コンテナ(1) はRuby1.9.3でテストが走りました！

![_34_-_giraffi_ruby-orchestrate_io_-_CircleCI-6.png](https://qiita-image-store.s3.amazonaws.com/0/7454/9c7927d8-c50a-9529-e7cb-bfbbb47d884c.png "_34_-_giraffi_ruby-orchestrate_io_-_CircleCI-6.png")




Rcovによるカバレッジも、ファイル名の後にコンテナ番号がついで別々に収集。


![_33_-_giraffi_ruby-orchestrate_io_-_CircleCI-2.png](https://qiita-image-store.s3.amazonaws.com/0/7454/1dae4e61-4d14-8a02-38f5-83f340c27396.png "_33_-_giraffi_ruby-orchestrate_io_-_CircleCI-2.png")


無事、複数RubyでやってくれというDevの要望に答えることができました。



コンテナIDで処理を分けるやり方は、他にも面白いことができそうですね。
