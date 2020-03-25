<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

bashスクリプト間で関数を共通化したい時、ファイルならsourceやドットコマンドでやるところをGist等からcurlで取ってきたい時。


## Gistにサンプルのファンクション
<script src="https://gist.github.com/3126623.js?file=sample.func"></script>


## curlでgistのrawフォーマットを取り込むコード
ダブルクォートでevalを実行。

```bash:sample.sh 
#!/bin/bash

EXT=`curl -f -k -s https://raw.github.com/gist/3126623/35571a9e7ef8a7782368d5c39c2b0f03edf7b393/sample.func`
eval "$EXT"

pp
```

## 実行してみる

    $ ./sample.sh 
    pretty


Curlが対応しているurlなら同様の手順でOKのはず、特定環境内のコードを適当なWEBサーバで共有するのもよいだろう。
