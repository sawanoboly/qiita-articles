<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

書き換えの対象はJoyent SmartOSでの固有情報を収集するために書いたプラグイン。Ohai本家にプルリクエストして、長い間放置していたもの。

コミットはこのツリー。

https://github.com/ZCloud-Firstserver/ohai/tree/b55bbc4a887e01958d8ffc064011e49982fb1966


```ruby:6_ohai/lib/ohai/plugins/joyent.rb
provides "joyent"
require_plugin "os"
require_plugin "platform"

if platform == "smartos" then
  joyent Mash.new

  # get uuid
  status, stdout, stderr = run_command(:no_status_check => true, :command => "/usr/bin/zonename")
  joyent[:sm_uuid] = stdout.chomp

  # get zone id unless globalzone
  status, stdout, stderr = run_command(:no_status_check => true, :command => "/usr/sbin/zoneadm list -p | awk -F: '{ print $1 }'")
  joyent[:sm_id] = stdout.chomp unless joyent[:sm_uuid] == "global"
  
  # retrieve image name and pkgsrc
  if ::File.exist?("/etc/product") then
    ::File.open("/etc/product") do |file|
      while line = file.gets
        case line
        when /^Image/
          sm_image = line.split(" ")
          joyent[:sm_image_id] = sm_image[1]
          joyent[:sm_image_ver] = sm_image[2]
        when /^Base Image/
          sm_baseimage = line.split(" ")
          joyent[:sm_baseimage_id] = sm_baseimage[2]
          joyent[:sm_baseimage_ver] = sm_baseimage[3]
        end
      end
    end

    ## retrieve pkgsrc
    sm_pkgsrc = ::File.read("/opt/local/etc/pkg_install.conf").split("=")
    joyent[:sm_pkgsrc] = sm_pkgsrc[1].chomp
  end
end
```

チケット[OHAI-458](https://tickets.opscode.com/browse/OHAI-458)で色々指摘があったので、それも込みで上記のやつをそのままOhai7の書式に書きなおしたのがこちら。

```ruby:7_ohai/lib/ohai/plugins/joyent.rb
Ohai.plugin(:Joyent) do
  %w{ sm_uuid sm_id sm_image sm_image_id sm_image_ver sm_baseimage sm_baseimage_id sm_baseimage_ver sm_pkgsrc }.each do |info|
    provides "joyent/#{info}"
  end
  provides 'virtualization/guest_id'
  depends 'os', 'platform', 'virtualization'


  def collect_solaris_guestid
    command = '/usr/sbin/zoneadm list -p'
    so = shell_out(command)
    so.stdout.split(':').first
  end

  def collect_product_file
    lines = []
    if ::File.exists?("/etc/product")
      ::File.open("/etc/product") do |file|
        while line = file.gets
          lines << line
        end
      end
    end
    lines
  end

  def collect_pkgsrc
    if File.exist?('/opt/local/etc/pkg_install.conf')
      sm_pkgsrc = ::File.read("/opt/local/etc/pkg_install.conf").split("=")
      sm_pkgsrc[1].chomp
    else
      nil
    end
  end

  def is_smartos?
    platform == 'smartos'
  end

  collect_data do
    if is_smartos?
      joyent Mash.new

      # copy uuid
      joyent[:sm_uuid] = virtualization[:guest_uuid]

      # get zone id unless globalzone
      unless joyent[:sm_uuid] == "global"
        virtualization[:guest_id] = collect_solaris_guestid
        joyent[:sm_id] = virtualization[:guest_id]
      end

      # retrieve image name and pkgsrc
      collect_product_file.each do |line|
        case line
        when /^Image/
          sm_image = line.split(" ")
          joyent[:sm_image_id] = sm_image[1]
          joyent[:sm_image_ver] = sm_image[2]
        when /^Base Image/
          sm_baseimage = line.split(" ")
          joyent[:sm_baseimage_id] = sm_baseimage[2]
          joyent[:sm_baseimage_ver] = sm_baseimage[3]
        end
      end

      ## retrieve pkgsrc
      joyent[:sm_pkgsrc] = collect_pkgsrc if collect_pkgsrc
    end
  end
end
```

同じようでところどころ違いますね。カタマリ単位で再分配した感じ。


## ピックアップ解説

### `require_plugin`は`depends`になった

これは見たままですね。
事前に依存をある程度解決するようになったのかな。


### `Ohai.plugin(:Joyent)`のブロックで囲んだ

Ohai7で作られたしきたり。とりあえず囲もう。


### データ収集を`collect_data`ブロックで囲んだ

`def`と明確に分けるのが目的っぽい。 
`Ohai.plugin`ブロック直下の`def`(メソッド定義)は、`collect_data`の中でしか呼べなかった。

例えば`if is_smartos?`は2行上では`No Methodエラー`で動かないんですね。

その他、他のプラグインで収集済みのattributeも、`def`内または`collect_data`内でのみ使えるようです。
`Ohai.plugin`直下で使えないのは、誤爆を防いだりするためでしょうかね。


### 環境依存がある所をメソッドにした

要はファイルや、コマンドラインで結果を取得するものをメソッドにしています。
これは7だからってわけではないのですが、テストの書きやすさに影響しています。この件は次の段落もどうぞ。


## 本家にPRするため、RSpec

Ohai6で見られた`@ohai`を使ったテストが姿を潜め、軒並み`@plugin`を使う形式に様変わりしています。

defを別にしたのはここでスタブしやすくするためでもあるようです。
結構ラクにモックを使ったテストができるようになりました。


```ruby:ohai/spec/unit/plugins/joyent_spec.rb
require 'spec_helper'


describe Ohai::System, "plugin joyent" do
  before(:each) do
    @plugin = get_plugin('joyent')
  end

  describe "without joyent" do
    before(:each) do
      @plugin.stub(:is_smartos?).and_return(false)
    end

    it "should NOT create joyent" do
      @plugin.run
      @plugin[:joyent].should be_nil
    end
  end

  describe "with joyent" do
    before(:each) do
      @plugin.stub(:is_smartos?).and_return(true)
      @plugin[:virtualization] = Mash.new
      @plugin[:virtualization][:guest_uuid] = "global"
    end

    it "should create joyent" do
      @plugin.run
      @plugin[:joyent].should_not be_nil
    end

    describe "under global zone" do
      before(:each) do
        @plugin.run
      end

      it "should ditect global zone" do
        @plugin[:joyent][:sm_uuid].should eql 'global'
      end

      it "should NOT create sm_id" do
        @plugin[:joyent][:sm_id].should be_nil
      end
    end

    describe "under smartmachine" do
      before(:each) do
        @plugin[:virtualization][:guest_uuid] = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx'
        @plugin.stub(:collect_solaris_guestid).and_return('30')
        @plugin.stub(:collect_product_file).and_return(["Name: Joyent Instance", "Image: base64 13.4.2", "Documentation: http://wiki.joyent.com/jpc2/SmartMachine+Base"])
        @plugin.stub(:collect_pkgsrc).and_return('http://pkgsrc.joyent.com/packages/SmartOS/2013Q4/x86_64/All')
        @plugin.run
      end

      it "should retrive zone uuid" do
        @plugin[:joyent][:sm_uuid].should eql 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx'
      end

      it "should collect sm_id" do
        @plugin[:joyent][:sm_id].should eql '30'
      end

      it "should collect images" do
        @plugin[:joyent][:sm_image].should_not nil
        @plugin[:joyent][:sm_image_id].should_not nil
        @plugin[:joyent][:sm_image_ver].should_not nil
      end

      it "should collect pkgsrc" do
        @plugin[:joyent][:sm_pkgsrc].should eql 'http://pkgsrc.joyent.com/packages/SmartOS/2013Q4/x86_64/All'
      end
    end
  end
end
```

テストもOK。

```shell:run_joyent_spec.output
Ohai::System plugin joyent
  without joyent
    should NOT create joyent
  with joyent
    should create joyent
    under global zone
      should ditect global zone
      should NOT create sm_id
    under smartmachine
      should retrive zone uuid
      should collect sm_id
      should collect images
      should collect pkgsrc

Finished in 0.01159 seconds
8 examples, 0 failures
```

## おわりに

だいたいこの辺を抑えておけば、7書式への移行もスムーズに行くんじゃないかと。
