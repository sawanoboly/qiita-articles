

最近[opscode/omnibus](https://github.com/opscode/omnibus "opscode/omnibus")でServerspecの非公式パッケージを作っています。

[opscode/omnibus](https://github.com/opscode/omnibus "opscode/omnibus")については大体の説明はこちら。3以降はまたちょっと方針違います。

> [Omnibus-rubyプロジェクトでツールの周辺依存をまるごとrpm,debに固める - Qiita](http://qiita.com/sawanoboly/items/a2c258a235824b91b70f "Omnibus-rubyプロジェクトでツールの周辺依存をまるごとrpm,debに固める - Qiita")

パッケージ作成とリリースを、手元でやるとうるさいのでCircleCIにやらせてみた記録。

リポジトリはここ： https://github.com/OpsRockin/omnibus-serverspec


## circle.ymlを晒す

`circle.yml`本体に特殊な要素はなし。


```circle.yml
---
---
general:
  artifacts:
    - "pkg"
machine:
  ruby:
    version: 2.1.5
dependencies:
  pre:
    - wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.5_x86_64.deb
    - sudo dpkg -i vagrant_1.6.5_x86_64.deb
    - if [ ! -d /home/ubuntu/.vagrant.d/gems/gems/vagrant-digitalocean-0.7.1 ] ; then sudo vagrant plugin install vagrant-digitalocean ; fi
  cache_directories:
    - ".kitchen"
    - "~/.vagrant.d/"
test:
  override:
    - "ci/circle_build_parallel.sh":
        timeout: 1200
        parallel: true
    - "bundle exec rake sync":
        parallel: true
    - "ci/circle_release.sh":
        parallel: true
```


## パッケージ並行ビルド: circle_build_parallel.sh

ビルドの手口はTest-KitchenとVagrant-DigitalOceanです。
omnibusはポータビリティー面で良いのですが、初回のビルドにはすごく時間がかかります。

CircleCIはビルドを30分で終わらせないとコンテナが停止するので、rpmとdebを並行して作ることにしました。


Test-Kitchenのconvergeでそれぞれ準備をさせたら、それぞれのDroplet内で`omnibus build`をするため`kitchen exec`をかけました。

```
#!/usr/bin/env bash

set -e
case $CIRCLE_NODE_INDEX in
  0) TARGET="centos"
    ;;
  1) TARGET="ubuntu"
    ;;
esac

RUBYPATH="/opt/rubies/ruby-2.1.5/bin"

bundle exec kitchen destroy ${TARGET}
bundle exec kitchen converge ${TARGET}

bundle exec kitchen exec ${TARGET} -c "export PATH=${RUBYPATH}:\$PATH; gem install bundler --no-ri --no-rdoc"
bundle exec kitchen exec ${TARGET} -c "export PATH=${RUBYPATH}:\$PATH; cd /home/vagrant/serverspec; bundle install --jobs=2 --verbose --binstubs --without development"
bundle exec kitchen exec ${TARGET} -c "export PATH=${RUBYPATH}:\$PATH; cd /home/vagrant/serverspec; ./bin/omnibus build serverspec"
```

omnibusの2回目以降は変更点以外キャッシュをつかうのですぐ終わります。実際の所ビルドサーバは必要な時にスタート、終わったらストップにするのがよいです。
が、今回は毎回破棄コースで。


### .kitchen.yml

結局Dropletは`8GB`でないと、ビルドが安定して30分を切らなかった。
あとは普通の複数プラットフォーム用`.kitchen.yml`ですね。

```
driver:
  name: vagrant
  provider: digital_ocean
  box: digital_ocean
  box_url: https://github.com/smdahlen/vagrant-digitalocean/raw/master/box/digital_ocean.box
  vagrantfile_erb: <%= File.join(File.dirname(__FILE__), "templates/Vagrantfile.erb") %>
  digitalocean:
    provider:
      token: <%= ENV['DIGITALOCEAN_TOKEN'] %>
      ssh_key_name: <%= ENV['DO_SSH_KEY_NAME'] %>
      size: 8GB
      region: nyc3
    override:
      ssh.private_key_path: <%= ENV['DO_SSH_KEY'] %>
  synced_folders:
    - ['./', '/home/vagrant/serverspec']

provisioner:
  require_chef_omnibus: 11.16.4

platforms:
  - name: ubuntu-14.04
    run_list: apt::default
    driver:
      digitalocean:
        provider:
          image: ubuntu-14-04-x64
  - name: centos-6.5
    driver:
      digitalocean:
        provider:
          image: centos-6-5-x64

suites:
  - name: default
    run_list: omnibus::default
    attributes:
      omnibus:
        build_user:  vagrant
        build_dir:   /home/vagrant/serverspec
        install_dir: /opt/serverspec
        ruby_version: 2.1.5
```

## パッケージ回収: rakeタスク

omnibusは作成したパッケージをDroplet内に放置します。
Vagrant+VirtualBoxとかなら共有ディレクトリに指定すればOKですが、相手先によっては回収するかその場でリリースさせます。

今回はTest-Kitchenのインスタンスステータスを利用して、(pkgが作成されていることを信じて)RsyncでCircleCIのコンテナ側に回収しました。

```
require 'yaml'
require 'json'
require 'open-uri'

platforms = Dir.glob('.kitchen/*.yml')
@instances = []
platforms.each do |platform|
  @instances << YAML.load(File.read(platform))
end

...

desc "collect packages from remote server"
task :sync do
  # puts @instances
  remote_user = ENV['REMOTE_USER_NAME'] || 'root'
  @instances.each do |instance|
    system "rsync -avz --progress -e 'ssh -oStrictHostKeyChecking=no -C -i #{ENV['DO_SSH_KEY']}' #{remote_user}@#{instance['hostname']}:/home/vagrant/serverspec/pkg/ ./pkg"
  end
end

...
```

Dropletから回収さえできれば、とりあえずartifactsとして保管してもらえるので、リリースがこけてもまあなんとか。

![OpsRockin_omnibus-serverspec__22_-_CircleCI.png](https://qiita-image-store.s3.amazonaws.com/0/7454/9d32c10e-04d5-1c72-5138-f96ff26e3045.png "OpsRockin_omnibus-serverspec__22_-_CircleCI.png")


## Packagecloudにリリース: circle_release.sh

これも並行コンテナごと、`package_cloud`コマンドで普通にpushするだけ。

```circle_release.sh
#!/usr/bin/env bash

set -e
if [ "${CIRCLE_BRANCH}" == "master" ]; then
  case $CIRCLE_NODE_INDEX in
    0) TARGET="centos"
      bundle exec kitchen destroy ${TARGET}
      bundle exec package_cloud push omnibus-serverspec/dummy_with_ci/el/6 pkg/*.rpm
      ;;
    1) TARGET="ubuntu"
      bundle exec kitchen destroy ${TARGET}
      bundle exec package_cloud push omnibus-serverspec/dummy_with_ci/ubuntu/trusty pkg/*.deb
      ;;
  esac
elif [ "${CIRCLE_BRANCH}" == "release_package" ]; then
  case $CIRCLE_NODE_INDEX in
    0) TARGET="centos"
      bundle exec kitchen destroy ${TARGET}
      bundle exec package_cloud push omnibus-serverspec/serverspec/el/6 pkg/*.rpm
      ;;
    1) TARGET="ubuntu"
      bundle exec kitchen destroy ${TARGET}
      bundle exec package_cloud push omnibus-serverspec/serverspec/ubuntu/trusty pkg/*.deb
      ;;
  esac
fi
```

で、こんな感じで置かれます。

https://packagecloud.io/omnibus-serverspec/serverspec

![omnibus-serverspec_serverspec_-_Packages.png](https://qiita-image-store.s3.amazonaws.com/0/7454/f09a564c-9caf-0172-117e-1c99a941a628.png "omnibus-serverspec_serverspec_-_Packages.png")


## 残課題

- パッケージがちゃんと動くかのテストは自動でしてない。
    - `kitchen verify`で結合テストはかけなくもない。
- バージョン同じ、リビジョン違いに対応してない。(pushこける)
    - packagecloudから一覧をとるAPIが見当たらないので困っている。
- Serverspec本体のリリースと連動してない。
    - Rubygems Feed => IFTTT => CricleCIキックを画策。
    - ベータ版とかでたぶんこける。
    - Specinfraだけ更新されたときどうしよう。

そのうち困ったら更新しよう。
