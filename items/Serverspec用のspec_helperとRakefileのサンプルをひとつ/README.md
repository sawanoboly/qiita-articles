
Serverspecで実施するテストを適当に書いていたら、Rakefileとspec_helperが度重なる継ぎ足し建築で大きくなったので晒す。

この記事に出てくるファイルのリポジトリはこちら。 [OpsRockin/serverspec_helper_example](https://github.com/OpsRockin/serverspec_helper_example)

## テスト対象のはなし

相手はだいたいこんな感じ。

- ほぼ同じ構成で、staging, productionほか複数の環境がある
- 環境はプロビジョニングツールの管理下である
    - ツールではミドルウェア構成に対応する形のroleとして定義が作られている
    - 各ホストは大分類名(webとか)がついていて、roleの組み合わせで構築される

構築側のことはあまり気にせず、『仕様を探ってspecだけ書いてちょー』という条件の下でやってます。

### Rake & spec_helperで吸収したかったこと

- 大分類が同じホストが環境別に複数あったり無かったり
- 共通のroleもある
- たまにホスト特有の値がある
- 任意に対象を絞りたい
   - 環境の全ホスト
   - 大分類別に全ホスト
   - 特定のホスト
- たまにローカルのDockerで試したい。

ということで、roleに相当するspecを作って流せるようにしていった。 

[O'Reilly Japan - Serverspec](http://www.oreilly.co.jp/books/9784873117096/ "O'Reilly Japan - Serverspec") から、 **3.7 specファイルを複数のホストで共有** と **3.8 ホスト固有情報の利用** の応用・合わせ技ですね。


## 関連ファイルの構成

そのまま紹介とはいかないので、説明用に軽めにアレンジしてます。

```
２階層までのツリー。(※)はファイルで、印がないのはディレクトリ。

├── Gemfile(※)
├── Gemfile.lock(※)
├── Rakefile(※)
├── common_spec     # 他プロジェクトとも共有するspec、gitのサブモジュールだったりする。
│   ├── common
│   ├── group
│   ├── mackerel-agent
│   └── user
├── ferture_spec    # プロジェクト特有のspec、別に分けなくてもいい。
│   ├── fluentd
│   ├── nagios
│   ├── nginx
│   ├── postgresql
│   └── user
├── host_vars       # ホスト特有の値をロードしたいときなどに使うなど。
│   ├── 10.48.1.31.yml(※)
│   └── 10.48.2.31.yml(※)
├── hosts_production(※)   # production環境のホストとrole一覧
├── hosts_staging(※)      # staging環境のホストとrole一覧
└── spec                  # helper他。これもサブモジュールなどでもいい。
    └── spec_helper.rb(※) 
```


## Rakefile


早速なんかデカイ。多分余計な部分もある。
要は対象をyamlから抽出して、タスクを生成しています。


```ruby:Rakefile
require 'rake'
require 'yaml'
require 'rspec/core/rake_task'

## 環境変数 SPEC_ENV で環境名を指定。RackやRailsのパクリ。
spec_env = ENV['SPEC_ENV']
if spec_env
  path_candidate = File.expand_path("../hosts_#{spec_env}", __FILE__)
  if File.exists?(path_candidate)
    hosts_defined = path_candidate
  else
    raise RuntimeError, "\n======\nERROR: No hosts defined for #{spec_env}.\n======"
  end
else
  ## SPEC_ENV が省略されたらとりあえずstagingとしてあつかう。
  spec_env = 'staging'
  hosts_defined = File.expand_path("../hosts_staging", __FILE__)
end


## 環境名に対応する定義ファイルを読む
properties = YAML.load_file(hosts_defined)

task :spec    => 'spec:all'
task :default => :spec

namespace :spec do
  ## 定義ファイルから spec:大分類:ホスト名を全部作成する。spec:all 用
  all_tasks = properties.each_pair.map { |key, values|
    ## 共通設定はホスト扱いしない。
    next if key == 'shared_settings'
    values[:hosts].map {|host| 'spec:' + key + ':' + host }
  }.flatten.compact

  ## 全部実行するタスク (spec:all)
  desc "all target for #{spec_env}"
  task :all => all_tasks

  ## ホスト定義をまわす、大分類はmaster_rollって名前で扱う
  properties.each_pair do |master_roll, entries|
    ## 共通設定は大分類扱いしない(spec_helperで使う)
    next if master_roll == 'shared_settings'

    ## 大分類に割り当てられているroleを抽出する
    role_pattern = entries[:roles].join(',')

    namespace master_roll.to_sym do
      hosts = entries[:hosts]

      ## 大分類別に全ホスト実行するタスク (spec:大分類:all)
      desc "all target of #{master_roll} for #{role_pattern}"
      task :all => hosts.map {|h| 'spec:' + master_roll + ':' + h }

      ## 大分類別に個別ホスト実行するタスクを定義する (spec:大分類:ホスト名)
      hosts.each do |host|
        desc "Run serverspec tests to #{master_roll}: #{host} for #{role_pattern}"
        RSpec::Core::RakeTask.new(host.to_sym) do |t|
          ## どれかがこけても途中でやめない。
          t.fail_on_error = false
          ENV['TARGET_HOST'] = host
          ENV['SPEC_ENV'] = spec_env

          ## specとcommon_specとferture_specをざっくり取って、定義ファイル上のロールに対応するspecを読み込ませる。
          t.pattern = "{spec,common_spec,ferture_spec}/{#{role_pattern}}/**/*_spec.rb"
        end
      end
    end
  end
end
```

`spec:大分類:all`のあたりがややこしいです。あると便利なので少々強引に。
ちなみに定義ファイル側のroleに対応するspecのディレクトリは無くてもスルーされます。ある分だけテスト実施。


### yamlに大分類と所属ホストを書いた

じゃあ大分類が`web`, `db`, `press`の3つあるとします。

hosts_stagingとhosts_productionというファイルをそれぞれ書いてみるとこんな感じ。

大分類と所属ホスト、ロードするspec(roleにおおまかに対応)という構成です。 SSHの環境別接続設定に継ぎ足し感あります。

```yaml:hosts_staging
---
shared_settings:
  :ssh_opts:
    :user: operator
    :keys: /Users/sawanoboriyu/.ssh/my_staging_key
    :port: 22
    :paranoid: false
web:
  :hosts:
    - 192.168.1.11
    - 192.168.1.12
  :roles:
    - common
    - group
    - user/system
    - user/web
    - fluentd
    - mackerel-agent
    - nagios/nrpe
    - nginx
db:
  :hosts:
    - 192.168.1.31
  :roles:
    - common
    - group
    - user/system
    - user/db
    - fluentd
    - mackerel-agent
    - nagios/nrpe
press:
  :hosts:
    - 192.168.1.21
  :roles:
    - common
    - group
    - user/system
    - user/press
    - mackerel-agent
    - nagios/nrpe
    - nagios/server
```

productionはsshの設定が違ったり、ちょっとホストが多かったり。

```yaml:hosts_production
---
shared_settings:
  :ssh_opts:
    :user: opera_singer
    :keys: /Users/sawanoboriyu/.ssh/my_production_key
    :port: 9022
    :paranoid: false
web:
  :hosts:
    - 10.48.1.11
    - 10.48.1.12
    - 10.48.2.11
    - 10.48.2.12
  :roles:
    - common
    - group
    - user/system
    - user/web
    - fluentd
    - mackerel-agent
    - nagios/nrpe
    - nginx
db:
  :hosts:
    - 10.48.1.31
    - 10.48.2.31
  :roles:
    - common
    - group
    - user/system
    - user/db
    - fluentd
    - mackerel-agent
    - nagios/nrpe
press:
  :hosts:
    - 10.48.1.21
    - 10.48.2.21
  :roles:
    - common
    - group
    - user/system
    - user/press
    - mackerel-agent
    - nagios/nrpe
    - nagios/server
```

こういう定義一覧をプロビジョニングツールと連動するかは時と場合によります。
両方修正が面倒な場合や、１つの修正ミスがプロビジョニングとテストの両方の結果に影響するのを避けたいか。ケースによって変えましょう。
今回のはテスト側にあるていど独立性を確保した感じですね。


### rakeで対象を確認する

前項のyaml達をRakeに通すとどうなるか、タスク一覧を表示してみます。

SPEC_ENVを省略した場合はstatingが対象です。
全ホストを対象にする `spec:all`, webの全ホストを対象の`spec:web:all`や任意ホストの`spec:web:192.168.1.11`をそれぞれ実行できるようになりました。

```shell:
$ ./bin/rake -vT
rake spec:all                 # all target for staging
rake spec:db:192.168.1.31     # Run serverspec tests to db: 192.168.1.31 for common,group,user/system,user/db,fluentd,mackerel-agent,nagios/nrpe
rake spec:db:all              # all target of db for common,group,user/system,user/db,fluentd,mackerel-agent,nagios/nrpe
rake spec:press:192.168.1.21  # Run serverspec tests to press: 192.168.1.21 for common,group,user/system,user/press,mackerel-agent,nagios/nrpe,nagios/server
rake spec:press:all           # all target of press for common,group,user/system,user/press,mackerel-agent,nagios/nrpe,nagios/server
rake spec:web:192.168.1.11    # Run serverspec tests to web: 192.168.1.11 for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
rake spec:web:192.168.1.12    # Run serverspec tests to web: 192.168.1.12 for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
rake spec:web:all             # all target of web for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
```

`SPEC_ENV=production`で一覧を出すと、`hosts_production`からタスクを生成します。ロードするspecも表示するようになってます。

```shell:
$ SPEC_ENV=production ./bin/rake -vT
rake spec:all               # all target for production
rake spec:db:10.48.1.31     # Run serverspec tests to db: 10.48.1.31 for common,group,user/system,user/db,fluentd,mackerel-agent,nagios/nrpe
rake spec:db:10.48.2.31     # Run serverspec tests to db: 10.48.2.31 for common,group,user/system,user/db,fluentd,mackerel-agent,nagios/nrpe
rake spec:db:all            # all target of db for common,group,user/system,user/db,fluentd,mackerel-agent,nagios/nrpe
rake spec:press:10.48.1.21  # Run serverspec tests to press: 10.48.1.21 for common,group,user/system,user/press,mackerel-agent,nagios/nrpe,nagios/server
rake spec:press:10.48.2.21  # Run serverspec tests to press: 10.48.2.21 for common,group,user/system,user/press,mackerel-agent,nagios/nrpe,nagios/server
rake spec:press:all         # all target of press for common,group,user/system,user/press,mackerel-agent,nagios/nrpe,nagios/server
rake spec:web:10.48.1.11    # Run serverspec tests to web: 10.48.1.11 for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
rake spec:web:10.48.1.12    # Run serverspec tests to web: 10.48.1.12 for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
rake spec:web:10.48.2.11    # Run serverspec tests to web: 10.48.2.11 for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
rake spec:web:10.48.2.12    # Run serverspec tests to web: 10.48.2.12 for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
rake spec:web:all           # all target of web for common,group,user/system,user/web,fluentd,mackerel-agent,nagios/nrpe,nginx
```

## spec_helper

spec_helperも肥大。

バックエンドのdocker or ssh(および環境別オプション)を選択可能にしたり、ホスト特有の値をロードという機能を継ぎ足したりしていったのでこんな感じ。

```ruby:spec_helper.rb
require 'serverspec'
require "docker"
require 'net/ssh'
require 'yaml'

case ENV['SPEC_BACKEND']
## 環境変数 SPEC_BACKEND がdocker|DOCKERだったらSSHじゃなくてDockerバックエンドを使う。
when "DOCKER", 'docker'
  set :backend, :docker
  set :docker_url, ENV['DOCKER_HOST'] || 'unix:///var/run/docker.sock'
  ## Dockerでためす場合、DOCKER_IMAGEを指定する。
  set :docker_image, ENV['DOCKER_IMAGE']
  set :docker_container_create_options, {'Cmd' => ['/bin/sh']}
  Excon.defaults[:ssl_verify_peer] = false
else
  ## デフォルトのバックエンドはSSH
  set :backend, :ssh
  set :request_pty, true

  ## このへんはRakeと一緒、定義ファイルを決定
  spec_env = ENV['SPEC_ENV']
  if spec_env
    path_candidate = File.expand_path("../../hosts_#{spec_env}", __FILE__)
    puts path_candidate
    if File.exists?(path_candidate)
      hosts_defined = path_candidate
    else
      raise RuntimeError, "\n======\nERROR: No hosts defined for #{spec_env}.\n======"
    end
  else
    hosts_defined = File.expand_path("../../hosts_staging", __FILE__)
  end


  ## spec_helperでもRakefile同様にホスト定義を読み込む
  properties = YAML.load_file(hosts_defined)

  host = ENV['TARGET_HOST']
  mainrole = properties.select {|k,v| v[:hosts].include?(host) if v[:hosts] }.keys.first

  ## ホスト固有の値を書いたファイルがあればつかう。
  host_vars = YAML.load_file(
    File.expand_path("../../host_vars/#{host}.yml", __FILE__)
  ) if File.exists?(File.expand_path("../../host_vars/#{host}.yml", __FILE__))

  spec_property =  properties[mainrole]
  spec_property[:host_vars] =  host_vars ||= {}

  ## 環境変数DEBUGがあったらset_propertyに渡される値を表示する
  puts spec_property.to_yaml if ENV['DEBUG']

  set_property spec_property

  ## specの中で大分類を使うかもしれないと思ってとりあえず環境変数に突っ込んである。
  ENV['SPEC_MAINROLE'] = mainrole

  ## 環境別SSH接続設定をマージしていく
  options = Net::SSH::Config.for(host).merge(properties['shared_settings'][:ssh_opts])

  ### 大分類の下にもssh_optsがあったらそっちを優先で上書き
  options.merge!(properties[mainrole][:ssh_opts]) if properties[mainrole][:ssh_opts]
  options[:user] ||= 'root'
  options[:keys] ||= File.expand_path("#{ENV['HOME']}/.ssh/my_staging_key" ,__FILE__)

  set :host,        options[:host_name] || host
  set :ssh_options, options

  # Disable sudo
  # set :disable_sudo, true

  RSpec.configure do |config|
    config.color = true
    config.tty = true
  end

  # Set environment variables
  set :env, :LANG => 'C', :LC_MESSAGES => 'C'
end
```

いくらか冗長だったり、initの分が残ってたりです。やってることは単純。


### とりあえずDockerで

大分類`db`相当のイメージをビルドしたら、該当する`spec:db:all`なりでspecを実行してきてもらったり。
テスト内容はダミーです。

```shell:
$ SPEC_BACKEND=docker DOCKER_IMAGE=db ./bin/rake spec:db:all
(省略) rspec --pattern \{spec,common_spec,ferture_spec\}/\{common,group,user/system,user/db,fluentd,mackerel-agent,nagios/nrpe\}/\*\*/\*_spec.rb

Command "id"
  exit_status
    should eq 0

File "/etc"
  should be directory

Finished in 4.18 seconds (files took 0.54121 seconds to load)
2 examples, 0 failures
```

実際使っているspec内ではいくつかの箇所でSPEC_BACKENDがDockerなら実施しないテストなどを分岐しました。


## おわりに

Serverspec(ここはRSpecとしてもいいけど)は最終的に個々のタスクを別々にこなすようになっているので、好きなようにタスクを生成して乱雑にわたしてもきっちり動いてくれますね。

