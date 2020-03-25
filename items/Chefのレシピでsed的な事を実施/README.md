<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

`Infrastructure as Code`に感化されていざChefを使うとなった時、ちょっと既存のファイルを弄りたいだけなのに`cookbook`に`file`とか`template`とかメンドクセって思うじゃないですか。

『ほな`execute`リソースでちょっとsedコマンドしたら…』と思う前にちょっと`Chef::Util`を検討しませんか。


## 対象ファイルとレシピ

サンプルとしてこんな2行で出来たファイルを2つ用意しました。

```shell:tmp1,tmp2
hogehoge
mogemoge
```

では 'tmp1'に一致した行の置換、'tmp2'では文字列の置換を実施してみます。

```ruby:apply.rb
file './tmp1' do
  _file = Chef::Util::FileEdit.new(path)
  _file.search_file_replace_line('^hoge', "piyopiyo\n")
  content _file.send(:contents).join
end

file './tmp2' do
  _file = Chef::Util::FileEdit.new(path)
  _file.search_file_replace('ge', "ga")
  content _file.send(:contents).join
end
```

> 追記中：Chef-Clientが11.12以降では FileEditの@contentsがなくなり、@editor.linesに変更。

```ruby:apply2.rb
## 11.12.0以降のクライアント
file './tmp1' do
  _file = Chef::Util::FileEdit.new(path)
  _file.search_file_replace_line('^hoge', "piyopiyo\n")
  content _file.send(:editor).lines.join
end

file './tmp2' do
  _file = Chef::Util::FileEdit.new(path)
  _file.search_file_replace('ge', "ga")
  content _file.send(:editor).lines.join
end
```

`Chef::Util::FileEdit.new(path)`の`path`について。
レシピのリソース定義内では自分(self)の`attributes`を利用することができるので、path(※省略時はnameと同じ)を使いまわして`Chef::Util::FileEdit`読み込み対象にしてるんですね。

参考：[file &mdash; Chef Docs](http://docs.opscode.com/resource_file.html)

`#contents`がprivate_methodなのでちょっと`:send`しましたがまあいいっしょ。


## Result

`:dry_run`です、きっと目論見通りですね。

```shell:chef-apply(dry_run)
$ chef-apply appry.rb -w
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * file[./tmp1] action create
    - Would update content in file ./tmp1 from 639ed8 to 55d5ca
        --- ./tmp1	2013-08-20 00:45:11.000000000 +0900
        +++ /var/folders/s9/kwcs_8ln0d32v_n4hdfb7td00000gn/T/.tmp120130820-24879-gemfn5	2013-08-20 00:45:19.000000000 +0900
        @@ -1,2 +1,2 @@
        -hogehoge
        +piyopiyo
         mogemoge
  * file[./tmp2] action create
    - Would update content in file ./tmp2 from 639ed8 to 93260c
        --- ./tmp2	2013-08-20 00:45:03.000000000 +0900
        +++ /var/folders/s9/kwcs_8ln0d32v_n4hdfb7td00000gn/T/.tmp220130820-24879-ieq0kc	2013-08-20 00:45:19.000000000 +0900
        @@ -1,2 +1,2 @@
        -hogehoge
        -mogemoge
        +hogahoga
        +mogamoga
```

実施します、まあ`:dry_run`と変わらず。

```shell:chef-apply(conversion)
$ chef-apply appry.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * file[./tmp1] action create
    - update content in file ./tmp1 from 639ed8 to 55d5ca
        --- ./tmp1	2013-08-20 00:45:11.000000000 +0900
        +++ /var/folders/s9/kwcs_8ln0d32v_n4hdfb7td00000gn/T/.tmp120130820-24938-1bwvf6g	2013-08-20 00:45:41.000000000 +0900
        @@ -1,2 +1,2 @@
        -hogehoge
        +piyopiyo
         mogemoge
  * file[./tmp2] action create
    - update content in file ./tmp2 from 639ed8 to 93260c
        --- ./tmp2	2013-08-20 00:45:03.000000000 +0900
        +++ /var/folders/s9/kwcs_8ln0d32v_n4hdfb7td00000gn/T/.tmp220130820-24938-h5bq4k	2013-08-20 00:45:41.000000000 +0900
        @@ -1,2 +1,2 @@
        -hogehoge
        -mogemoge
        +hogahoga
        +mogamoga
```

続いて`chef-apply`実施、もうファイルに対して残す変更はありません。

```shell:chef-apply(up_to_date)
$ chef-apply appry.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * file[./tmp1] action create (up to date)
  * file[./tmp2] action create (up to date)
```

よし、本日も冪等である。

## その他FileEditのメソッド

その他sedでよく使うマッチ行の次に挿入や、sedでやるのは多分面倒くさい'無かったら追記'がメソッドとして用意されています。


```shell:Chef::Util::FileEdit.instance_methods(false)
pry(main)> Chef::Util::FileEdit.instance_methods(false)
=> [:search_file_replace_line,
 :search_file_replace,
 :search_file_delete_line,
 :search_file_delete,
 :insert_line_after_match,
 :insert_line_if_no_match,
 :write_file]
```

`Chef::Util`のあたりChefの中でもかなり古いコードです、初期はファイルをちょろっといじるようなツールだったのでしょうかね。

----

追記：
念のため書いておくと`Chef::Util::FileEdit`本来の使い方は`#write_file`で、`Chef::Util::Backup`などと組み合わせて主にLibraryやLWRP方面で利用します。
ただレシピ内でやるのは芸が無いと思うのです。

追記2：

普通の使い方のサンプルも引用しておきます。

timezone cookbook(fork版)より。

https://github.com/pearj/cookbooks/blob/master/timezone/recipes/default.rb

```
  clock = Chef::Util::FileEdit.new("/etc/sysconfig/clock")
  clock.search_file_replace_line(/^ZONE=.*$/, "ZONE=\"#{node[:tz]}\"")
  clock.write_file
```

