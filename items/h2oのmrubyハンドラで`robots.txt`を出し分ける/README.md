[諸処の事情](https://getshifter.io)で、本番とは違うドメインで起動して操作しているWordPressなどがGoogleなどにインデックスされてイヤン、ということもあるでしょう。

いちいちrobotsを切り替えながら操作するのも手間なので、[h2o](https://h2o.examp1e.net)に組み込まれているmrubyで出力をコントロールした。

```ruby:robots.rb
# デフォルトのDisallowルール
robots = <<EOL
User-Agent: *
Disallow: /
EOL

lambda do |env|
  # /robots.txt以外のリクエストはすぐ処理をh2oに返す
  unless env["PATH_INFO"] == '/robots.txt'
    return [399, {}, []]
  end

  # 任意の条件で、これまた処理をh2oに返す。これはフラグファイル有無での例
  # 環境変数で判断とかでもよい
  if File.exist?('/path/to/flagfile')
    return [399, {}, []]
  end

  # Cookieに`wordpress_logged_in_*` => WordPressにログイン済みっぽいとして扱う
  if env["HTTP_COOKIE"]
    cookies = env["HTTP_COOKIE"].split
    if cookies.select {|a| a.match(/^wordpress_logged_in_/)}.any?
      # カスタマイズしたときのプレビューが出来ないと不便なので、ユーザ定義のrobots.txtを見せる
      return [399, {}, []]
    end
  end

  # これまでの条件にマッチしなければ、デフォルトのDisallowルールを適用
  return [200, {'Content-Type' => 'text/plain;charset=utf-8'}, [robots]]
end
```

rackベースなのでやりやすい。
