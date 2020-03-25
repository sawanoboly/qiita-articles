<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[ProFTPd](http://www.proftpd.org/)のユーザ管理をChefのCookbookでしたいという話があって、リストのファイルを見てみると1行1レコード、コロン区切りのファイルでした。各フィールドは`/etc/passwd`と同じです。

通常ユーザ管理はCLI(ftpasswd)でやるとのことでしたが、どうせ中身が正しければちゃんと動くのでRubyで取り扱ってみます。

## サンプルのftpd.passwdファイル

さて、サンプルの`ftpd.passwd`を用意しました。
ユーザは`hoge1-3`の3名分、パスワードはそれぞれ`password1-3`を`salted hash(SHA512)`に加工しています。

```ftpd.passwd
hoge1:$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1:99:99:hoge1-san:/home/hoge1:/sbin/nologin
hoge2:$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/.:99:99:hoge2-san:/home/hoge2:/sbin/nologin
hoge3:$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0:99:99:hoge3-san:/home/hoge3:/sbin/nologin
```


### `unix_crypt`で認証文字列を生成するメソッド

`RubyGems:unix_crypt`をつかって、プレーンパスワードから`/etc/shadow`で見かける形式に変換するサンプルです。
記事中のサンプルで`#build_encrypt_by_plane`が出てきますがこのような処理をしています。

```ruby:unix_crypt_example.rb
require 'openssl'
require 'unix_crypt'

def _make_salt_by_plane(password)
  OpenSSL::Digest::MD5.hexdigest(password + 'salt_me').slice(0,15)
end

def build_encrypt_by_plane(password, type = :SHA512) # [:MD5, :SHA256, :SHA512]
  UnixCrypt.const_get(type).build(password, _make_salt_by_plane(password))
end
```

saltはランダムで生成するべきなのですが、Chefなどで管理する際に毎回更新されて多少うっとおしくなる事を考慮して今回は固定文字列にしています。
ランダムかつ毎回更新されないようにするには、`UnixCrypt.valid?($plainpass, $encrypted_strings)`でパスワードが更新されているかのチェックを行うなど工夫の余地がありますね。

`#build_encrypt_by_plane`実行すると`salted hash`を出力します。


```shell:Pry_build_encrypt_by_plane
pry(main)> build_encrypt_by_plane('password3')
=> "$6$f22eb55ffd0731f$FhoAmeAQmcqsGbEnRJWLuqv.AwlpYHiRz8xecGE.0teBnIY3pzko2y7lRl.rXcUraZVLJ4Kc.vF5EUU5HjTir0"
```

プレーンパスワード自体をどこに保管するかはそれなりに課題ですが、とりあえず棚上げしておきます。


## CSV.tableとして読み込む

では各フィールドに名前をつけつつ、カラムの区切りにコロンを指定して`ftpd.passwd`を読み込んでみます。

```ruby:Pry_read_from_ftp_passwd
pry(main)> require 'csv'
=> true

pry(main)> table = CSV.table('ftpd.passwd', {:headers => ['name', 'encrypt', 'uid', 'gid' ,'info' ,'home' ,'shell'], :col_sep => ':'})
=> #<CSV::Table mode:col_or_row row_count:4>

pry(main)> table[0]
=> #<CSV::Row name:"hoge1" encrypt:"$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1" uid:99 gid:99 info:"hoge1-san" home:"/home/hoge1" shell:"/sbin/nologin">

pry(main)> table[1]
=> #<CSV::Row name:"hoge2" encrypt:"$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/." uid:99 gid:99 info:"hoge2-san" home:"/home/hoge2" shell:"/sbin/nologin">

pry(main)> table[2][:info]
=> "hoge3-san"
```

ファイルの各行が`CSV::ROW`のオブジェクトになって色々と取り回しやすくなりましたね。


## 表示する(Stringに変換する) [to_csv]

無事にRubyのオブジェクトとして取り込めたので、`:write_headers`オプションの有り無しで表示してみます。

```ruby:Pry_show
pry(main)> puts table.to_csv(:write_headers => true, :col_sep => ':')
name:encrypt:uid:gid:info:home:shell
hoge1:$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1:99:99:hoge1-san:/home/hoge1:/sbin/nologin
hoge2:$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/.:99:99:hoge2-san:/home/hoge2:/sbin/nologin
hoge3:$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0:99:99:hoge3-san:/home/hoge3:/sbin/nologin
=> nil

pry(main)> puts table.to_csv(:write_headers => false, :col_sep => ':')
hoge1:$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1:99:99:hoge1-san:/home/hoge1:/sbin/nologin
hoge2:$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/.:99:99:hoge2-san:/home/hoge2:/sbin/nologin
hoge3:$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0:99:99:hoge3-san:/home/hoge3:/sbin/nologin
=> nil
```

加工後に保存する際の中身としても問題なさそうです。


## 探す [find, find_by_index]

では`CSV.table`から任意のレコードを探してみます。

### ユーザ名から行Get

行を取得するには`#find`を使います。

```ruby:Pry_find_by_name
pry(main)> user = table.find { |row| row[:name] == 'hoge2' }
=> #<CSV::Row name:"hoge2" encrypt:"$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/." uid:99 gid:99 info:"hoge2-san" home:"/home/hoge2" shell:"/sbin/nologin">

pry(main)> user[:encrypt]
=> "$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/."```
```

最初にヒットしたものを`CSV::ROW`として利用できます。

### 目的ユーザの行インデックスを取得する

`#find_index`ではヒットしたレコードのインデックスを得られます。

```ruby:Pry_find_by_index
pry(main)> table.find_index {|x| x[:name] == 'hoge3'}
=> 2

pry(main)> table[2]
=> #<CSV::Row name:"hoge3" encrypt:"$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0" uid:99 gid:99 info:"hoge3-san" home:"/home/hoge3" shell:"/sbin/nologin">
```

インデックスを使っても目的の行を取得することができます。


## 検索して更新する

先程の`#find_index`を利用して、新しいパスワードを設定してみます。ユーザ`hoge3`に対しパスワードを`update_passwd3`とします。


```ruby:Pry_update
pry(main)> table[2]
=> #<CSV::Row name:"hoge3" encrypt:"$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0" uid:99 gid:99 info:"hoge3-san" home:"/home/hoge3" shell:"/sbin/nologin">

pry(main)> (table[table.find_index {|x| x[:name] == 'hoge3'}])[:encrypt] = build_encrypt_by_plane('update_passwd3')
=> "$6$fd75fc6a14847b3$CHMTRMkQg3WHqTh.6OrTiahpuPvnoO4IFWFRHWjepXujMdb8NcJFwqeuSH4OCcgi8jaZu.q1Ht7e13RK9YKwW."

pry(main)> table[2]
=> #<CSV::Row name:"hoge3" encrypt:"$6$fd75fc6a14847b3$CHMTRMkQg3WHqTh.6OrTiahpuPvnoO4IFWFRHWjepXujMdb8NcJFwqeuSH4OCcgi8jaZu.q1Ht7e13RK9YKwW." uid:99 gid:99 info:"hoge3-san" home:"/home/hoge3" shell:"/sbin/nologin">
```

`encrypt`フィールドの中身が更新されました、行単位の更新も同じように可能です。


## 加える [<<]

新しいユーザを作成する要領です。

`CSV::Row`のインスタンスを任意の内容で作成し、`#<<`でテーブルに行を追加します。
ユーザ`hoge4`をパスワード`password4`で作成します。


```ruby:Pry_find
pry(main)> new_row = CSV::Row.new(
  [:name,:encrypt,:uid,:gid,:info,:home,:shell],  
  ['hoge4', build_encrypt_by_plane('password4'), 99, 99, 'hoge4-san', '/home/hoge4', '/sbin/nologin']  
)  
=> #<CSV::Row name:"hoge4" encrypt:"$6$b0782018ad3bf4a$hGkXfKKV0wY0OM2KcvvhLs86izBDd46HEwf1ZgD0NIHdmj7n.J54S/p0IHQ43AVUlgyb/nLDexXIvYE1qjkRg0" uid:99 gid:99 info:"hoge4-san" home:"/home/hoge4" shell:"/sbin/nologin">

pry(main)> table << new_row
=> #<CSV::Table mode:col_or_row row_count:5>

pry(main> table[3]
=> #<CSV::Row name:"hoge4" encrypt:"$6$b0782018ad3bf4a$hGkXfKKV0wY0OM2KcvvhLs86izBDd46HEwf1ZgD0NIHdmj7n.J54S/p0IHQ43AVUlgyb/nLDexXIvYE1qjkRg0" uid:99 gid:99 info:"hoge4-san" home:"/home/hoge4" shell:"/sbin/nologin">

## 既に居るユーザなら追加しない
pry(main)> table << new_row unless table.find {|row| row[:name] == 'hoge4' }
=> nil

```

`#push`で複数のレコードを同時に追加もできるようです。

## 削除する [delete, delete_if]

任意の行を削除するには`#delete` または `#delete_if`です。
ユーザ`hoge2`の行を削除します。

```ruby:Pry_delete
pry(main)> table.delete table.find_index {|x| x[:name] == 'hoge2'}
=> #<CSV::Row name:"hoge2" encrypt:"$6$1a4a5969f81da2d$kT.LO1omkKavQs/kkh24ZnZO5o/FxlFfjVMverUr9iA4pL/iiEBCNwla.pLi3zlHAhCA4jr056msXD5CwAGk/." uid:99 gid:99 info:"hoge2-san" home:"/home/hoge2" shell:"/sbin/nologin">

pry(main)> table.delete table.find_index {|x| x[:name] == 'hoge2'}
=> [nil, nil]

pry(main)> puts table.to_csv(:write_headers => false, :col_sep => ':')
hoge1:$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1:99:99:hoge1-san:/home/hoge1:/sbin/nologin
hoge3:$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0:99:99:hoge3-san:/home/hoge3:/sbin/nologin
=> nil

# -------Reload table

pry(main)> table.delete_if {|row| row[:name] == 'hoge2'}
=> #<CSV::Table mode:col_or_row row_count:3>

pry(main)> puts table.to_csv(:write_headers => false, :col_sep => ':')
hoge1:$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1:99:99:hoge1-san:/home/hoge1:/sbin/nologin
hoge3:$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0:99:99:hoge3-san:/home/hoge3:/sbin/nologin
=> nil
```

`#delete` は indexを渡すと消してくれるんですが、消すとindexが詰められて普通に`Fixnum`を渡すと冪等にならないので`find`を使っています。
`#delete_if`は戻りのオブジェクトが`CSV::Table`なので、対象のレコードがあってもなくても結果に違いがありません。

この点はちょっと注意が必要かもしれませんね。

## 書き出す

加工が終わったらファイルに書き出します。Chefなら`Resource::File`または`LWRP`でやるところですがとりあえず`File.open`で。
先程の`user2`を消す処理を施した内容で出力してみます。

```ruby:Pry_output_to_file
pry(main)> File.open('ftpd.passwd_new','w') {|f| f.write(table.to_csv(:write_headers => false, :col_sep => ':'))}
=> 308

pry(main)> .cat ftpd.passwd_new
hoge1:$6$17ed5828ed832c5$3KWXpvu9tyOF5UEGswbahSQVAYRMt7ZEoxiMOT./8or8ipRrpL090sD.AowBeFyDAEIxBhxzG4DA/nqnzhHWR1:99:99:hoge1-san:/home/hoge1:/sbin/nologin
hoge3:$6$031669a10a1ff3b$BNDb4nGS1cxGfJzKJntaegQZ8NkOKFKr2S6zJsyFO5Ph6TbD0nCMUtbIVN1KqPhUIItcLU35I2gHjs.16UGNn0:99:99:hoge3-san:/home/hoge3:/sbin/nologin
```

構成管理フレームワークを使わない場合は、元のファイルを事前にコピーなどしておくと良いかもしれませんな。

## おわりに

使いやすいCRUDのインターフェイスがなかったり、ぱっと見よく分からんフォーマットの生ファイルも、そもそもそれを使うプログラムがパースしやすいようになっています。合わせてあげるのはそれほど難しいことではないのかもしれません。
なお、`httpd`の設定とかはおとなしくテンプレート使います。

