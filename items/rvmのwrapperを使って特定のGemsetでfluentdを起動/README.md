<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

ほぼ本家[Using RVM with God](https://rvm.io/integration/god/) からの引用。



```bash:cli
rvm wrapper 1.9.3-p194@fluentd bootup fluentd 
```

とすると`/usr/local/rvm/bin/bootup_fluentd` というラッパーコマンドが完成する。

中身はこう。

```bash:/usr/local/rvm/bin/bootup_fluentd
#!/usr/bin/env bash

if [[ -s "/usr/local/rvm/environments/ruby-1.9.3-p194@fluentd" ]]
then
  source "/usr/local/rvm/environments/ruby-1.9.3-p194@fluentd"
  exec fluentd "$@"
else
  echo "ERROR: Missing RVM environment file: '/usr/local/rvm/environments/ruby-1.9.3-p194@fluentd'" >&2
  exit 1
fi
```

rvmのbin以下にもshbang用のアレコレあるけど、UPstart等から起動するにはこうしておくのがいい。

```
$ ruby
-bash: ruby: command not found
$ /usr/local/rvm/bin/bootup_fluentd --version
fluentd 0.10.24
```

このようにRuby．．．なにそれ？という状態からも一発稼働。
