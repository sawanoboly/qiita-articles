

Ruby(Puma)を使う場合の話。SingleならSGでいいんだけど。


- 環境ごとに許可対象を管理したい。
- 許可対象の変更でアプリごとデプロイは嫌。
- 許可対象の追加削除は自動で適用してほしい。

サンプルのSinatraアプリはこちら。 [https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB](https://github.com/ZCloud-Firstserver/eb_nginxBrockRubyviaELB)


## ELB x Nginx

周知のとおりELBは一般的なLBと同様に`X-Forwarded-For`にクライアントのIPを入れてきます。
なのでnginxにはこんな感じで設定しておけば接続元で`allow/deny`をつかえます。

```
set_real_ip_from 0.0.0.0/0;
real_ip_header X-Forwarded-For;
allow xxx.xxx.xxx.xxx/32;
deny all;
```

## Hookに仕込む

環境ごとに値を変えたいので、`ALLOW_HOSTS`なる変数をEBに置いて、カンマ区切りで許可対象を与えられるようにします。
Ruby(Puma)環境のNginxは`/etc/nginx/conf.d/*.conf`を読むようになっているので、ポリシーはそこに設置しましょう。

`Configration`を変更したら適用してほしいので、Hookは`configdeploy::enact`にしてみます。アクセスポリシーファイルの生成とNginxのリロードを設置。
インスタンスの追加や環境のリビルド時やにも有効にするため、`postinit`にも同じものを置いときます。


```yaml:.ebextensions/01_nginx_access.config
---
files:
  "/opt/elasticbeanstalk/hooks/configdeploy/enact/10_nginx_access.rb":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env ruby

      require '/opt/elasticbeanstalk/support/get_envvars'
      require 'erb'

      envs = get_env_vars

      erb = ERB.new(DATA.read, nil, '-')
      File.open("/etc/nginx/conf.d/access.conf", "w") do |file|
        file.puts(erb.result(binding))
      end


      __END__
      set_real_ip_from 0.0.0.0/0;
      real_ip_header X-Forwarded-For;
      <% envs['ALLOW_HOSTS'].split(',').each do |host| -%>
      allow <%= host -%>;
      <% end if envs['ALLOW_HOSTS'] -%>
      deny all;
  "/opt/elasticbeanstalk/hooks/configdeploy/enact/11_nginx_reload.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env bash

      set -xe
      service nginx reload \|\| service nginx start
  "/opt/elasticbeanstalk/hooks/postinit/10_nginx_access.rb":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env ruby

      require '/opt/elasticbeanstalk/support/get_envvars'
      require 'erb'

      envs = get_env_vars

      erb = ERB.new(DATA.read, nil, '-')
      File.open("/etc/nginx/conf.d/access.conf", "w") do |file|
        file.puts(erb.result(binding))
      end


      __END__
      set_real_ip_from 0.0.0.0/0;
      real_ip_header X-Forwarded-For;
      <% envs['ALLOW_HOSTS'].split(',').each do |host| -%>
      allow <%= host -%>;
      <% end if envs['ALLOW_HOSTS'] -%>
      deny all;
  "/opt/elasticbeanstalk/hooks/postinit/11_nginx_reload.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env bash

      set -xe
      service nginx reload \|\| service nginx start
```

## eb setenvでアクセスポリシー操作


configdeployのフックが無事設置できれば、あとはWebからでも`eb`からでも、ALLOW_HOSTSを適当に変更すればOK.

```
$ eb setenv ALLOW_HOSTS="xxx.xx.xxx.xxx/32,xxx.xxx.xxx.xxx/28" -e YOUR_ENV_NAME
```

![allow_hosts.png](https://qiita-image-store.s3.amazonaws.com/0/7454/246b3c30-8313-e3ee-3fdd-a24be965e9b0.png "allow_hosts.png")

↑Configureに許可リスト。

↓ELB配下へのRubyアプリへのアクセス。

![nginx_403.png](https://qiita-image-store.s3.amazonaws.com/0/7454/2afc515d-82b7-d1d1-9736-2faaae952e08.png "nginx_403.png")


