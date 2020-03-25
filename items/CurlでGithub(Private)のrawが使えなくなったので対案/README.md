<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


よくGithubプライベートリポジトリからcurlでrawデータを取って来てたりしたのです。

こんな具合に。

```shell:
curl -s -f -L -u 'LOGIN_NAME/token:API_TOKEN' https://raw.github.com/ORG_NAME/REPO_NAME/master/README.md
```

raw.githubはスクリプトをbashに渡したりするのに都合が良かったんですよね。

が、今朝(12/12/2013)トークンが古い形式だからか、rawの仕様が変わったのか、とりあえず使えなくなったことを私のNagiosが検知しました。


これはそろそろ真面目にAPIを呼ぶか、と幾らか試して次のやり方に落ち着きそう。

※ node.jsのjsonコマンドは使える前提、代わりも色々あるので置き換えて。

## base64コマンドがある場合
 
```shell:
curl -s -H "Authorization: token OAUTH_TOKEN" https://api.github.com/repos/ORG_NAME/REPO_NAME/contents/PATH_TO_CONTENT | json content | base64 -D
```

## base64コマンドがない場合

```shell:
curl -s -H "Authorization: token OAUTH_TOKEN" https://api.github.com/repos/ORG_NAME/REPO_NAME/contents/PATH_TO_CONTENT | json content | perl -MMIME::Base64 -ne 'print MIME::Base64::decode_base64($_)
```

api.githubはraw.githubとちがい、必ずJSONオブジェクトになるのでパースを一個はさもう。で、base64デコードと。
jsonコマンドはyajlなどのパッケージで何か代わりがあった気がします、


ちなみにブランチ/タグの指定はクエリでOK`?ref=master`

## Nagiosで監視しておく

このやり方がいつまで有効かは知らないので、Nagiosで監視しますわ。
check_httpのサンプルはこちら。

```shell:
check_http -H api.github.com -u /repos/MY_ORG/REPO_NAME/contents/README.md -S -k "Authorization: token OAUTH_TOKEN"
HTTP OK: HTTP/1.1 200 OK - 3366 bytes in 0.887 second response time |time=0.886742s;;;0.000000 size=3366B;;;0
```

