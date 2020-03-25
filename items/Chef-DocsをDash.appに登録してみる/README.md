<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

が、イマイチだった。

MacのアプリDashというドキュメントビュワーで、[OpscodeのChef Docs](http://docs.opscode.com)が読めるといいかもしれないと思い挑戦。
Dash.appについてはこの辺で紹介されてます。 => [Dash | naoyaのはてなダイアリー][naoya_dash]

[naoya_dash]: http://d.hatena.ne.jp/naoya/20130218/1361171277

## `doc2dash`のセットアップ

ChefDocsはSphinxプロジェクトなので、Dashが用意している[doc2dash][doc2dash_url]ユーティリティを使えるようセットアップ。

[doc2dash_url]: https://pypi.python.org/pypi/doc2dash

OSX標準で長らく触ってなかったpip(1.0.2)がマトモに動かなかったのでとりあえずbrewのを使うことに。

```shell:install_doc2dash
$ brew install python
$ brew link --overwrite python

$ /usr/local/bin/pip install --user doc2dash
$ /usr/local/bin/pip install --user sphinx

$ doc2dash --version
doc2dash 1.1.0
```
OK。

## ChefDocsをローカルでビルドする

GithubからChefDocsを取ってきます。

`$ git clone --depth 1 https://github.com/opscode/chef-docs.git `

たまに`sphinx-build`がFailするコミットがあるようなので、一応コミットログをみて安定してそうなリビジョンをCheckOutしましょう。

```shell
$ cd chef-doc
$ export PATH="/usr/local/share/python:$PATH"
$ make master
```

ビルドは時間がかかるので、しばらくほっときましょう。

### genindexの捏造

`doc2dash`ユーティリティでsphinxプロジェクトをdashに取り込むためには索引の`genindex.html`が必要ですが、Chef Docsのプロジェクトではこれを生成するようになっていません。
`conf.py`を編集すれば`genindex.html`を作れるのですが、元々のファイルがindexに登録する単語を定義してないので、生成しても空っぽです。

仕方ないので一旦トップページを使いまわすことにしてコピーします。

`$ cp build/index.html build/genindex.html`


## Dashに登録

`build/`以下に出力されたhtml達を`doc2dash`ユーティリティを使用してDashに登録します。

`$ doc2dash --name 'Chef Docs' -A build --icon ./images/opscode_chef_html_logo.png`

![Dash.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/60c02423-bec3-d28b-126c-e4d4b6477425.jpeg "Dash.jpg")


Okayです。

## Dashで閲覧

登録されたドキュメントをDashで開いてみます、見た目は良い感じです。


![Dash2.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/92cc8916-2976-a67e-3d47-af3b533576e1.jpeg "Dash2.jpg")


### 既知の残念

ローカルなので閲覧はサクサクですが、Dashならではというところまでは至らず。 変換時に何か仕込めば良いのでしょうが。

- 検索不可
- モジュール一覧なし
- クリックしたらWebに飛んじゃう箇所がいくつか

要はhtmlファイルをブラウザで見るのと変わらないので、docsetsの作り方を理解してもう少し便利にしたいところ。
とりあえず右上の`index`から索引を開く運用でなら使えるかも。
