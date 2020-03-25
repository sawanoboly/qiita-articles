<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

serverspecで状態をテストした後、Webのアプリなら振る舞いもテストしようと思いました。

で、serverspecがRSpecならば、[Capybara](https://github.com/jnicklas/capybara "jnicklas/capybara")も混ぜたらいいんじゃね？ と試してみた。

## コード

https://github.com/higanworks/serverspec_with_capybara-example


`spec_helper`はこんな感じで。

アプリのコードを読むわけではないので、webkitドライバでリモート扱いとしてテストすることにしました。


```ruby:spec_helper.rb
require 'serverspec'
require 'capybara/rspec'
require 'capybara-webkit'

include SpecInfra::Helper::Exec
include SpecInfra::Helper::DetectOS

RSpec.configure do |c|
  if ENV['ASK_SUDO_PASSWORD']
    require 'highline/import'
    c.sudo_password = ask("Enter sudo password: ") { |q| q.echo = false }
  else
    c.sudo_password = ENV['SUDO_PASSWORD']
  end
end

Capybara.default_driver = :webkit
```


で、serverspecとcapybaraのテストを混ぜたspecファイルを書いた。

- (serverspec) `openssl version` の戻りをチェック
- (Capybara練習用) [Test your server for Heartbleed (CVE-2014-0160)](https://filippo.io/Heartbleed/ "Test your server for Heartbleed (CVE-2014-0160)") のサイトに到達性チェック
- (Capybara練習用) とばっちり的に`google.com:443`のHeartbleedをテストして、`All good〜`の出力を得る

接続先は`Capybara.app_host`で自由に調整できるし、サイト上でのクライアント挙動も端的に記述できていい感じです。
実際にやる場合はサーバにデプロイしたアプリや、自前で運用しているWEBのエントリポイントに対して行いましょう。


```ruby:heart_spec.rb
# coding: utf-8
require 'spec_helper'

## serverspecのコマンドリソースでopensslのバージョンをチェックする
describe command('openssl version') do
  it { should return_stdout /^OpenSSL\ 1\.0\.1g/ }
end


## `:type => :feature`で`capybara`を使う
describe 'visit Heartbleed test' ,:type => :feature do

  before :each do
    Capybara.app_host = 'https://filippo.io'
  end

  ## Heartbleedテストのトップページを表示するテスト
  it 'show testpage' do
    visit '/Heartbleed/'
    expect(page).to have_content 'Heartbleed test'
  end

  ## google.com(任意)に対してHeartbleedテストを実施して、`All good`が表示されることを確認
  it 'check google ssl' do
    visit '/Heartbleed/'
    fill_in 'hostname', :with => 'google.com'
    click_on 'Go!'
    expect(page).to have_content 'All good, google.com seems fixed or unaffected!'
  end
end
```

どちらもRSpecなので、統一感がありますね。



## テスト実行

では実行してみましょう。

```rspec:rspec_output
$ rspec -fd --color

Command "openssl version"
  should return stdout /^OpenSSL\ 1\.0\.1g/

visit Heartbleed test
  show testpage
  check google ssl

Finished in 4.79 seconds
3 examples, 0 failures
```


いけました。

## 追記:Sshバックエンドなときー

describe内のbeforeではなく、テンプレートから生成されたspec_helperに入れちゃう手があります。

```
# -- snip --
    if c.host != host
      c.ssh.close if c.ssh
      c.host  = host
## ここでCapybara.app_hostを設定しちゃう。
      Capybara.app_host = "http://#{host}"
      options = Net::SSH::Config.for(c.host)
      user    = options[:user] || Etc.getlogin
      c.ssh   = Net::SSH.start(host, user, options)
    end
# -- snip --
```

これだとRSpec実行端末から到達性が必要なので、いっそSocksやポートフォワード張らせてもいいのかもしれない。
