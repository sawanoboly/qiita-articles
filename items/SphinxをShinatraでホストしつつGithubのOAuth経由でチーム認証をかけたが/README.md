

[Sphinxで作成したドキュメントをherokuでホスティング、Continuous Documentationなススメ(Basic認証付き)](http://qiita.com/sawanoboly@github/items/be1dbc9f19e93e4b62cf) の外伝。
『Basic認証』 => 『Github認証+特定のチームに属するかで閲覧可』 に変更してみたコード。

## 困った

### 認証箇所で困った

- Basic認証はRackモジュールだったので`publix`含む全リクエストが対象だった
- Github連携の`sinatra_auth_github`はSinatraの仕様に依存、=> `public`へのアクセスにフィルタがかけられない

> = 1.0 / 2010-03-23
> -- snip --
>  * Filters do not run when serving static files anymore. (Ryan Tomayko)
> [https://github.com/sinatra/sinatra/blob/master/CHANGES](https://github.com/sinatra/sinatra/blob/master/CHANGES)

publicに依存しまくっているSphinxホスティングはsinatra1.0でのこの変更が非常に厳しい。



### モンキーパッチで解決してしまった

> Important: The following notes on Sinatra::Base and Sinatra::Application are provided for background only - extension authors should not need to modify these classes directly.
> [http://www.sinatrarb.com/extensions.html](http://www.sinatrarb.com/extensions.html)

すまない公式、`Sinatra::Base`オープンです。`before`フィルタを`public(static)`配信時にも実行するように猿りました。


## 問題のコード



```ruby:config.ru
require 'sinatra'
require 'sinatra_auth_github'
set :public_folder, File.dirname(__FILE__) + '/build/html'

use Rack::Session::Cookie,
  :secret => Digest::SHA1.hexdigest(rand.to_s),
  :expire_after => 3600

set :github_options, {
  :scopes    => "user",
  :secret    => ENV['GITHUB_CLIENT_SECRET'],
  :client_id => ENV['GITHUB_CLIENT_ID'],
}

@@github_team = ENV['GITHUB_TEAM_ID']

register Sinatra::Auth::Github


## Monkey for static filter
module Sinatra
  class Base
    class_eval do
      alias :org_static! :static!
      def static!
        filter! :before
        org_static!
      end
    end
  end
end

## user is team_member?
before do
  if request.path_info.end_with?('.html')
    begin
      body = github_team_authenticate!(@@github_team)
    rescue
      ## return securocat!
      halt 401, body
    end
  end
  if request.path_info == '/logout'
    logout!
    redirect 'https://github.com'
  end
end

get '/' do
  redirect "/index.html", 301
end

not_found do
  unless request.path_info.end_with?('.ico')
    redirect "#{request.path_info}.html", 301
  end
  halt 404
end

run Sinatra::Application
```

## 認証

とりあえず`*.html`へのリクエストに対して、Github未ログインorアプリ認証許可なしならGithubへ飛ばします。Github認証が済んでおり、特定のチームに所属するならドキュメントを閲覧OK。

そうでない奴は`securocat`さんにご登場願う仕様になりました。

![Denied.png](https://qiita-image-store.s3.amazonaws.com/0/7454/6e22fa97-da13-c272-c711-20e000c1285d.png "Denied.png")


## モンキーパッチ脱出案？

- RackのミドルウェアとしてSinatraの前で認証をかける
- `public`を使わずあくまで`get`で処理し、`File.exists?`と`file_send`で頑張る

どうしたものかと。
