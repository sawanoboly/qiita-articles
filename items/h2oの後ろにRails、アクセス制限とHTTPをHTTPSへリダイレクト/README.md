
先人たちが色々踏んでおり、h2oとRailsの組み合わせでの問題は特に無くなっているような気がしたので使ってみた。


h2oはただの`proxy.reverse`でも色々よしなするようで、設定はとても少なくて済む。
ファイルがあったら〜な処理も並べるだけで、次へ次へと進んでいく。ほかはセキュリティ要件などに応じて、それ系のヘッダをつけておく位でよいかな。

```yaml:h2o.conf
access-log: /dev/stdout
user: nobody

hosts:
  "*":
    listen: 80
    listen:
      port: 443
      ssl:
        certificate-file: certs/server.crt
        key-file: certs/server.key
    paths:
      "/":
        mruby.handler-file: limit_access.rb
        file.dir: /srv/app/public
        proxy.reverse.url: http://rails:8080/
        proxy.preserve-host: ON
```

これはDocker-Compose管理で、`rails`サービスにlinkしています。なので`file.dir`に指定しているpublicはVOLUMEです。


## アクセス制限とHTTP接続のリダイレクト

今回これにテスト環境用のアクセス制限と、HTTP=>HTTPSへのリダイレクトをつけることにしました。だったらmrubyかなー。

アクセス制限は下記で紹介されているmruby-ipaddress_matcherでします。

- 参考: [kaihar4.com - mruby-ipaddress_matcherを使ってh2oでIPアドレスベースのアクセス制御をする](https://kaihar4.com/2015/11/01/mruby-ipaddress_matcher.html)

で、リダイレクトの要件、ホスト名をコンフィグに書いてしまってよいならh2oの`redirect`でいいんですが、実はドメインがまだ決まってないとかそういうこともあるのでアクセス制限のついでにやることにしました。

IP制限部分は参考サイトにちょっと環境変数での分岐をつけて、IPを評価するまえに`env['rack.url_scheme']`をチェックして`http`なら出なおしてもらう処理を追加しました。もちろんハンドラを分けても大丈夫のはず。

```ruby:limit_access.rb
case ENV['ACL_ENVIRONMENT']
when 'public'
ALLOW_CIDRS = %w(
0.0.0.0/0
)
else
ALLOW_CIDRS = %w(
x.x.x.x/32
x.x.x.x/32
x.x.x.x/28
)
end

class Acl
  def initialize(*args)
    @allow_regexps = ALLOW_CIDRS.map {|cidr| IPAddressMatcher::CIDR.new(cidr).to_regexp }
    super
  end

  def call(env)
    if env['rack.url_scheme'] == 'http'
      return [301, {'Location' => "https://#{env['HTTP_HOST']}/"}, []]
    end
    if @allow_regexps.select {|allow_regexp| env['REMOTE_ADDR'] =~ allow_regexp }.empty?
      [403, {'Content-Type' => 'text/plain;charset=utf-8'}, ['Forbidden']]
    else
      [399, {}, []]
    end
  end
end

Acl.new
```

なお、ここに書いた例ではenv['HTTP_HOST']を直接Locationに使ってますが、HTTP1.0などで来られてHostヘッダがないとここはSERVER_NAMEと同じ値が入ります。先のh2oコンフィグでは"*"になっちゃいますね。
`HTTP_HOST`が空なら`REMOTE_ADDR`を使うようにすれば良いケースもあります。それもDockerの内部ネットワークを使うとコンテナのIPになってたりして微妙。まあ環境に応じて適当に変更します。


ついでにHSTSをしたい場合は`header.append: "Strict-Transport-Security: max-age=39420000"`と入れたりします。

