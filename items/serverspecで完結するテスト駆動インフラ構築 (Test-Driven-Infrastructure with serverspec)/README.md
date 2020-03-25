<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

サーバの構築状況を、手動確認の段取りでチェックする[serverspec](http://serverspec.org/)、使いやすくて評判ですね。

> 注: このエントリはほぼネタなのであまり参考にしてはいけません。

さて、テスト駆動開発という手法は皆さん既にご存知でしょう。今日は折角テストツールがあるのですからサーバ構築をテスト駆動でやってみましょう。

## まずFailするSpecサンプル

こちらはパッケージ`nginx`がインストールされているかテストするserverspecのサンプルです。

```ruby:nginx_spec.rb
require 'spec_helper'

describe package('nginx') do
  it { should be_installed }
end
```

### Failすることを確認

Failからはじまるテスト駆動、`serverspec`はテストに使ったコマンドも表示してくれます。

```shell:run_rspec_fails
Failures:

  1) Package "nginx" 
     Failure/Error: it { should be_installed }
       dpkg -s nginx && ! dpkg -s nginx | grep -E '^Status: .+ not-installed$'
       Package `nginx' is not installed and no info is available.
Use dpkg --info (= dpkg-deb --info) to examine archive files,
and dpkg --contents (= dpkg-deb --contents) to list their contents.
       expected Package "nginx" to be installed
     # ./spec/xxx.xxx.xxx.xxx/nginx_spec.rb:4:in `block (2 levels) in <top (required)>'

Finished in 1.01 seconds
1 examples, 1 failures
```

目出度くFailしました、ではこれを満たすようにサーバ構築のコードを書きましょう！

## Rescueせよ

(重ねて言いますがあまり真面目に受け取らないでください。)

構築には何を使いましょう、`chef`?`puppet`? いえ今日は趣向をこらして、Failを検知できるRspec(serverspec)自身に直させましょう。

ということで`nginx_spec.rb`をこの様に変更です。

```ruby:nginx_converge_spec.rb
require 'spec_helper'

describe package('nginx') do
  it {
    begin
      should be_installed
    rescue => e
      puts e.message
      puts "======= Installing Nginx..."
      cmd = Serverspec::Type::Command.new('apt-get update && apt-get install -y nginx')
      expect(cmd.return_exit_status?(0)).to be_true
    end
  }
end
```

nginxの`should be_installed`が例外を吐いたら`rescue`してサーバ状態のつじつまを合わせにいきます。
コマンド実行にも`serverspec`のマッチャを使うことで、楽にエラーチェックができますね！


### serverspecで完結、テスト駆動

では初めにFailしたのと同じ状態のサーバに`serverspec`を実行してみます。

```shell:run_rspec_converge
$ rspec -fd

Package "nginx"
expected Package "nginx" to be installed
======= Installing Nginx...
  should be true

Finished in 13.42 seconds
1 example, 0 failures
```

`expected Package "nginx" to be installed` と、rescue側に入ってつじつま合わせが走りました。
すんだらもう一度`serverspec`を実行してみます。

```shell:run_rspec_up_to_date
$ rspec -fd

Package "nginx"
  should be installed

Finished in 1.03 seconds
1 example, 0 failures
```

`should be installed #=> true`、できました。まさに冪等な収束。

## 終わりに

テスト駆動はソフト開発でもしばしば難題と言われますが、インフラはこのくらい緩めな手法でもアリかな。

----

##  追記

chef-soloを導入してcommunity cookbookから `rabbitmq` のインストール状態を確認し、入ってなければ勝手にインストールするserverspecはこちらです。
テストが通ればただのserverspecなところが肝心ですね。

```ruby:chef_and_rabbitmq_spec.rb
require 'spec_helper'

describe command('which chef-solo') do
  it {
    begin
      should return_exit_status(0)
    rescue => e
      puts e.message
      puts "======= Installing Omnibus Chef..."
      cmd = Serverspec::Type::Command.new('curl -L https://www.opscode.com/chef/install.sh | sudo bash')
      expect(cmd.return_exit_status?(0)).to be_true
    end
  }
end

describe package('git') do
  it {
    begin
      should be_installed
    rescue => e
      puts e.message
      puts "======= installing git..."
      cmd = serverspec::type::command.new('apt-get update && apt-get install -y git')
      expect(cmd.return_exit_status?(0)).to be_true
    end
  }
end

describe file('/var/chef/cookbooks/.git') do
  it {
    begin
      should be_directory
    rescue => e
      puts e.message
      puts "======= initializing cookbooks directory..."
      cmd = Serverspec::Type::Command.new('mkdir -p /var/chef/cookbooks')
      expect(cmd.return_exit_status?(0)).to be_true
      cmd = Serverspec::Type::Command.new('git init /var/chef/cookbooks')
      expect(cmd.return_exit_status?(0)).to be_true
      cmd = Serverspec::Type::Command.new('touch /var/chef/cookbooks/.gitkeep')
      expect(cmd.return_exit_status?(0)).to be_true
      cmd = Serverspec::Type::Command.new('cd /var/chef/cookbooks && git add . && git commit .gitkeep -m "empty init"')
      expect(cmd.return_exit_status?(0)).to be_true
    end
  }
end

describe package('rabbitmq-server') do
  it {
    begin
      should be_installed
    rescue => e
      puts e.message
      puts "======= installing rabbitmq by chef..."
      cmd = Serverspec::Type::Command.new('knife cookbook site install rabbitmq')
      expect(cmd.return_exit_status?(0)).to be_true
      cmd = Serverspec::Type::Command.new('chef-solo -o "rabbitmq"')
      expect(cmd.return_exit_status?(0)).to be_true
    end
  }
end
```

くれぐれもserverspec導入の参考にしないで下さい。
