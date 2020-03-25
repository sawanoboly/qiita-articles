
サーバのプロビジョニングをテストする[Test-Kitchen](https://github.com/test-kitchen/test-kitchen)が、v1.4でテストのステップ(verify)を追加しやすい変更をいれてきました。
そこで一本`kitchen-verifier-shell`を作りました。(RubyGemは本家に取り込まれるまでの限定公開です。)

## busserとServerspec、Infrataster

従来のTest-Kitchenのテストは[busser](https://github.com/test-kitchen/busser)というラッパを使って、テストスイートを対象のプラットホーム(VMなど)にインストールします。
馴染みのある例では[busser-serverspec](https://github.com/test-kitchen/busser-serverspec)などは、もはや公式のポジションですね。

ただ、テスト対象に直接インストールするというあたりで時間がかかったり、少々トラブルも発生します。
元々、対象の外からつついてテストしようというServerspecや、外からテストしてなんぼというInfratasterは、できるならセットアップ済みの環境からさくっと叩きたい(※特にCapybaraリソース)です。

busser自体、Ruby必須になってしまうのもちょっと。


## kitchen-verifier-shell

で、busserを介さないTest-Kitchenでのverifyステップがあってもいいなと。
本家Test-Kitehcnに[プルリクエストを送り](https://github.com/test-kitchen/test-kitchen/pull/741)つつ、[kitchen-verifier-shell](https://rubygems.org/gems/kitchen-verifier-shell)として公開しました。

> 追記: Shell-Verifierは、インスタンスにスクリプトを送らず、Test-Kitchenを実行しているワークステーション上のコマンドを実行します。
> インスタンスの中でスクリプトを実行したい場合、commandに`kitchen exec`を含むシェルを実行すればOKです。

> 追記: 本家にマージされました。次回リリースからつかえそう。

使うためのGemfile例はこちら。

```Gemfile
source "https://rubygems.org"

gem 'kitchen-verifier-shell'
gem 'kitchen-vagrant'
gem 'infrataster'
```


Verifierに標準のbusser以外の指定する場合、`.kitchen.yml`で`verifier`セクションを作成します。
軽く[Infrataster](https://github.com/ryotarai/infrataster)のVagrantfile例を踏襲した例がこちら。

```.kitchen.yml
---
driver:
  name: vagrant

provisioner:
  name: shell

platforms:
  - name: ubuntu-14.04
    driver:
      network:
        - ["private_network", {ip: "192.168.33.10"}]   ## このへんが踏襲
        # - ["forwarded_port", {guest: 80, host: 8080}]

verifier:
  name: shell
  command: ./bin/rspec -fd    ## Infrataster実施

suites:
  - name: default
    run_list:
    attributes:
```

プロビジョンに使う`bootstrap.sh`を、適当に。

```bootstrap.sh
#!/usr/bin/env bash

sudo apt-get update -y
sudo apt-get install nginx -y
```

これで`kitchen converge`まで済ませておくといいです。

```
$ time ./bin/kitchen converge -l warn

real	2m29.396s
user	0m6.406s
sys	0m1.849s
```

2分半ってとこですね。

## InfratasterでNginxのテストを作成

ジェネレータにひな形を作らせて、とりあえず`Welcome to nginx!`でも拾いましょうか。

```spec_helper.rb
require 'infrataster/rspec'

Infrataster::Server.define(:nginx) do |server|
  server.address = '192.168.33.10'
end


RSpec.configure do |config|
  config.treat_symbols_as_metadata_keys_with_true_values = true
  config.run_all_when_everything_filtered = true
  config.filter_run :focus

  config.order = 'random'
end
```

このへんもサンプルドキュメントをほぼ踏襲で。

```nginx_spec.rb
require 'spec_helper'

describe server(:nginx) do
  describe http('http://nginx') do
    it "responds OK 200" do
      expect(response.status).to eq(200)
    end

    it "responds content including 'Welcome to nginx!'" do
      expect(response.body).to include('Welcome to nginx!')
    end
  end
end
```


### kitchen verify with rspec

ではconverge後、初回のverifyをやってみましょう。

```
$ time ./bin/kitchen verify 
-----> Starting Kitchen (v1.4.1)
-----> Verifying <default-ubuntu-1404>...
       [Shell] Verify on instance=#<Kitchen::Instance:0x007faa63b2b9b0> with state={:hostname=>"127.0.0.1", :port=>"2222", :username=>"vagrant", :ssh_key=>"/xxxx/kitchen_shell_verify/.kitchen/kitchen-vagrant/kitchen-kitchen_shell_verify-default-ubuntu-1404/.vagrant/machines/default/virtualbox/private_key", :last_action=>"verify"}

(※ Warning類カット)

server 'nginx'
  http 'http://nginx' with {:params=>{}, :method=>:get, :headers=>{}}
    responds content including 'Welcome to nginx!'
    responds OK 200

Finished in 0.00887 seconds (files took 0.3514 seconds to load)
2 examples, 0 failures

Randomized with seed 56543

       Finished verifying <default-ubuntu-1404> (0m0.62s).
-----> Kitchen is finished. (0m0.84s)


real	0m1.253s
user	0m1.064s
sys	0m0.157s

```

1秒ちょいすか。


## 環境変数、Vagrant以外のドライバで

Shell-Verifierがコマンドを実行する際、インスタンス(テスト対象)のパラメータが環境変数で渡るようにしています。
この記事で紹介している使用例では次のような内容。

```
KITCHEN_HOSTNAME=127.0.0.1
KITCHEN_PORT=2222
KITCHEN_USERNAME=vagrant
KITCHEN_SSH_KEY=/xxxxxxxxx/.kitchen/kitchen-vagrant/kitchen-kitchen_shell_verify-default-ubuntu-1404/.vagrant/machines/default/virtualbox/private_key
KITCHEN_LAST_ACTION=setup
```

ServerspecのSSH接続情報としてこれら環境変数を使うことができますね。たとえばkitchen-ec2などのクラウド向けプラグインではPublicIPなどが渡ってくるのでそっちでも重宝します。


## おわりに

もちろんテスト対象の作成とテストを個別に実行、セットアップしてもかまわないんです。しかしTest-Kitchenでやると、`--destroy=always`などがつかえてCIに組み込みやすいのがいいんですよね。もちろん並列実行もOK。

ちなみに作成の動機、[ngx_mruby](https://github.com/matsumoto-r/ngx_mruby)で作ったルーティングをInfratasterでさくさくとテストしやすくしたかったのです。

さて、verifyステップが早くなったので、Vagrantドライバーではインスタンス作成に要する時間が目立ちます。
コーヒーのおかわりを汲みに行ってもいいんですが、ここでさらに[kitchen-docker_cli](https://github.com/marcy-terui/kitchen-docker_cli)などと併用すると、`kitchen test`が休憩を取る間もなく終わるようになりますね。

