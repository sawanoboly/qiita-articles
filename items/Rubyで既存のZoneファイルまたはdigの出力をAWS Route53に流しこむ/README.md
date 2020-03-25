<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

bindで管理しているDNSのゾーンをAWSのRoute53に移行したくなったので作成。
ゾーンファイルを読み込んでRoute53上にレコードを登録します。

ついでに御存知の通り`dig`コマンドの出力はゾーンファイルにそのまま使えるので、digの結果パイプでスクリプトに渡せばroute53に適当に登録する実験も。


```ruby:to_route53.rb
require 'zonefile'
require 'route53'

my_access_key = 'REPLACE_TO_YOUR_KEY'
my_secret_key = 'REPLACE_TO_YOUR_SECRET'

domain = 'example.com.'
zone_id = '/hostedzone/ZONEID'

default_ttl = '600'
conn = Route53::Connection.new(my_access_key,my_secret_key,'2011-05-05' , 'https://route53.amazonaws.com/',false)
## you may need to use this blanch due to ssl verify error.
## https://github.com/higanworks/ruby_route_53/tree/add_option_ssl_no_verify
# conn = Route53::Connection.new(my_access_key,my_secret_key,'2011-05-05' , 'https://route53.amazonaws.com/',false , true)

mng_zone = Route53::Zone.new(domain,zone_id,conn)

zonefile = Zonefile.from_file('./example.com.zone')
# zonefile = Zonefile.new($stdin.read)

def build_namelist(records)
  records.map{|obj| obj[:name]}.uniq
end

def build_hostlist(name, records)
  hosts = []
  records.each do |rec|
    hosts << rec[:host] if name == rec[:name]
  end
  hosts
end

def build_mx_hostlist(records)
  records.map{|mx| [mx[:pri].to_s, mx[:host]].join(' ')}
end

def prepare_ttl(name, records)
  records.select{|x| x[:name] == records}.first.tap do |x|
    break x[:ttl] if x
  end
end

%w(a cname txt mx).each do |w|
  namelist = build_namelist(zonefile.records[w.to_sym])

  namelist.each do |name|
    hosts = build_hostlist(name, zonefile.records[w.to_sym])
    ttl = prepare_ttl(name, zonefile.records[w.to_sym]) || default_ttl
    case w
    when 'a'
      if  name == '@'
        new_name = domain
      elsif name.end_with?('.')
        new_name = name
      else
        new_name = [name ,domain].join('.')
      end
      new_record = Route53::DNSRecord.new(new_name, w.to_s.upcase , ttl.to_s, hosts ,mng_zone)
    when 'cname'
      if name.end_with?('.')
        new_name = name
      else
        hosts.map!{|h| [h,domain].join('.')}
        new_name = [name ,domain].join('.')
      end
      new_record = Route53::DNSRecord.new(new_name, w.to_s.upcase , ttl.to_s, hosts ,mng_zone)
    when 'txt'
      zonefile.records[w.to_sym].each do |record|
        new_record = Route53::DNSRecord.new(domain, w.to_s.upcase , record[:ttl] || default_ttl, [record[:text]] ,mng_zone)
    puts new_record
        new_record.create
      end
      next
    when 'mx'
      hosts = build_mx_hostlist(zonefile.records[w.to_sym])
      new_record = Route53::DNSRecord.new(name, w.to_s.upcase , ttl.to_s, hosts ,mng_zone)
    else
      puts "Unsupport resource type detected. -> #{w.to_s.upcase}"
    end

    new_record.create
  end
end
```


## 使い方

### 共通準備

先にAWS Route53で移行対象のドメインをHosted Zonesに追加しておきます。
次に下記の箇所を各自のものに差し替えます。

- my_access_key = 'REPLACE_TO_YOUR_KEY'
- my_secret_key = 'REPLACE_TO_YOUR_SECRET'
- domain = 'example.com.'
- zone_id = '/hostedzone/ZONEID'


### Zoneファイルから

- zonefile = Zonefile.from_file('./example.com.zone')

ここを自前のゾーンファイルに差し替えて実行すればOK。

`ruby ./to_route53.rb`

### digの出力から

zonefileを`$stdin`から作成するように切り替えます。

```ruby
# zonefile = Zonefile.from_file('./example.com.zone')
zonefile = Zonefile.new($stdin.read)
```

で、パイプで渡します、必要なレコードをdigりましょう。

`dig hoge.example.com @8.8.8.8 | ruby ./to_route53.rb`

ローカルキャッシュを使われるとCNAMEがAに差し替わることがあるので、他所に聞きましょう。例ではGoogleの Public DNSを問い合わせ先に指定しています。

```shell
$ dig hoge.example.com @8.8.8.8 | bundle exec ruby ./to_route53.rb
hoge.l.example.com. A 600 192.168.15.73,192.168.15.75,192.168.15.71
hoge.example.com. CNAME 600 hoge.l.example.com
```

こんなかんじでクエリ結果をroute53にもりもり登録出来ます。

## 後処理

レジストリのNSを`awsdns`に向けたら移行完了です、念のためレコードの確認はしておきましょう。

## 追記：SSLベリファイエラーが起こるとき

環境によっては`ruby_route53`の実行時にSSLベリファイエラーでコケることがあります。
その時はこちらのブランチを使います。

`https://github.com/higanworks/ruby_route_53/tree/add_option_ssl_no_verify`

で、`Route53::Connection.new`の引数に最後一つ`true`を追加すれば実行できます。

```
# conn = Route53::Connection.new(my_access_key,my_secret_key,'2011-05-05' , 'https://route53.amazonaws.com/',false)
conn = Route53::Connection.new(my_access_key,my_secret_key,'2011-05-05' , 'https://route53.amazonaws.com/',false , true)
```
