<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

[Opscode Chef](http://www.opscode.com/chef/)であるrailsアプリをデプロイするレシピを書いたのでメモ。

## 本Railsデプロイ概要

- 前提: rubyが入っている、postgresqlサーバは既にある
- start, stopは専用スクリプトを用意、`monit`配下で管理する。
- デプロイのプラットフォームはとりあえず`Joyent SmartOS`、多分ほかでもあまり違いはない。
- Rackupはunicorn, socketでnginxからリバースプロキシ
- `asset_pipiline`を使い、`assets`以下は`nginx`でホスト。

## レシピ

関連`Cookbooks`や`data_bags`のことは考えず、固有名称や複雑な箇所は説明用に改変してます。
`chef`の`deploy`リソースは基本的に`capistrano/deploy`のように決められたフェーズがあり、それぞれにフックが設定されています。
フック時に実行する内容は**Railsアプリケーションの**リポジトリ側に`deploy/#{hook_name}.rb`のように置いておくこともできます。

`## // `ではじまる行が書き下ろしの解説

```ruby:rails_app/deploy.rb
#
# Cookbook Name:: rails_app
# Recipe:: deploy
#

## // デプロイする先のディレクトリを適当に設定
app_name = 'my_app'
dir_localsystem = ::File.join(node[:my_app_common][:basedir], app_name, 'shared/localsystem')


## // nodeのenvironmentをキーにして、各種接続情報をdata_bagsから取ってくる
## build settings
## // ブランチ情報やdeploy or force_deployを選べる
deploy_settings = data_bag_item(:deploy,:items)[node.environment]
## // Postgresのrole, password
connection= data_bag_item(:postgresql,:connections)[node.environment]
## // 色々URLを拾ってくる、ここではNginxの仮想ホストや、アプリケーションのコンフィグで使用
domains = data_bag_item(:domains,:items)[node.environment]
## // Postgresqlの構成次第で接続先を変えたい
pg_master = data_bag_item(:postgresql,:clusters)[node.environment]['master']


## // ここから`deploy`リソース
deploy_branch ::File.join(node[:my_app_common][:basedir], app_name)  do

  ## // デプロイ基本情報
  repo 'https://github.com/higanworks/my_app.git'
  branch  deploy_settings[app_name]['ref'] ## // 'master' が入ったりする
  action deploy_settings[app_name]['action'] ## // 'deploy' または 'force_deploy'や 'roleback'とか
  migrate deploy_settings[app_name]['migrate'] ## // true or false


  ## // 各種コマンド実行時の環境変数
  environment "RAILS_ENV" => node.environment,  'BUNDLE_GEMFILE' => 'MyGemfile'

  ## // マイグレーションのフェーズで実行するコマンド。
  migration_command 'bundle exec rake db:migrate'

  ## // デプロイ基本情報
  create_dirs_before_symlink %w{tmp}

  ## // 標準のsymlinkにassetsを追加している
  symlinks "system"=>"public/system",
           "assets"=>"public/assets",                                                                                               
           "pids"=>"tmp/pids",
           "log"=>"log"

  ## // マイグレーション前のフック、sharedの下に吐いたコンフィグをsymlinkにして`rails_root`の下に持ってくる
  ## // キー側は `shared_dir`からの相対パス、ターゲット側は`release_path`からの相対パスで指定
  symlink_before_migrate "config/database.yml" => "config/database.yml",
                         "config/app_config.yml" => "config/app_config.yml"

  ## // 以降`relase_path`は使えるけど、`shared_path`が使えたり使えなかったりする。。
  ## // Shared_Dirの下に必要なディレクトリを置く、Docによるつデフォルトで幾つか作りそうだけど作られなかったので明示。
  before_migrate do
    %w{config pids log assets system localsystem}.each do |dir|
      directory ::File.join(release_path, "../../shared" , dir) do
        action :create
      end
    end

    ## // database.yml の生成
    ## // dbの名前は"#{app_name}_#(node.environment)"
    template ::File.expand_path(::File.join(release_path, '../../shared/config/database.yml')) do
      cookbook 'my_app_common'
      source 'database.yml.erb'
      variables({
        :connection => connection,
        :domains => domains,
        :pg_master => pg_master,
        :app_name => app_name
      })
    end

    ## // アプリケーション固有のコンフィグを生成
    template ::File.expand_path(::File.join(release_path, '../../shared/config/app_config.yml')) do
      source 'app_config.yml.erb'
      variables({
        :domains => domains
      })
    end

    ## // unicornの設定
    ## // ワーカープロセス数をdata_bagから
    ## // socketのパスに`app_name`を使用
    template ::File.expand_path(::File.join(release_path, '../../shared/config/unicorn.rb')) do
      cookbook 'nginx_upstream'
      source 'unicorn.rb.erb'
      variables({
        :worker_processes => node[:deploy][app_name][:unicorn_wps],
        :app_name => app_name
      })
    end

    ## // bundlerを使っているのでマイグレーション実行前にbundle
    ## // Gemを似たようなrailsアプリと共有するため、一段階上のSharedに置く
    bash "bundle install #{app_name}" do
      cwd release_path
      flags '-ex'
      code <<-BASHCODE
         bundle install --deployment --gemfile MyGemfile --without development test --path=#{::File.join(node[:my_app_common][:basedir], 'shared/bundle')}
      BASHCODE
    end
  end



  ## // リスタート前のフック
  before_restart do

    ## // data_bagに指定があったらassetsを生成する
    ## // RAILS_GROUPSの指定をするのでenviromentを設定
    ## asset pipeline
    if deploy_settings[app_name]['assets']
      bash "assets precompile #{app_name}" do
        cwd release_path
        environment "RAILS_ENV" => node.environment,
                    'BUNDLE_GEMFILE' => 'MyGemfile',
                    'RAILS_GROUPS' => 'assets'
        flags '-ex'
        code <<-BASHCODE
           bundle exec rake assets:precompile
        BASHCODE
      end
    end

    ## // monitからコントロールするため、制御スクリプトを設置する
    ## // 中身にはPATHやらENVやら色々設定してある
    %W(start_#{app_name}.sh stop_#{app_name}.sh).each do |cmd|
      template ::File.join(dir_localsystem, cmd) do
        source cmd + '.erb'
        mode '0700'
        variables({
          :app_name => app_name,
          :pid_file => 'unicorn.pid',
          :bundle_command => "bundle exec unicorn_rails -c #{::File.expand_path(::File.join(release_path, '../../shared/config/unicorn.rb'))} -D"
        })
      end
    end

    ## // monitのコンフィグ設置/更新を監視してリロードする 
    ## // 初回や、現在プロセスが起動してない場合などはInitalizeに少々もたつくのでSleepでwait. 
    ## // waitはだいたい5秒くらい
    execute 'monit_reload_' + app_name do
      action :nothing
      command "monit reload ; sleep #{node[:deploy][:monit][:sleep]}"
      subscribes :run, "template[/opt/local/etc/monit/conf.avail/#{app_name}]"
    end

    ## // monitのコンフィグファイルをincludeパスにリンクする
    execute 'monit_ensite_' + app_name do
      action :nothing
      command "monitensite #{app_name}"
      notifies :run, "execute[monit_reload_#{app_name}]"
      not_if "test -f #{node[:monit][:dir]}/conf.enable/#{app_name}"
    end

    ## // monit_binのLWRPを使ってコンフィグを生成する
    monit_bin_check_process app_name do
      action :create
      type "pid"
      pidfile ::File.join(node[:my_app_common][:basedir], app_name , 'shared/pids/unicorn.pid')
      start_program ::File.join(node[:my_app_common][:basedir], app_name , 'shared/localsystem', "start_#{app_name}.sh")
      stop_program ::File.join(node[:my_app_common][:basedir], app_name ,'shared/localsystem', "stop_#{app_name}.sh")
      notifies :run, "execute[monit_ensite_#{app_name}]"
    end
  end

  ## // リスタートのコマンド。
  ## // 初回はrestartにならないが問題ない。
  restart_command "monit restart #{app_name}"
  after_restart nil
end


## // アプリケーションログのローテーションを設定しておく
## // mvは色々面倒なので copy & truncate
logadm "app_#{app_name}" do
  path ::File.join(node[:my_app_common][:basedir], app_name , 'shared/log/*.log')
  copy true
  size '100m'
  period '7d'
  action :create
end

### add nginx upstream

## // nginxにホストさせる
## // domainsのデータバックからSSLのヴァーチャルホスト名を拾ってきている
## // 名前ベースのSSL(SNI)で複数のアプリケーションを実行
## // NginxでのUpstream設定やassets pipelineまわりは RailsCastsの#341を参考にしている
template ::File.join(node[:nginx_upstream][:conf_dir] , "conf.d/#{domains[app_name]}.conf") do
  cookbook 'nginx_upstream'
  source 'upstream_ssl.erb'
  variables({
    :app_name => app_name,
    :assets => deploy_settings[app_name]['assets'],
    :server_name => domains[app_name],
    :upstreams => ["unix:#{::File.join(node[:my_app_common][:basedir], app_name , 'shared/pids/unicorn.sock')}"]
  })
  notifies :restart, 'service[nginx]'
end
```

## 所感

- `capistrano`が使えれば、流れは同じなのでわかりやすい
- data_bagを使う所はCookbookにするほうが版の管理ができていいかもしれない。
- 開発中のデプロイ定期実行はアリ
- 本番更新には色々あるので、`node`のデフォルトランリストは環境周りの整備にしておきそれを定期実行、deploy, rollbackのランリスト(role)を指定して`knife`から流してあげるのがいいかもしれない。


----
追記ログ

- `symlink_before_migrate`でassetsを作るとなぜか`assets/assets`が増えたので`symlink`に移動
