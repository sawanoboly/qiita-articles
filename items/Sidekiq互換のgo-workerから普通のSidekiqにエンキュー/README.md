
非同期ジョブをSidekiqのようなものでやりたいなあ、でもgolangで受け付けたいなあ。とか考えていたら、互換実装があったので試してみた。

[jrallison/go-workers: Sidekiq compatible background workers in golang](https://github.com/jrallison/go-workers)

## RubyでSidekiqのジョブを書く

とりあえず引数をputsするだけのジョブ。

```ruby:sample.rb
require 'sidekiq'

class SampleWorker
  include Sidekiq::Worker
  def perform(*opts)
    puts "Kicked with options: #{opts.to_s}"
  end
end
```

んで、Sidekiqを起動しておきます。

```shell:console-sidekiq
$ sidekiq -r ./sample.rb -q myqueue 


         m,
         `$b
    .ss,  $$:         .,d$
    `$$P,d$P'    .,md$P"'
     ,$$$$$bmmd$$$P^'
   .d$$$$$$$$$$P'
   $$^' `"^$$$'       ____  _     _      _    _
   $:     ,$$:       / ___|(_) __| | ___| | _(_) __ _
   `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |

...
```

OK。

## Goでエンキューの処理を書く

[go-workers](https://github.com/jrallison/go-workers)の`Enqueue`を使ってみる。Configureは最低限これらのオプションが必要だった。

```golang:main.go
package main

import "github.com/jrallison/go-workers"

func main() {
	workers.Configure(map[string]string{
		"server":   "127.0.0.1:6379",
		"process":  "1",
	})

	// デフォルトオプションでエンキュー
	workers.Enqueue("myqueue", "SampleWorker", []int{1, 2})

	// リトライオプション指定でエンキュー
	workers.EnqueueWithOptions("myqueue", "SampleWorker", []string{"job", "jober"}, workers.EnqueueOptions{Retry: true, RetryCount: 10})
}
```


`go run main.go`とすると、sidekiqのワーカーが動いた。


```shell:console-sidekiq
SampleWorker JID-a25fa4a81126e990406bb30e INFO: start
Kicked with options: [1, 2]

SampleWorker JID-8bc1738220cf1e2303279054 INFO: start
Kicked with options: ["job", "jober"]

SampleWorker JID-8bc1738220cf1e2303279054 INFO: done: 0.0 sec
SampleWorker JID-a25fa4a81126e990406bb30e INFO: done: 0.001 sec
```



ただ、ざっくりみた感じではSentinel/Clusterでredisクライアントを作るようなオプション指定はサポートしてないように見えた。


[go-workers](https://github.com/jrallison/go-workers)はREADMEにあるとおりServer側の機能も実装してあるので、逆に既存のSidekiqプロセスの代わり(※sidekiq-webは本家の使い回し)をすることもできるようだけど、その場合はSidekiq互換にこだわる必要もない気はした。
極端な話、redisにsaddとrpushすりゃあSidekiqは動くので、他にも色々やり方は考えられそう。
