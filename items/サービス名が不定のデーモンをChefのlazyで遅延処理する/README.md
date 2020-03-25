<!-- permanent -->

先日JoyentのSmartOSで、困ったパッケージの配布をやられました。

- パッケージ名が一緒
- バージョンが一緒
- でもsvc上のサービスの名前が違うので、管理用に実行するコマンドが異なる

linux系で例えると、`service restart hogehoge` のhogehogeに指定する文字だけが違い、普通のパッケージ情報では分からんという状況です。


## lazyのサンプル

対象のサービスを、リソース収束の直前で、サーバの現状から調べてこなければならないと言うケースでは`lazy`による遅延処理が使用できます。

```Ruby:recipe.rb
##-本来はLibrariesに書く
module ::Helper
  def self.get_cron_service_name
    ## 本当は泥臭い判別処理を記述して、使用できる文字列を返す
    'cron'
  end
end


## サービス名を実行時に取得するリソース定義
service 'test' do
  service_name lazy { ::Helper.get_cron_service_name }
  # subscribes :restart, 'file[/tmp/hoge]', :immediately ## subscribesでもOK
end

file '/tmp/hoge' do
  content Time.now.to_s
  notifies :restart, 'service[test]', :immediately 
end
```

これで、リソースの名前は`service[test]`ですが、実際に発行されるコマンドではヘルパーが返してくれた文字列(この場合は`cron`)を使用します。


大体のケースでは`lazy`でなくとも大丈夫ですが、初回の実行などでパッケージのインストールが済んでない場合に、`lazy`しないとサービス名がわからないのですよね。


## chef-applyでお試し

```shell:chef-apply
# chef-apply test.rb 
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * service[test] action nothing (up to date)
  * file[/tmp/hoge] action create
    - update content in file /tmp/hoge from 2b795c to 04db98
        --- /tmp/hoge   2014-04-15 03:13:48.000000000 +0000
        +++ /tmp/.hoge20140415-4288-ulgn9f      2014-04-15 03:15:43.000000000 +0000
        @@ -1,2 +1,2 @@
        -2014-04-15 03:13:48 +0000
        +2014-04-15 03:15:43 +0000

  * service[test] action restart
    - restart service service[test]

```

予定通りCronがリスタートしました。

```
cron[4831]: (CRON) INFO (pidfile fd = 3)
cron[4832]: (CRON) STARTUP (fork ok)
```


## 追記

`lazy`でやると楽な事象は、なかなかのレアケースだと思います。
