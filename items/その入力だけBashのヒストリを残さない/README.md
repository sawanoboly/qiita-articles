<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

入力時に1つ以上の半角スペースを入れるとBashヒストリに残らない。

```shell
# lsの前に半角スペース
$  ls -a
. ..
$ history
1 history
```


下記のようなケースで役に立つ。

* VMのひな形作成などでヒストリを残したくない
* ヒストリにあったshutdownコマンドを勢いで叩いてしまいたくない


## 追記：依存する設定
ちゃんと設定があるという突っ込みを頂いたので追記、環境変数`HISTCONTROL`に ignorespace または ignoreboth が設定されていたらこの記事の挙動になる。
という事がmanにちゃんと書いてあった。

```man-bashより
HISTCONTROL
       A colon-separated list of values controlling how commands are saved on the history  list.   If
       the  list  of  values  includes  ignorespace, lines which begin with a space character are not
       saved in the history list.  A value of ignoredups causes lines matching the  previous  history
       entry  to not be saved.  A value of ignoreboth is shorthand for ignorespace and ignoredups.  A
       value of erasedups causes all previous lines matching the current line to be removed from  the
       history list before that line is saved.  Any value not in the above list is ignored.  If HIST-
       CONTROL is unset, or does not include a valid value, all lines read by the  shell  parser  are
       saved  on  the  history  list,  subject to the value of HISTIGNORE.  The second and subsequent
       lines of a multi-line compound command are not tested, and are added to the history regardless
       of the value of HISTCONTROL.

```
