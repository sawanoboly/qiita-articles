
今日は次のような仕様でnginxにリバースプロキシさせたかった。

- http://foo.example.higan.works/index.html で来たリクエストに対して、 (fooは任意)
- コンテンツを バックエンド `example.higan.works` の `/foo/index.html` から取ってくる。
    - バックエンドへのリクエストは => `http://example.higan.works/foo/index.html` になる


仕組みはこうしてみればいいか。

- ディレクトリ名の抽出
- `Host:`ヘッダの付け替え(バックエンドが名前ベースなので)
- uriにディレクトリ名を挟む

nginx単品でもifからの補足マッチで頑張ればできそうな気がしたが、ngx_mrubyが使える環境なので使ってみた。


```nginx
-- snip --

env UPSTREAM_DOMAIN;

-- snip --

http {

-- snip --

    server {
        listen       80;
        server_name  localhost;

        location / {
          resolver 8.8.8.8;
          mruby_set_code $backend '
            ENV["UPSTREAM_DOMAIN"]
          ';

          mruby_set_code $subdir_name '
            r = Nginx::Request.new
            subdir_name =  r.headers_in["Host"].split(".").first
            r.headers_in["Host"] = ENV["UPSTREAM_DOMAIN"]
            subdir_name
          ';

          rewrite ^(.*)$ /$subdir_name$1 break;
          proxy_pass http://$backend;
          proxy_intercept_errors on;
        }

        # redirect server error pages
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
        error_page  404              /404.html;
        location = /404.html {
            root   html;
        }
        error_page  403              /403.html;
        location = /403.html {
            root   html;
        }
    }
}
```

まあ何とか読みやすい。mruby_set_code内でヘッダを付け替えるのはどーなんだろと思いつつ動いたのでよし。
