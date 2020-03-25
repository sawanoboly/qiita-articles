<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[実践Vim](http://ascii.asciimw.jp/books/books/detail/978-4-04-891659-2.shtml) を読んでいます、基本機能の習得・おさらいにはとても良い本だと思います。

さて、

> コラム：矢印キーに指を伸ばすクセをやめる

によると、 **矯正期間に限り** vimrcで矢印キーを無効にしてしまうという例があった。

まだまだhjklでの移動をしない軟弱者なので、切り替えできるようにしてみた。

```vim:.vimrc
function! HardMode ()
  noremap <Up> <Nop>
  noremap <Down> <Nop>
  noremap <Left> <Nop>
  noremap <Right> <Nop>
endfunction

function! EasyMode ()
  noremap <Up> <Up>
  noremap <Down> <Down>
  noremap <Left> <Left>
  noremap <Right> <Right>
endfunction

command! HardMode call HardMode()
command! EasyMode call EasyMode()
```

これで`:HardMode`と打てば矢印キーが無効になり、`:EasyMode`でまた有効にできた。集中して作業できる時にHardModeにしてみよう。


なお、`EasyMode`が`Explore`と出だしがかぶっているので、`E`をよく使う人は別名にしましょう。
