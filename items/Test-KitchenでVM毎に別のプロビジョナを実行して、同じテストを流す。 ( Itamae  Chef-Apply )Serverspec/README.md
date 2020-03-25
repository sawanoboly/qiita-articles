

前回、 [Test-Kitchenの対応プロビジョナを増やした Itamae, chef-apply - Qiita](http://qiita.com/sawanoboly/items/7cf323e65a4757321553 "Ruby - Test-Kitchenの対応プロビジョナを増やした Itamae, chef-apply - Qiita")ということをやりました。

作った後、ついやりたくなったのが次の通り。

1. Itamae, Chef-Applyで同じレシピを流す
2. それぞれにServerspec

やってみた。 => [各ファイルはここにも置いた。(kitchen-itamae_playground/itamae_or_chef_and_serverspec)](https://github.com/OpsRockin/kitchen-itamae_playground/tree/master/itamae_or_chef_and_serverspec)

## Test-Kitchenの設定とレシピとテスト

VM毎に別のプロビジョナを使う場合は、`platforms`の下に`provisioner`をはやすといい。(`driver`なら個別のVMプロバイダを使うことも可。)

```yaml:.kktchen.yml
---
driver:
  name: vagrant
  box: opscode-ubuntu-14.04

platforms:
  - name: vm-chef   # このVMはChef-Apply
    provisioner:
      name: chef_apply
      apply_path: kitchen
  - name: vm-itamae # このVMはItamae
    provisioner:
      name: itamae

suites:
  - name: default
    run_list:
      - package
      - file
    attributes:
      message: EHLO world.example.com
```


### Recipes

ではItamaeとChef-Applyで互換性のあるレシピを用意。

```ruby:package.rb+file.rb
package 'tmux' do
  action :install
end

file '/tmp/date.txt' do
  content Time.now.to_s
end

# care for chef-apply
node.normal.merge!(
  JSON.parse(
    File.read(
      File.expand_path("../../dna.json", __FILE__)
    )
  )
) if self.class.to_s == 'Chef::Recipe'

file '/tmp/message.txt' do
  content node[:message]
end
```

Chef-ApplyがJSONファイルを受け取るようになっていないので、ちょっと介護が必要だった。

> 追記： Chefの12.1.0で chef-applyにも`-j JSON_ATTRIBS`オプションが追加されています。

### Tests

次はテスト、普通。

```ruby:package_spec+file_spec
require 'spec_helper'

describe package('tmux') do
  it { should be_installed }
end

describe file('/tmp/date.txt') do
  it { should be_file }
end

describe file('/tmp/message.txt') do
  it { should be_file }
  its(:content) { should match /EHLO/ }
end
```


## Result

`kitchen test`してみた。

Chef-Applyの出力。

```shell:vm-chef
-----> Testing <default-vm-chef>
...

       Recipe: (chef-apply cookbook)::(chef-apply recipe)
         * apt_package[tmux] action install
           - install version 1.8-5 of package tmux
       Recipe: (chef-apply cookbook)::(chef-apply recipe)
         * file[/tmp/date.txt] action create
           - create new file /tmp/date.txt
           - update content in file /tmp/date.txt from none to 01bf66
           --- /tmp/date.txt	2015-03-02 09:19:11.671843579 +0000
           +++ /tmp/.date.txt20150302-1864-3n708w	2015-03-02 09:19:11.671843579 +0000
           @@ -1 +1,2 @@
           +2015-03-02 09:19:11 +0000
         * file[/tmp/message.txt] action create
           - create new file /tmp/message.txt
           - update content in file /tmp/message.txt from none to 30d48b
           --- /tmp/message.txt	2015-03-02 09:19:11.679843579 +0000
           +++ /tmp/.message.txt20150302-1864-1ucvcz8	2015-03-02 09:19:11.679843579 +0000
           @@ -1 +1,2 @@
           +EHLO world.example.com
       Finished converging <default-vm-chef> (1m11.48s).
```

Itamaeの出力。

```shell:vm-itamae
-----> Testing <default-vm-itamae>
...

        INFO : Starting Itamae...
        INFO : Loading node data from /tmp/kitchen/dna.json...
        INFO : Recipe: /tmp/kitchen/package.rb
        INFO :    package[tmux]
        INFO :       action: install
        INFO :          installed will change from 'false' to 'true'
        INFO : Starting Itamae...
        INFO : Loading node data from /tmp/kitchen/dna.json...
        INFO : Recipe: /tmp/kitchen/file.rb
        INFO :    file[/tmp/date.txt]
        INFO :       action: create
        INFO :          exist will change from 'false' to 'true'
        INFO :    file[/tmp/message.txt]
        INFO :       action: create
        INFO :          exist will change from 'false' to 'true'
zlib(finalizer): the stream was freed prematurely.
       Finished converging <default-vm-itamae> (2m12.19s).
```


Serverspecは両方こうなった。

```shell:Serverspec
       File "/tmp/date.txt"
         should be file
       
       File "/tmp/message.txt"
         should be file
         content
           should match /EHLO/
       
       Package "tmux"
         should be installed
       
       Finished in 0.1372 seconds (files took 0.32675 seconds to load)
       4 examples, 0 failures
```


