<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


先日リリースされたAWS OpsWorksでもELBが使える機能ですが、利用するためには事前に設定の調整が必要です※。

※新規利用なら必要ないかもしれません。


## IAM設定を促される

いざレイヤにELBを構成しようとすると『パーミッションが無いからELBがリストアップできない』と言われてしまいます。

解説はこちらにありました。

[http://docs.aws.amazon.com/opsworks/latest/userguide/opsworks-security-servicerole.html](http://docs.aws.amazon.com/opsworks/latest/userguide/opsworks-security-servicerole.html)

## Q. 結局どうすればよいの？

**A1.** Stack作成時に新規IAMロールの作成を選びましょう
**A2.** Stackで使われるデフォルトのロール、`Role: aws-opsworks-ec2-role`に下記のポリシーを適用しましょう。

![Role for OpsWorks](http://farm8.staticflickr.com/7424/8799140443_00fd60b062_z.jpg)


これでレイヤにELBを関連付けることができ、配下のインスタンスが自動的にAttach＆Removeされるようになります。
※OpsWorksで利用できるようになるまで若干のタイムラグがあります。
