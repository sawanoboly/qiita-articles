<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

単純なxmlのパースをshellからもやりたい場合、gawk(awk)でやってみる。

タグの中だけ取りたい例。

```bash::ShellOut
$ echo "<url>http://www.example.com</url>" | gawk 'match($0, /<url>(.*)<\/url>/, a) {print a[1]}'
http://www.example.com
```


gawkのマッチ部分について

1. $0は貰った文字列
2. // で正規表現
3. 元の文字列とマッチグループを 配列`a`に格納

で、`a[0]`が元の文字、`a[1]`にマッチグループの一番目。


solaris系ならawkでも可。
スクリプト内なら`BASH_REMACH`を使っても出来るけど、ちょっと長くなる。
