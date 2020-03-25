

よく見る`/path/to/sendmail -i -t`が機能するためには、SMTPをどこかで受ける必要があります。そのためだけにローカルでMTAを起動したり、何かしらのフォワーダを動かしたりもします。

今回は`sendmail`への入力に対して、どうせテキストなんだしSMTPにのせる前に横取りして好きなメッセンジャーに流してしまいます。

サンプルコードのリポジトリはこちら。 https://github.com/sawanoboly/replace_sendmail_to_slackwebhook_example


## sendmailコマンドでメールを送信している例： crond

crondはMAILTOオプションをセットしておくと、stdout/stderrに何かしら出力があった場合にメールを送信します。
crontabに次のように書いたとします。

```
MAILTO="sawanoboly"
* * * * * /bin/not_found
```

このコマンドはエラーで終わります。その後、crondの内部ではsendmailコマンドに次のような標準入力をわたします。(※debugレベルをかなり詳細に設定すればその他引数も確認できるでしょう。)


```
To: "sawanoboly"
Subject: cron: /bin/not_found

/bin/sh: /bin/not_found: not found
```

ヘッダ＋空白行＋本文 の形式ですね。


## slackに流すコード(Go)

`net/mail`でパースできる形式なのでそのまま使い、slackへ。引数はまあ、、無視でいいや。

- 参考:
  - [mail - The Go Programming Language](https://golang.org/pkg/net/mail/)
  - [ashwanthkumar/slack-go-webhook: Go Library to send messages to Slack via Webhooks](https://github.com/ashwanthkumar/slack-go-webhook)


```golang:main.go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/mail"
	"os"
	"strings"

	"github.com/ashwanthkumar/slack-go-webhook"
)

var webhookURL = "https://hooks.slack.com/services/foo/bar/baz"

func main() {
	sendmainInput, _ := ioutil.ReadAll(os.Stdin)
	// net/mailのサンプルをほぼそのまま
	m, err := mail.ReadMessage(strings.NewReader(string(sendmainInput)))

	if err != nil {
		log.Fatal(err)
	}
	header := m.Header
	body, err := ioutil.ReadAll(m.Body)
	if err != nil {
		log.Fatal(err)
	}

	// slack-go-webhookのサンプルをほぼそのまま
	attachment1 := slack.Attachment{}
	attachment1.AddField(slack.Field{Title: header.Get("Subject"), Value: string(body)})

	username, _ := os.Hostname()
	payload := slack.Payload{
		Username:    username,
		Text:        string(body),
		Attachments: []slack.Attachment{attachment1},
	}

	errs := slack.Send(webhookURL, "", payload)
	if len(errs) > 0 {
		fmt.Printf("error: %s\n", errs)
	}
}
```

ビルドするときに`main.webhookURL`を渡せばOKです。こんな感じですね。

```
GOOS=linux go build -ldflags="-X main.webhookURL=${YOUR_WEBHOOK_URL}" -o override_bin/sendmail main.go
```



## docker + crondのデモ

先ほどの`override_bin/sendmail`と、次の内容で`crontab/root`ファイルを用意して、Dockerfileを記述します。

```crontab:crontabs/root
MAILTO="dummy-mailto"
* * * * * /bin/not_found
```

Dockerfileはこのようにbusyboxベースで。

```dockerfile:Dockerfile
FROM alpine:latest AS cafile
RUN apk add --no-cache ca-certificates


FROM busybox:latest
## これをもってこないとCAの関係で各所へのHTTPSのコネクションが張りづらい。
COPY --from=cafile /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt

ADD crontabs /var/spool/cron/crontabs
ADD override_bin/sendmail /bin/sendmail

ENTRYPOINT ["/bin/crond", "-f", "-d", "4", "-L", "/dev/stderr"]
```

### 実行

```
$ docker run -it --rm fakesendmail:latest
crond: crond (busybox 1.26.2) started, log level 4

...(中略)

crond: wakeup dt=28
crond: file root:
crond:  line /bin/not_found
crond:  job: 0 /bin/not_found
crond: child running /bin/sh
crond: USER root pid   6 cmd /bin/not_found
crond: wakeup dt=10
crond: child running sendmail # sendmailコマンドをキック

...
```


OK, slackに来ましたわ。

![sendmail.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/4ff96e9f-2580-3ca8-a7a4-1877b87346cf.jpeg "sendmail.jpg")


余談ですが、このデモ通りにcrond(busybox版)を単発で実行するだけのdockerコンテナはstopで素直に止まらないので、実際は[s6-overlay](https://blog.tutum.co/2015/05/20/s6-made-easy-with-the-s6-overlay/)等をカマすとちょっと扱いやすくなります。

## おわりに

まともにSMTPにのせることも再送やルーティングなどそれなりの利点はあるので、これでも十分ってところでだけ使いましょう。

`sendmail`が受け取っている引数と入力を単純に吐き出して確認する場合、こんなスクリプトに差し替えてから色々動かしてみるとよいです。


```shell:/path/to/sendmail
#!/bin/bash

echo $@ >> /tmp/sendmail_args
cat - >> /tmp/sendmail_stdin
```


