<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Chefなどを使ってインフラ構築がメタデータ編集で済ませるようになると、jsonフォーマットとの付き合いが多くなる。




## CLIでjsonを扱うケース

例えばバックアップからChefのRoleを流用しようとすると、くたびれた状態のjsonテキストになっており面倒。

```shell::cli_stdout
$ cat server_backup/roles/base.json 
{"name":"base","description":"","json_class":"Chef::Role","default_attributes":{},"override_attributes":{},"chef_type":"role","run_list":["recipe[ntp]"],"env_run_lists":{}}
```

※ 補足：knifeの出力はppなjsonにも出来る。

同様にchef-soloでもattributeのためにjsonを使うため、書式等チェックが出来たほうがよい。

## ツールを導入する

インストールしやすいものから、下記を入れておく。

- json_reformat
- json_verify
- json_pp(jsonpp)

ppはいつもの通りpretty_print、STDINなどを整形してくれる。
他も大体名前の通り。


### ubuntuでインストール

```
apt-get install libjson-pp-perl
apt-get install yajl-tools
```

libjson-pp-perlはいつの間にか入ってることも多い。

### macOSのHomebrewでインストール

```
brew install yaji
brew install jsonpp
```

こちらのjsonppはubuntuのperl版とは違うが役割は一緒。



## 使用例

### jsonpp(json_pp)

先ほどの平べったいJsonファイルをppする。

```shell::cli_stdout
$ cat server_backup/roles/base.json | jsonpp 
{
  "name": "base",
  "description": "",
  "json_class": "Chef::Role",
  "default_attributes": {},
  "override_attributes": {},
  "chef_type": "role",
  "run_list": [
    "recipe[ntp]"
  ],
  "env_run_lists": {}
}
```

見やすい。

### json_verify

閉じ忘れたJsonを渡してみる。

```shell::cli_stdout
$ cat roles/base.json | json_verify 
lexical error: invalid character inside string.
                                        {   "name": "base,   "descript

                     (right here) ------^
JSON is invalid
```

invalidはexit_codeも1になる。

該当箇所を直してみる。

```shell::cli_stdout
$ cat roles/base.json | json_verify 
JSON is valid
```

validになった。
