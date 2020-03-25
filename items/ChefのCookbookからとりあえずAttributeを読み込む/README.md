<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[Chefのソース](https://github.com/opscode/chef)に結合テスト用のサンプルCookbookがあります。


```
$ tree kitchen-tests/cookbooks/webapp/
kitchen-tests/cookbooks/webapp/
├── Berksfile
├── README.md
├── attributes
│   └── default.rb
├── metadata.rb
├── recipes
│   └── default.rb
└── templates
    └── default
        ├── index.html.erb
        └── index.php.erb

4 directories, 7 files
```

`attributes/default.rb`ってのがありますね。

```kitchen-tests/cookbooks/webapp/attributes/default.rb 
default['apache']['remote_host_ip'] = '127.0.0.1'

default['mysql']['version'] = "5.5"

default['webapp']['database'] = 'webapp'
default['webapp']['db_username'] = 'webapp'
default['webapp']['path'] = '/var/www/'
```

外部プログラムから使うこともあるかもしれないので、`attributes/default.rb`を読んでみます。


## Pryで

```
## とりあえずChefを読みます
[1] pry(main)> require 'chef'
=> true

## Nodeオブジェクトをつくります
[2] pry(main)> node = Chef::Node.new
=> #<Chef::Node:0x007f88e30ab7c0
 @attributes={},
 @chef_environment="_default",
 @name=nil,
 @override_runlist=#<Chef::RunList:0x007f88e30ab6f8 @run_list_items=[]>,
 @primary_runlist=#<Chef::RunList:0x007f88e30ab770 @run_list_items=[]>,
 @run_state={}>


## #from_fileでattributeファイルを読みます
[3] pry(main)> node.from_file('kitchen-tests/cookbooks/webapp/attributes/default.rb')
=> "/var/www/"


## default.rbの分がnodeに格納されました。
[4] pry(main)> node[:mysql]
=> {"version"=>"5.5"}

[5] pry(main)> node[:webapp][:db_username]
=> "webapp"
```



ファイルだけ読んでも仕方ないという場合は、ChefClient終了時の状態を丸ごとレポートハンドラでjsonダンプしとけば使いまわせるんじゃないかな。
