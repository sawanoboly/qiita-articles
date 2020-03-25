
HHVMは3.11のリリースから、PHP7モードを備えるようになったそうな。

[PHP 7 Support « HHVM](http://hhvm.com/blog/10859/php-7-support)

試してみるぞ。

## PHP7モード設定

設定項目の案内はこちら。

[Configuration: INI Settings #php-7-settings](https://docs.hhvm.com/hhvm/configuration/INI-settings#php-7-settings)

とりあえず有効にしてみるなら、`hhvm.php7.all = 1` で良いっしょ。


## WordPressとAmazon Linuxで試してみる

HHVMは毎度自分でビルドして、AmazonLinux用のRPMを作っている。 ビルドの内訳はこちら。 => [OpsRockin/hhvm-for-amazon-linux](https://github.com/OpsRockin/hhvm-for-amazon-linux)


これをインストール済AMIがあるので、起動する。

[WordPress Powered by AMIMOTO (HHVM)](https://aws.amazon.com/marketplace/pp/B00V5JYXTO)


```shell:
      ___         _            __
     / _ | __ _  (_)_ _  ___  / /____
    / __ |/  ' \/ /  ' \/ _ \/ __/ _ \
   /_/ |_/_/_/_/_/_/_/_/\___/\__/\___/

https://aws.amazon.com/amazon-linux-ami/2015.09-release-notes/

 Nginx 1.9.12, HHVM 3.12.1, Percona MySQL 5.6.29, WP-CLI 0.22.0

 amimoto     http://www.amimoto-ami.com/
 digitalcube https://en.digitalcube.jp/
```

うむ、`3.12.1`が入ってる。PHP7モードを有効にするため、`/etc/hhvm/server.ini`を編集。

```/etc/hhvm/server.ini
...

hhvm.php7.all = true

...
```

コンパイルされたバイナリを消しつつ、リスタートしよう。

```
$ sudo service hhvm stop
$ sudo rm -f /tmp/.hhvm.hhbc 
$ sudo service hhvm start
```

このあとシステムの情報をチェックすれば、PHPバージョンが`7.0.99-hhvm`になっている。


```
## Server Environment ##

Server Info:              nginx/1.9.12
Host:                     DBH: localhost, SRV: _
Default Timezone:         UTC
MySQL Version:            5.6.29-76.2-log

-- PHP Configuration

PHP Version:              7.0.99-hhvm
PHP Post Max Size:        
PHP Time Limit:           0
PHP Max Input Vars:       
PHP Safe Mode:            No
PHP Memory Limit:         256M
PHP Upload Max Size:      
PHP Upload Max Filesize:  
PHP Arg Separator:        &
PHP Allow URL File Open:  Yes
```


## WordPress的に注意するとこ

基本的には動くんだけど、気になる点があったのでメモ。

- /var/log/hhvm/error.log に、わんさかと型のWarningがでる。
    - ちょっと厳密にチェックするようにしたせいかしらね。
    - ex: `Warning: version_compare() expects parameter 1 to be string,`
    - ex: `Warning: setcookie() expects parameter 5 to be string,`
- 管理画面から新規プラグインが探せない（ｗ
    - 多分PHPバージョンが`7.0.99-hhvm`を返すからかな？下記画像のように止まる。

![Add_Plugins_‹_HHVM_mode_php7_—_WordPress.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/ef0702ae-c169-a05e-5887-1bf581003d63.jpeg "Add_Plugins_‹_HHVM_mode_php7_—_WordPress.jpg")


このあたりはそのうち調整されそう。
