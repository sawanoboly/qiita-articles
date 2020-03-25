[Swagger](http://swagger.io)ってのがあってね。概要は次のスライドで。

- スライド: [Swaggerで始めるモデルファーストなAPI開発 (Takuro Sasaki, Software Developer)](http://www.slideshare.net/takurosasaki/swaggerapi)

で、以前にSTNSのバックエンド仕様を調べたことがあります。このときのSTNS APIはバージョン1(以下、`V1`とか)です。

[STNSのバックエンドをつくろう - Qiita](http://qiita.com/sawanoboly/items/d8b017760d8b51d7ded4)

さて、Swaggerの手習いで作ってみるAPIとして、STNSのAPI バージョン2.1を模すれば二兎かもしれないと考えたところ、これミスマッチだったというお話。
何かの役に立つわけでもないが、手記としてしたためておく。

## STNSのAPI Version 2

[STNS Api Version 2](http://stns.jp/en/interface) について。

- プレフィクス`/v2`
- レスポンスは`metadata`と`items`という2要素で構成
    - itemsの中身はV1のレスポンスと同じ
    - metadataはなんだかグローバルな情報
- sudoリソース
- ヘルスチェックはプレフィクスなしのv1と共通

うむ、ここまでよし。

### レスポンスv2

リスト`/v2/user/list`のレスポンスはこんな感じ。

```json:
{
  "metadata": {
    "api_version": 2.1,
    "result": "success",
    "min_id": 1001
  },
  "items": {
    "example1": {
      "id": 1001,
      "password": "",
      "group_id": 0,
      "directory": "",
      "shell": "",
      "gecos": "",
      "keys": [
        "ssh-rsa xxxx"
      ],
      "link_users": null
    },
    "example2": {
      "id": 1002,
      "password": "",
      "group_id": 0,
      "directory": "",
      "shell": "",
      "gecos": "",
      "keys": [
        "ssh-rsa xxxx"
      ],
      "link_users": null
    }
  }
}
```

単品`/v2/user/name/example1`のレスポンスはこうなった。

```json:
{
  "metadata": {
    "api_version": 2.1,
    "result": "success",
    "min_id": 1001
  },
  "items": {
    "example1": {
      "id": 1001,
      "password": "",
      "group_id": 1001,
      "directory": "",
      "shell": "",
      "gecos": "",
      "keys": [
        "ssh-rsa xxxx"
      ],
      "link_users": null
    }
  }
}
```

一見Swaggerでも定義は作りやすそうに見えたんだけど。。

### なにが困ったのか1: データモデル

まず、データのキー名にユニークなIDをつかうとSwaggerで定義するモデル作成に困るのです。
例えば次の形でサンプルレスポンスを作るにはベタ打ちしか方法がないっぽい。

```
"example1": {
  "id": 1001,
...
```

構造的にこうなってると。Google Bigtableとかのような持ち方。

```
|          example1          |
| id   | password | group_id |
|------|----------|----------|
| 1001 | $6....   | 1001     |
```

こういう状態だと、プロパティ側のモデルは作れるんだけども、それを使いまわす形でのレスポンスが作成できなかった。ドキュメントとしてなら全部サンプルレスポンス手打ちでよいのだけども、OpenAPI Specとしてはそうもいかない。

あくまでSwaggerのdefinitionsで定義するとなると、次のような構造が必要だった。

```
{
  "name": "example1",
  "id": 1001,
...

```

つまりこんな状態。RDB的なレコード。

```
| name     | id   | password | group_id |
|----------|------|----------|----------|
| example1 | 1001 | $6....   | 1001     |

```

比べると、ああそうだよねえ。。としか。

### なにが困ったのか2: metadataのmin_id

もう一つ、metadataのmin_idがレスポンスに含む含まない関係なしで、常にレコード全体から算出した値であることがネックになりそうだった。
こちらはSwaggerからすこし外れる話でただの感想なんだけども、バックエンドのデータストア次第でちょっと困るなと。

```json:
"metadata": {
    "api_version": 2.1,
    "result": "success",
    "min_id": 1001
  }
```

min_idに入れる値についてはRDBならMINな関数でいいけど、例えばDynamoDBなんかだと横断してMAXとかMINとかとれない。

[java - MAX operations in Amazon DynamoDB - Stack Overflow](http://stackoverflow.com/questions/18219562/max-operations-in-amazon-dynamodb)

この場合、オブジェクトの更新時にフックしてチェック＆更新をかける専用テーブル(アイテム)が必要なのよね。。プラットフォームの選定に影響するのだ。

## 完

まずここで完全移植用の定義をつくるのは諦めた。あとはおまけ。

## Swaggerぽいモデルにしてみる

> 元々がディレクトリ型のデータなので、キーとプロパティの組み合わせというレスポンスで当たり前であることはひとまず置いといてください。

当初の目的のうち、Swaggerの練習くらいはやっておきたい。折角なのでSwaggerで作れるようにAPI仕様を改変してみよう。

この辺を参考に。

- [PetStore | Swagger UI](http://petstore.swagger.io/)
- [swaggerの基礎。swaggerの設定ファイルの書き方とか - Qiita](http://qiita.com/magaya0403/items/0419d84d8df7784ac465)

### 変更点1: metadataをなくす

`metadata`自体はキー名が固定なので普通にモデル定義できる。実は残してもいい。

しかし`api_version`はそもそもリクエスト側が指定できればいい気がするぞ。。すでに`/v2`プレフィクスもあるし。
もちろんデフォルトがあるからレスポンスに入れるとしても`X-API-VERSION`というヘッダにでもいれちゃおう。

`result`はHTTPのステータスコードでいいんじゃないのかなあ。

`min_id`が厳しい。ちょい不自然だが...ヘッダだ。`X-MIN-ID`にいれちゃう。


### 変更点2: user_name, group_nameをモデルに含める

まあAPI v1に近くなちゃうね。キーをnameにしまっただけ。
ほか、nullもなんだかまずい気がするのでlink_usersなどは無しの場合に空配列としよう。

```
> GET /user/name/example1

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-API-VERSION: 2.1
X-MIN-ID: 1001
Date: xxxxxxxx
Content-Length: xxx

{
  "name": "example1",
  "id": 1001,
  "password": "",
  "group_id": 1001,
  "directory": "",
  "shell": "",
  "gecos": "",
  "keys": [
    "ssh-rsa xxxx"
  ],
  "link_users": []
}
```


### 変更点3: Get_By_IDは単品オブジェクトを、リストはアレイでレスポンス

v2の`items`にあたるところを変えてしまおう。

単品のGETについては変更点2のとおり、リストを取得する場合はアレイにする。これなら同じモデルのオブジェクトが並ぶだけだ。

```
> GET /user/list

HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-API-VERSION: 2.1
X-MIN-ID: 1001
Date: xxxxxxxx
Content-Length: xxx

[
  {
    "name": "example1",
    "id": 1001,
    "password": "",
    "group_id": 1001,
    "directory": "",
    "shell": "",
    "gecos": "",
    "keys": [
      "ssh-rsa xxxx"
    ],
    "link_users": []
  },
  {
    "name": "example2",
    "id": 1002,
    "password": "",
    "group_id": 1001,
    "directory": "",
    "shell": "",
    "gecos": "",
    "keys": [
      "ssh-rsa xxxx"
    ],
    "link_users": []
  }
]
```


### 変更点4: 404はmessageで

V2では`"items": null`で返ってくる。Swaggerサンプルを参考にするとこれもV1に寄るなあ。

```
> GET /user/name/example1

HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
X-API-VERSION: 2.1
X-MIN-ID: 1001
Date: xxxxxxxx
Content-Length: xxx

{
  "message": "Resource not found"
}
```

### できたSwaggerのSpec (パチもんAPI)

とりあえず`/v2/user/name/{id}`と`/v2/user/list`だけ実装。

一応こちらでソースとインタラクティブなドキュメントが見られるはずです。
=> https://app.swaggerhub.com/api/sawanoboly/sandbox1/2.1

`swaggerhub`だとヘッダがドキュメントに含まれなかったりと妙な挙動をしてますが、[editor.swagger.io](http://editor.swagger.io/)の方に貼り付ければレスポンスヘッダもドキュメントに含まれてました。


```yaml:
---
swagger: '2.0'
info:
  version: "2.1"
  title: mysandbox1
  description: example 1
paths:
  /healthcheck:
    get:
      description: health check url
      produces:
        - application/json
      responses:
        '200':
          description: 'returns success'
          schema:
            $ref: "#/definitions/Healty"
          headers:
            X-API-VERSION:
              description: api version
              type: number
              default: 2.1
            X-MIN-ID:
              description: please return the minimum Id in the All users, group
              type: number
  /v2/metadata:
    get:
      description: Returns metadata (demo)
      produces:
        - application/json
      responses:
        '200':
          description: 'Metadata'
          schema:
            $ref: "#/definitions/Metadata"
          headers:
            X-API-VERSION:
              description: api version
              type: number
              default: 2.1
            X-MIN-ID:
              description: please return the minimum Id in the All users, group
              type: number
  /v2/user/name/{id}:
    parameters:
      - name: id
        in: path
        required: true
        type: string
    get:
      description: find by user name
      produces:
        - application/json
      responses:
        '404':
          description: '404 error'
          schema:
            $ref: "#/definitions/NotFound"
        '200':
          description: '404 error'
          schema:
            $ref: "#/definitions/User"
          headers:
            X-API-VERSION:
              description: api version
              type: number
              default: 2.1
            X-MIN-ID:
              description: please return the minimum Id in the All users, group
              type: number
  /v2/user/list:
    get:
      description: list of all users
      produces:
        - application/json
      responses:
        '404':
          description: '404 error'
          schema:
            $ref: "#/definitions/NotFound"
        '200':
          description: '404 error'
          schema:
            type: array
            items:
              $ref: "#/definitions/User"
          headers:
            X-API-VERSION:
              description: api version
              type: number
              default: 2.1
            X-MIN-ID:
              description: please return the minimum Id in the All users, group
              type: number
definitions:
  Healty:
    description: Healty response
    type: string
    example: "success"
  NotFound:
    description: 404 response
    type: object
    properties:
      message:
        type: string
        example: "Resource not found"
      items:
        example: null
  Metadata:
    description: metadata object
    type: object
    required:
      - api_version
      - result
      - min_id
    properties:
      api_version:
        description: api version
        type: number
        example: 2.1
      result:
        description: success only
        type: string
        example: "success"
      min_id:
        description: please return the minimum Id in the All users, group
        type: integer
        example: 1001
  User:
    description: User object
    type: object
    required:
      - name
      - id
    properties:
      name:
        type: string
        example: example1
      id:
        description: uid
        type: integer
        example: 1001
      password:
        description: login password by /etc/shadow format
        type: string
        example: "$6$salt$hash"
      group_id:
        description: gid
        type: integer
        example: 1001
      directory:
        description: home directory
        type: string
        example: "/home/example1"
      shell:
        description: default shell
        type: string
        example: "/bin/bash"
      keys:
        description: public keys
        type: array
        items:
          type: string
          example: "ssh-rsa xxxxx"
      gecos:
        description: general information about the account
        type: string
        example: "example user"
      link_users:
        description: stns link users
        type: array
        items:
          type: string
          example: "example2"
# Added by API Auto Mocking Plugin
host: virtserver.swaggerhub.com
basePath: /sawanoboly/sandbox1/2.1
schemes:
 - https
```

ちなみにレスポンスは`metadata`+`items`という構成にもちゃんとできる。itemsはアレイになっちゃうけどもたしかこんな感じ。

```yaml:
schema:
  type: object
  properties:
    metadata:
      $ref: "#/definitions/Metadata"
    items:
      type: array
      items:
        $ref: "#/definitions/User"
```

(前回のだけど)普通に実装したSinatraのコードより長いぜ。

curlで取ってみる。(※モックのvirtserverはヘッダ定義が無効っぽい)

```
$ curl -i https://virtserver.swaggerhub.com/sawanoboly/sandbox1/2.1/v2/user/list
HTTP/1.1 200 OK
Access-Control-Allow-Headers: Content-Type
Access-Control-Allow-Methods: *
Access-Control-Allow-Origin: *
Content-Type: application/json
Server: Jetty(6.1.26)
Content-Length: 212
Connection: keep-alive

[ {
  "name" : "example1",
  "password" : "$6$salt$hash",
  "directory" : "/home/example1",
  "shell" : "/bin/bash",
  "keys" : [ "ssh-rsa xxxxx" ],
  "gecos" : "example user",
  "link_users" : [ "example2" ]
} ]
```

まあよし。


## で、どうなるの

STNSのバックエンドAPIをSwaggerに寄せる(要はRESTぽくする)とバックエンドの移植はしやすく、POST/PUTでのデータ管理が実装しやすくなるけども、肝心のLinuxで動くNSS/PAMクライアント側でのデータの取り回しが煩雑になりそうだなと思いました。

以前V1を自分がsinatraで作ってみたり、cloudpackさんがDynamoDB直読みで作った(下記リンク)ように、STNSバックエンドの実装自体は簡単で、それはV2でも同じです。
結局Swaggerを使ってみようとした時点で選択を誤っていたということですね。

[Linuxユーザ管理の決定版？ 〜STNSとサーバレスで夢が広がる〜【cloudpack大阪ブログ】 - Qiita](http://qiita.com/shogomuranushi/items/f09fcdeb146b45452403)

そもそも元がシンプルなキーバリューなので愚直に実装すればよいんだよな。

さて今回得られたものは。

- Swagger Specの書き方
- モデルファーストで作れるAPIの判定方法をひとつ
- STNSバックエンドのパチもん

の3点でした。
