<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Opscode Chefではレシピをランリスト(run_list)としてコマンドラインクライアントに渡したり、Role等に記述します。

あれって結局どういう書式が有効なんでしょうかと思いまとめました。


## 有効な書式一覧

ｰ `recipe[recipe_name]`
- `recipe[recipe_name@1.0.0]`
- `role[role_name]`
- `recipe_name@1.0.0`
- `recipe_name`

引用元はは[こちらのソース](https://github.com/opscode/chef/blob/master/lib/chef/run_list/run_list_item.rb)、MasterなのでChef11ですね。


目を引くのは@ですね、実はランリストで直接バージョン指定ができるんですね。


## `recipe_name`について

さて、`recipe_name`を更に分解すると`cookbook_name::recipe_name`、または`recipe_name`という書式になります。
バージョン指定もつける時は、`mysql::default@1.2.1`のように@より前にレシピの名前をつければOKです。

## ちなみにsoloでは

@でのバージョンは無視されるようです？
ディレクトリを分けるという対応があります、soloならmetadata.rbの`name`はかぶってOK。
