<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

`chef-server-webui`はrails3で、セッションの管理は`cookie_store`を使用しています。
このCookieを発行する際にDomain属性を付けるようになっていますが、デフォルトはRails用の`:all`がセットされます。

インストール用Cookbookのテンプレートはこんな感じ。

```ruby:cookbooks/chef-server/templates/default/session_store.erb 
# Be sure to restart your server when you modify this file.

ChefServerWebui::Application.config.session_store :cookie_store, :key => '<%= @session_key %>', :domain => <%= @cookie_domain == "all" ? ':all' : "'#{@cookie_domain}'" %>
```

## Domainの値で困る例

この`:all`のデフォルト挙動がちょっとアレで、例えばEC2に立てたインスタンス、`ec2-xx-xx-xx-xxx.ap-northeast-1.compute.amazonaws.com`のようなドメインで来ると、次のようなCookieを発行します。

```
_sandbox_session=SESSION_KEY; domain=.amazonaws.com; path=/; HttpOnly
```

`domain=.amazonaws.com`はちょっとレベル高すぎだろう。
最近の主要なブラウザでは、ホスト名を使ったアクセス(IPならOK)ではログインできない事態になります。

> ※: Rails側の`config.action_dispatch.tld_length`で深さの変更も一応可能。


Domainに何をセットしておけばいいのかはこの辺を参考に、[CookieのDomain属性は *指定しない* が一番安全 | 徳丸浩の日記](http://blog.tokumaru.org/2011/10/cookiedomain.html "CookieのDomain属性は *指定しない* が一番安全 | 徳丸浩の日記")

## Domainを送らない設定

じゃあ、指定しないようにしましょう。そのための設定はこちら。


```ruby:chef-server.rb
chef_server_webui['cookie_domain'] = ''
```

テンプレートの関係で、`:domain => hogehoge`が省略出来ないので空文字で。
これでreconfigureすれば、ホスト名でアクセスした際もDomain属性が発行されません。

```
_sandbox_session=SESSION_KEY; path=/; HttpOnly
```



> ### ログにはこんなの出ます
> ホスト名アクセスでログインできない時はログにこんな文言。
> `Filter chain halted as :require_login rendered or redirected`
> このとき、(多分)Domain属性のレベルのせいで、ブラウザによってはsessionがカラっぽなのでトップに戻されます。
>
> 同様事例の報告があったけど、その時はイマイチ対策ができなかった模様
> http://community.opscode.com/chat/chef/2014-05-29

