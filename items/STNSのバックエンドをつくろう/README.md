この夏、あるシステムでSTNSを使えそうな気がしました。

STNSとは何か、ですって？ なら落ち着いて、[時代が求めたSTNSと僕 // Speaker Deck](https://speakerdeck.com/pyama86/shi-dai-gaqiu-metastnstopu) を読みましょう。

[![時代が求めたSTNSと僕____Speaker_Deck.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/b085f587-16fa-655d-1dd7-b8c4a6ffaaa9.jpeg "時代が求めたSTNSと僕____Speaker_Deck.jpg")](https://speakerdeck.com/pyama86/shi-dai-gaqiu-metastnstopu)

OK、読みましたかね。

で、ユーザのDBにSaaSの何かを使えればいいなとおもってバックエンドを書けるかなという試み。時代とはつまり私だった。

この記事で出てくるコードを置きました。 https://github.com/sawanoboly/stns_backend-sinatra-example

## STNSのバックエンド仕様を調べました

> 注: こちら、記事作成時点のSTNS APIバージョン1.0の仕様についての調査です。※バックエンドの指定にv2をつけなければそのまま使えます。
> API v2.1(またはそれ以降)については http://stns.jp/en/interface

STNS製作者の[P山さん](https://twitter.com/pyama86)曰く、

<blockquote class="twitter-tweet" data-lang="ja"><p lang="ja" dir="ltr">バックエンドなんでもいんすよーって言いながら全くインターフェースのドキュメント書いてないアカウントがこちらです。</p>&mdash; P山 (@pyama86) <a href="https://twitter.com/pyama86/status/760825621199085568">2016年8月3日</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

ソースを読もうかと思いましたが、現状では自分の都合でGoを読むよりRubyで書いてみるほうが早い。

というわけで次の手順でじわじわと調査。STNSの設定等は公式サイト [STNS: Simple TOML Name Service](http://stns.jp/) を参照しましょう。

1. STNSをインストール、設定して起動。※nscdは使わない
2. 空のsinatra(Ruby)サーバを起動。
3. `libnss_stns.conf`のエンドポイントをsinatraサーバに向ける。
4. idコマンドを実行する。
5. sinatraに来たリクエスト(404)のパスを確認。
6. stns(`tcp:1104`)にcurlで聞く(`例: curl http://localhost:1104/user/name/example`)。
7. レスポンスを真似してsinatraに実装する。(とりあえず固定文字列で返しちゃえばよい)
8. 4-7を適当に繰り返す。
9. sinatra側をバックエンドにidコマンドとsshログインがうまくいったら終了。

疎結合ばんざい。


## STNSバックエンドに必要だったもの

|path|response|
|----|----|
|/user/list|ユーザ(`/etc/passwd`に相当)の一覧を返す|
|/user/name/:username|ユーザ名から、詳細を返す|
|/user/id/:uid|ユーザIDから、詳細を返す|
|/group/list|グループ(`/etc/group`に相当)の一覧を返す|
|/group/name/:groupname|グループ名から、詳細を返す|
|/group/id/:gid|グループIDから、詳細を返す|


`/*/id/:id`では、IDをかぶらせているケースは先勝ちっぽい。

## データを適当に持つ

当初の目的ではAmazon DynamoDBにデータをもたせる予定だったけど、ひとまずバックエンド仕様の理解を優先するためファイルにした。

ちなみにこれもSTNSにまずデータを作り、ユーザ(`/user/list`)とグループ(`/group/list`)をcurlでSTNSから取得してYamlに変換。まあこれで互換性あるっしょ。

ユーザをこんな感じでー。

```yaml:users.yml
---
example:
  id: 1001
  password: ''
  hash_type: ''
  group_id: 1001
  directory: "/tmp"
  shell: "/bin/bash"
  gecos: ''
  keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDi+pWex+o7Urn6uKJXeVrFv/V4+/y4Kpu51JtEiTEp4k12ASYhCJ28dbSi/VcowJEjs2Ll6h7doQ+4VLUx7kZH11cZ7cKemRotnLVOQYJhY2C9bLRwj3MgKNP8o3Kiv67uM6ppH7RbqKsmdXlkCIOfLSDKOe5AnRGROLZtC9mr3dvhdDlUHXVUlbeH0loXCA41acwv8Zt+LwYH3Qpdo+Wm25YREPfmBYIG+I5ogWS8wE1igckQbvDCadGzMPqkOjF8Gzs+BxC9q6Fke92bYXIwdQBgVLYjETtWspDUDrEvYPxteh6hXgW3FxCUyCN6l/q5kbtvBq+Ca+4XPjLo2M+t
  link_users: null
example2:
  id: 1002
  password: ''
  hash_type: ''
  group_id: 1002
  directory: "/home/example2"
  shell: "/bin/bash"
  gecos: ''
  keys:
  - ssh xxxxxxx
  link_users: null
```

グループをこう。

```yaml:groups.yml
---
example:
  id: 1001
  users:
  - example
  link_groups: null
example2:
  id: 1002
  users:
  - example2
  link_groups: null
```




## そしてsinatraへ

じゃあsinatraをyamlからデータ取得するように実装しなおす。

```ruby:app.rb
require 'sinatra/base'
require 'yaml'
require 'json'

class MyApp < Sinatra::Base
  @users = YAML.load(File.read('./data/users.yml'))
  @groups = YAML.load(File.read('./data/groups.yml'))

  ERROR_MSG = '{"Error": "Resource not found"}'

  def users
    self.class.instance_variable_get(:@users)
  end

  def groups
    self.class.instance_variable_get(:@groups)
  end

  get('/') { "Hello STNS." }

  get('/user/name/:username') do
    if users.has_key?(params['username'])
      res = {}
      res[params['username']] = users[params['username']]
    JSON.pretty_generate(res)
    else
      ERROR_MSG
    end
  end

  get('/group/name/:groupname') do
    if groups.has_key?(params['groupname'])
      res = {}
      res[params['groupname']] = users[params['groupname']]
    JSON.pretty_generate(res)
    else
      ERROR_MSG
    end
  end

  get('/user/list') do
    JSON.pretty_generate(users)
  end

  get('/group/list') do
    JSON.pretty_generate(groups)
  end

  get('/user/id/:uid') do
    user = users.find {|k,v| v['id'] == params['uid'].to_i }
    if user
      JSON.pretty_generate({user[0] => user[1]})
    else
      ERROR_MSG
    end
  end

  get('/group/id/:gid') do
    group = groups.find {|k,v| v['id'] == params['gid'].to_i }
    if group
      JSON.pretty_generate({group[0] => group[1]})
    else
      ERROR_MSG
    end
  end
end
```

## Dockerで試す

ここにあるコードで、[sawanoboly/stns_backend-sinatra-example](https://github.com/sawanoboly/stns_backend-sinatra-example)。

```
$ docker build -t stns_backend-sinatra-example . && docker run -it --rm stns_backend-sinatra-example 

...

Successfully built 83efde9edf72

root@3f809aa5376e:/code# 
```

### sinatraを起動して

あえて`&`で起動しておくと処理の様子がみれて面白い。

```
# bundle exec shotgun &
[1] 7
== Shotgun/WEBrick on http://127.0.0.1:9393/
[2016-08-04 10:28:52] INFO  WEBrick 1.3.1
[2016-08-04 10:28:52] INFO  ruby 2.3.1 (2016-04-26) [x86_64-linux]
[2016-08-04 10:28:52] INFO  WEBrick::HTTPServer#start: pid=7 port=9393
```

### idコマンド

idコマンドは4つリクエストがきてるねー。

```
# id example
127.0.0.1 - - [04/Aug/2016:10:30:12 +0000] "GET /user/name/example HTTP/1.1" 200 602 0.0133
127.0.0.1 - - [04/Aug/2016:10:30:12 +0000] "GET /user/id/1001 HTTP/1.1" 200 602 0.0106
127.0.0.1 - - [04/Aug/2016:10:30:12 +0000] "GET /group/id/1001 HTTP/1.1" 200 100 0.0107
127.0.0.1 - - [04/Aug/2016:10:30:12 +0000] "GET /group/list HTTP/1.1" 200 200 0.0107

uid=1001(example) gid=1001(example) groups=1001(example)
```

no such user も一応。

```
# id hogehoge
127.0.0.1 - - [04/Aug/2016:10:30:31 +0000] "GET /user/name/hogehoge HTTP/1.1" 200 31 0.0103
2016/08/04 10:30:31 command error:json: cannot unmarshal string into Go value of type stns.Attribute
id: hogehoge: no such user
```

エラー出てる気がする。。がまあいいや。


### sshログイン

じゃあ次はSSHによるログインだ。これもわりと色々聞かれるんだな。

```
# service ssh start
[ ok ] Starting OpenBSD Secure Shell server: sshd.

 'localhost (::1)' can't be established.
ECDSA key fingerprint is fb:35:be:88:20:ba:b8:49:67:99:37:15:cd:71:53:53.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
127.0.0.1 - - [04/Aug/2016:10:37:49 +0000] "GET /user/name/example HTTP/1.1" 200 602 0.0107
127.0.0.1 - - [04/Aug/2016:10:37:49 +0000] "GET /group/list HTTP/1.1" 200 200 0.0097
127.0.0.1 - - [04/Aug/2016:10:37:49 +0000] "GET /user/name/example HTTP/1.1" 200 602 0.0125
127.0.0.1 - - [04/Aug/2016:10:37:49 +0000] "GET /user/name/example HTTP/1.1" 200 602 0.0123
127.0.0.1 - - [04/Aug/2016:10:37:49 +0000] "GET /user/id/1001 HTTP/1.1" 200 602 0.0099

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
127.0.0.1 - - [04/Aug/2016:10:37:49 +0000] "GET /user/id/1001 HTTP/1.1" 200 602 0.0096


## ユーザ`example`でログインできた！
example@4b11cf1b387e:~$ 
$ pwd
/tmp
```

できました。これでSTNSのバックエンド実装はOKだと思います。

## おわりに

とても単純に好きなバックエンドに変更できることがわかりました。
大体の仕様が判ったので、次の目標はAWSのAPI Gateway+Lambdaから、DynamoDBをデータ置き場にすることかなー。

STNSのスライドから次の言葉を引用します。

![時代が求めたSTNSと僕____Speaker_Deck.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/4b9b3ad2-8238-20d0-dc3e-99324dd626d9.jpeg "時代が求めたSTNSと僕____Speaker_Deck.jpg")

ほんまやねー。
