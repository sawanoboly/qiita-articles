<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


まず`mysql_secure_installation`するじゃないですか。もののついでにChefでやっておこうと思いました。

`mysql_secure_installation`実行時の質問を抜粋。

1. Change the root password [Y/n]
2. Remove anonymous users? [Y/n]
3. Disallow root login remotely? [Y/n] 
4. Remove test database and access to it? [Y/n] 
5. Reload privilege tables now? [Y/n]

で、同様のことを実施。

```ruby:mysql_secure_install.rb
bash 'mysql_secure_install emulate' do
  code <<-"EOH"
    /usr/bin/mysqladmin drop test -f
    /usr/bin/mysql -e "delete from user where user = '';" -D mysql
    /usr/bin/mysql -e "delete from user where user = 'root' and host = \'#{node[:hostname]}\';" -D mysql
    /usr/bin/mysql -e "SET PASSWORD FOR 'root'@'::1' = PASSWORD('newpassword');" -D mysql
    /usr/bin/mysql -e "SET PASSWORD FOR 'root'@'127.0.0.1' = PASSWORD('newpassword');" -D mysql
    /usr/bin/mysql -e "SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpassword');" -D mysql
    /usr/bin/mysqladmin flush-privileges -pnewpassword
  EOH
  action :run
  only_if "/usr/bin/mysql -u root -e 'show databases;'"
end
```

パスワードは`encrypted_data_bag`に置くとして、フルbashじゃないですか。

> 追記： `3. Disallow root login remotely? [Y/n] `に当たる処理はコメント欄にある方が適切です。

## result

試してみます、パスワード無しでmysqlに接続できるようなら`mysql_secure_installation`相当の処理をさせます。

```shell:chef-apply
# cat <<'EOL' | chef-apply -s
> bash 'mysql_secure_install emulate' do
>   code <<-"EOH"
>     /usr/bin/mysqladmin drop test -f
>     /usr/bin/mysql -e "delete from user where user = '';" -D mysql
>     /usr/bin/mysql -e "delete from user where user = 'root' and host = \'#{node[:hostname]}\';" -D mysql
>     /usr/bin/mysql -e "SET PASSWORD FOR 'root'@'::1' = PASSWORD('newpassword');" -D mysql
>     /usr/bin/mysql -e "SET PASSWORD FOR 'root'@'127.0.0.1' = PASSWORD('newpassword');" -D mysql
>     /usr/bin/mysql -e "SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpassword');" -D mysql
>     /usr/bin/mysqladmin flush-privileges -pnewpassword
>   EOH
>   action :run
>   only_if "/usr/bin/mysql -u root -e 'show databases;'"
> end
> EOL
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * bash[mysql_secure_install emulate] action run
    - execute "bash"  "/tmp/chef-script20130822-12249-10qdpq7-0"
```

```shell:mysql_user_table
# mysql -e 'select user,host,password from user;' -D mysql -pnewpassword
+------+-----------+-------------------------------------------+
| user | host      | password                                  |
+------+-----------+-------------------------------------------+
| root | localhost | *FE4F2D624C07AAEBB979DA5C980D0250C37D8F63 |
| root | 127.0.0.1 | *FE4F2D624C07AAEBB979DA5C980D0250C37D8F63 |
| root | ::1       | *FE4F2D624C07AAEBB979DA5C980D0250C37D8F63 |
+------+-----------+-------------------------------------------+
```

まあできてるか。

２度目以降は`only_if`が効いてお断りしてくれます。

```
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * bash[mysql_secure_install emulate] action run (skipped due to only_if)
```
