<!-- permanent -->

`@reboot`とかのアレ。Chef-Clientの`11.12`以降(多分)で実装されています。

[Cron#Predefined_scheduling_definitions | Wikipedia](http://en.wikipedia.org/wiki/Cron#Predefined_scheduling_definitions)


## レシピサンプル

リブート時を指定してジョブを登録するには`time` Resource Attributeを使ってこうする。

```ruby:cron.rb
cron 'at_reboot' do
  command '/bin/true'
  time :reboot
end
```

## chef-applyで試してみる

```shell:chef-apply_stdout
$ chef-client -v
Chef: 11.12.4


$ sudo chef-apply cron.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * cron[at_reboot] action create
    - add crontab entry for cron[at_reboot]
```

エントリーを確認すると、`@reboot`なジョブが。

```
$ sudo crontab -l -u root
# Chef Name: at_reboot
@reboot /bin/true
```

指定できるのは次の通り。

```
SPECIAL_TIME_VALUES = [:reboot, :yearly, :annually, :monthly, :weekly, :daily, :midnight, :hourly]
```

## 消せない件

Amazon linux+omunibus installのchef-clientで試してるんですが、`action :delete`でジョブがちゃんと消えない。

```
$ sudo chef-apply cron.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * cron[at_reboot] action delete
    - save unmodified crontab

## ?? unmodified ??

$ sudo crontab -l -u root
@reboot /bin/true
```

マークだけ消えた。改行の扱い周りだと思うので、原因調べて治したいとこです。
