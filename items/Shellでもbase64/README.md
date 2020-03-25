<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


`GNU coreutils`パッケージに含まれる**base64コマンド**はbase64のエンコード/デコードが出来る。

pemファイルを動的なコンフィグに格納する際などに使える。

## エンコード

```
$ echo -n "giraffi" | base64            
Z2lyYWZmaQ==
```

## デコード

```
$ echo -n "Z2lyYWZmaQo=" | base64 -d
giraffi
```

大概のディストリビューションに入ってるかと思う。

