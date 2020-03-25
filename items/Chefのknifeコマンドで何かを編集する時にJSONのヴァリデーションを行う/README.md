<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


knifeでChefServer上の情報を編集したら、しょっちゅうJSONのパースエラーで全部パアになる。
`from_file`でローカル編集してからというのも手だが、アップロード前に手軽に確認する方法がある。

この方法は`EDITOR`がvimのとき限定ですが、他のエディタでも似たようなことはできるでしょう。

## コケるとき

エディタの終了=ChefServerへのアップロードなので、編集したJsonがおかしいとコケる上に編集していた内容がなくなる。

```bash:Shell-Out
$ knife role edit monit_smartos

-- 編集後 --

ERROR: Yajl::ParseError: parse error: unallowed token at this point in JSON text
                                 "name": "monit_smartos",   "descripti
                     (right here) ------^
```

これが意外と面倒くさい。

## Vimからjson_verifyをコール

`!` でシステムコマンド実行、と`%`がカレントのファイルパスに置き換えられるので、`json_verify`コマンドに渡してみよう。

```vim:Vimのコマンドモードで
:!cat % | json_verify

-- コンソール側にvarifyの結果 --

JSON is valid

Press ENTER or type command to continue
```

またVimに帰ってくるので、続けて編集するか終了してChefServerにアップロードするか選ぼう。

json_verifyはMacOSXならbrewで手に入る、linux系でも`yajl`等のパッケージに付いてくるはずなのでそちらを使うとよい。
