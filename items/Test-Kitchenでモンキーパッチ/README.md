<!-- permanent -->

古めのJoyent SmartOSでTest-Kitchenをしようとしたら、Ohai7.2以上,Chef11.8以上がさくっとインストール出来ない状態だった。
Omnibusにも対応してないのでプラグインを作ってRubyGemsからChefを入れていたけど、バージョン指定の処理を実装しておらず困ったので、とりあえずモンキーパッチすることにした。


## rubyコマンドでrequireする。

実行時に適当なRubyファイルを読ませるには-`r(require)`オプションを付けてRubyを実行すればよい。

```monkey_patch/hoge.rb
puts 'hogehoge'
```

ファイルを用意して`-r`

```shell:
$ ruby -r ./monkey_patch/hoge ./bin/kitchen -v
hogehoge
Test Kitchen version 1.2.1
```


ただ毎回rubyコマンドからkitchenするのは面倒。


## じゃあ.kitchen.ymlに書いちゃう 

driverで`ohai_version`、`chef_version`を指定したら、`gem install`時に指定するようにしたかったので、応急処置として`.kitchen.yml`でクラスをオープンしてしまうことにした。`def install_chef_for_smartos`をまるっとオーバーライド。

なんか色々と酷い気がするので常用はおすすめしません。

```yaml:.kitchen.yml
---
driver_plugin: zcloudjp
driver_config:
  api_key: PLEASE_REPLACE_IT_BY.kitchen.local.yml
  ohai_version: 7.0.4
  chef_version: 11.6.2

provisioner:
  name: chef_solo
  require_chef_omnibus: false
  data_path: <%= ENV['NO_DATA'] ? "null" : "data" %>

platforms:
- name: sm-base64
  driver_config:
    username: root
    metadata_file: tk/metadata.json
    dataset: "sdc:sdc:base64:1.9.1"
    package: xxx_xxx
    with_gcc: true

suites:
- name: default
  use_sudo: false
  run_list:
  attributes: {}


<%
## Monkey Patch for SmartOS 1.9.1
require 'kitchen/driver/zcloudjp'

module Kitchen
  module Driver
    class Zcloudjp
      class_eval do
        def install_chef_for_smartos(ssh_args)
          if config[:with_gcc]
            install_pkgs = "gcc47 gcc47-runtime scmgit-base scmgit-docs gmake ruby193-base ruby193-yajl ruby193-nokogiri ruby193-readline pkg-config"
          else
            install_pkgs = "scmgit-base scmgit-docs ruby193-base ruby193-yajl ruby193-nokogiri ruby193-readline"
          end

          ssh(ssh_args, <<-INSTALL.gsub(/^ {10}/, ''))
            if [ ! -f /opt/local/bin/chef-client ]; then
              pkgin -y install #{install_pkgs}

            ## for smf cookbook
              pkgin -y install libxslt

            ## install chef
              gem update --system --no-ri --no-rdoc
              gem install --no-ri --no-rdoc ohai #{config[:ohai_version] ? '--version ' + %Q{'=  #{config[:ohai_version]}'} : nil }
              gem install --no-ri --no-rdoc chef #{config[:chef_version] ? '--version ' + %Q{'=  #{config[:chef_version]}'} : nil }
              gem install --no-ri --no-rdoc rb-readline
            fi
          INSTALL
        end
      end
    end
  end
end
%>
```

### 外部ファイルにして.kitchen.ymlでロード

流石に`.kitchen.yml`が読みにくいのでloadにした。`SafeYAML::OPTIONS[:default_mode]`は警告の抑制。

```yaml:.kitchen.yml
...

<%
## Monkey Patch for SmartOS 1.9.1
SafeYAML::OPTIONS[:default_mode] = :unsafe
load File.expand_path('../monkey_patch/fix_install_for191.rb', __FILE__)
%>
```

なお、後日Gemのほうに反映してモンキーパッチは消しました。
