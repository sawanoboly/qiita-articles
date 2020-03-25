<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

`Infrastructure as code`の実践では意外とJSONの敷居が高いが、インフラ屋がもっとJSONと仲良くなるためのツールがある。
Shellから使いやすければ、他のツールとも連携がしやすくなる…はず。

ちなみに[JoyentのSmartOS(SmartMachine)](http://joyent.com/technology/smartos)では仮想サーバ作成直後から`json`コマンドが使えてラク。


## jsontoolのインストール

`node.js`は好きなようにに導入して、npm。

```bash:ShellOut
$ npm install -g jsontool
$ json --version
json 5.1.1
```

ソースのリポジトリはこちら[https://github.com/trentm/json](https://github.com/trentm/json)。

## サンプルJsonファイルを作る



```json:metadata.json
{
    "metadata": {
        "user-script": "",
        "credentials": {
            "root": "root_password",
            "admin": "admin_password",
            "mysql": "mysql_password",
            "pgsql": "pgsql_password"
        }
    }
}
```

## パースして出力する

`json`コマンドのオプションにキーを指定するとValueが取得できる、階層をつけるなら.(ピリオド)で区切る。

```bash:ShellOut
$ cat metadata.json | json metadata.credentials      
{
  "root": "root_password",
  "admin": "admin_password",
  "mysql": "mysql_password",
  "pgsql": "pgsql_password"
}

$ cat metadata.json | json metadata.credentials.mysql
mysql_password
```

テキスト処理を使って自力でパースするのとは雲泥の差！

## スクリプトファイル内で扱う

```bash:sample.sh
MYSQL_PASS=`cat metadata.json | json metadata.credentials.mysql`  
echo ${MYSQL_PASS}
mysql_password
```
`cat`でやっているが、王道は`curl`で他所のAPIからの取得でしょう。
