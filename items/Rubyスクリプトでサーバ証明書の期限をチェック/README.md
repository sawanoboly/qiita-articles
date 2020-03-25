<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

SSLコネクション用のサーバ証明書の期限をチェックして、OKなら0,NGなら2で終了するNagiosプラグイン的なスクリプト。


```ruby:check_cert_period.rb
#!/usr/bin/env ruby
# -*- coding:utf-8 -*-
require 'socket'
require 'openssl'
require 'timeout'
include OpenSSL

timeout=15

r_host = ARGV[0] || "google.com"
r_date = ARGV[1] || 30
r_port = ARGV[2] || 443
r_date_s = r_date.to_i * 60 * 60 * 24

# set SSL config
ssl_conf = SSL::SSLContext.new()
# ssl_conf.verify_mode=SSL::VERIFY_PEER

# create ssl connection.
begin
  timeout(timeout) {
    @soc = TCPSocket.new(r_host.to_s, r_port.to_i)
    @ssl = SSL::SSLSocket.new(@soc, ssl_conf)
    @ssl.connect
  }

rescue Timeout::Error => e
  puts "CRITICAL - #{e.class} couldn't connext to #{r_host.to_s}:#{r_port.to_i}"
  exit 2
rescue => e
  puts "CRITICAL - #{e.class} #{e.message}"
  exit 2
end

# check period.
if (@ssl.peer_cert.not_after - Time.now) < r_date_s
  puts "CRITICAL - Certificate expired on #{@ssl.peer_cert.not_after}"
  exit 2
else
  puts "OK - Certificate will expire on #{@ssl.peer_cert.not_after}"
end
  
@ssl.close
@soc.close
```


## 実行結果

第2引数に期限、3つ目にポートをオプションで。

    $ ./check_cert_period.rb google.com 30 
    OK - Certificate will expire on 2013-06-07 19:43:27 UTC
    $ ./check_cert_period.rb google.com 360
    CRITICAL - Certificate expired on 2013-06-07 19:43:27 UTC
    $ echo $?
    2
