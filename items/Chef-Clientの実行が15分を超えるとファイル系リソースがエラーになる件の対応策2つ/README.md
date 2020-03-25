<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

FAQなので書いておく。ファイル系ってのはこのへん。

- cookbook_file
- template


問題の詳細は[CHEF-3045](https://tickets.opscode.com/browse/CHEF-3045)のチケットで確認できます。

要約すると。

1. Chef-Clientが実行されたら、最初にChef-Server/Client間の認証が行われる
2. ↑の有効期限はデフォルトで15分
3. ファイル系のリソースはCookbookロード時にはChef-Client実行NodeのCacheには落ちてこない
4. Chef-Clientのリソース収束の段階で、必要なファイルがあったら1で作った認証情報を使ってChef-Server(Bookchelf API)に取りに行く
5. 15分超えてたら4の挙動が403でアウトー

キャッシュがある場合はそもそも取りに行かないが、NodeでのChef-Client初回実行時、いっぺんに色々やっていると良くある。



## 1. Chef-Serverの認証期限デフォルト15分を伸ばす

Chef-Serverの設定ファイル`/etc/chef-server/chef-server.rb`で、有効期限を伸ばす。

```ruby:/etc/chef-server/chef-server.rb

erchef['s3_url_ttl'] = 3600  # 例
```

で、設定の適用すればOK。

`chef-server-ctl reconfigure`


## 2. Chef-Clientで遅延ダウンロードを無効にする

Chef-Clientが`10.28.0`以降、または`11.6.0`以降なら、`no_lazy_load `オプションが使える。

```ruby:/etc/chef/client.rb
Chef::Config[:no_lazy_load] = true
```

こちらはChef-ClientがCookbookをダウンロードする時に`files/`、`templates/`を除外しないオプション。
関係ないファイルも全部Nodeに落ちてくるので注意。
