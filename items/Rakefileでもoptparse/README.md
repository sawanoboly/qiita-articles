<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

optparseの使い方を知ったので、Rakeで使ってみた。
直接渡すのは多分ムリなので、ENVでもらって配列に。

```ruby:Rakefile
# -*- coding: utf-8 -*-

require 'optparse'

argv_array = ENV["ARGV"] ? (ENV["ARGV"]).split(" ") : []

options = {}
OptionParser.new do |opts|
  opts.banner = 'Usage: rake [rake_options] task  ARGV="[options]"'

  opts.on('-e', '--environment ENVIRONMENT') {|v| options[:rake_env] = v}
  opts.on('-p', '--port PORT_NUM') {|v| options[:port] = v.to_i}
end.parse!(argv_array)

desc "print parsed options."
task :print_opt do
  p options
end
```

パースしたいオプションは`ARGV=""`とまとめて放り込む仕様。

```bash:shell_output
$ rake print_opt ARGV="-h"
Usage: rake [rake_options] task  ARGV="[options]"
    -e, --environment ENVIRONMENT
    -p, --port PORT_NUM


$ rake print_opt ARGV="-e production -p 8875"
{:rake_env=>"production", :port=>8875}
```

取ってつけた感はあるけども、書式のチェックは楽になるかも。
