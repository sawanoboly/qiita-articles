

Chefのクライアントバージョン11から`chef-apply`とコマンドラインツールが追加されていた。

ヘルプを見るとレシピのDSLを渡して実行してくれるものらしい。

> 追記： Chef12.1.0からnode.json的なファイルを渡せるようになったよ。
>     -j JSON_ATTRIBS,                 Load attributes from a JSON file or URL
>        --json-attributes



```bash:Shell-Out
$ chef-apply --help
Usage: chef-apply [RECIPE_FILE] [-e RECIPE_TEXT] [-s]
        --[no-]color                 Use colored output, defaults to enabled
    -e, --execute RECIPE_TEXT        Execute resources supplied in a string
    -j JSON_ATTRIBS,                 Load attributes from a JSON file or URL
        --json-attributes
    -l, --log_level LEVEL            Set the log level (debug, info, warn, error, fatal)
    -s, --stdin                      Execute resources read from STDIN
    -v, --version                    Show chef version
    -W, --why-run                    Enable whyrun mode
    -h, --help                       Show this message
```


レシピDSLを渡してみた。単体のレシピファイルでも使えるので、solo用に`chef-repo`を用意するのが面倒、などの時に使うのかな。

```bash:Shell-Out
$ chef-apply -e 'service "cron" do action :enable end'         
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * service[cron] action enable (up to date)
```

ちなみにレシピ用DSLということで、普通にRubyも実行出来る。

```bash:Shell-Out
$ chef-apply -e '10.times { |x| puts [x.to_s, node.platform ].join("-") }'
0-smartos
1-smartos
2-smartos
3-smartos
4-smartos
5-smartos
6-smartos
7-smartos
8-smartos
9-smartos
```

Ohaiも効いているね。

----

追記：


シェルからattributesを使いたいとき、Ohaiを直接叩くより地味に便利？

```bash:Shell-Out
$ chef-apply -e 'puts  node[:ipaddress]'
210.152.xxx.xxx
```

```bash:Shell-Out
$ chef-apply -e 'node.keys.map {|x| puts x }'      
tags
languages
kernel
os
os_version
hostname
fqdn
domain
virtualization
etc
current_user
chef_packages
network
counters
ipaddress
macaddress
ip6address
ohai_time
keys
platform_version
platform_build
platform
platform_family
dmi
command
zpools
uptime_seconds
uptime
filesystem
recipes
roles
```
