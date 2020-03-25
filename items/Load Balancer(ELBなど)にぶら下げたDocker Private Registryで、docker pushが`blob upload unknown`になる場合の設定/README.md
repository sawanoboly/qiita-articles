
ほか、症状としてはすぐ`Retrying...`などになっちゃう。

これの解決方法はログにでている。
`http.secret`をコンフィグファイルにセットするか、`REGISTRY_HTTP_SECRET`を環境変数で指定しましょうとのこと。

> level=warning msg="No HTTP secret provided - generated random secret. This may cause problems with uploads if multiple registries are behind a load-balancer. To provide a shared secret, fill in http.secret in the configuration file or set the REGISTRY_HTTP_SECRET environment variable." go.version=go1.6.3 instance.id=xxxx version=v2.5.0


指定がなければ起動時にそれぞれがランダム生成しちゃうのね。なるほど。

そしてセットした。

```
$ docker push reg.example.com/sawanoboly/ubuntu:latest
The push refers to a repository [reg.example.com/sawanoboly/ubuntu]

...

latest: digest: sha256:f92c2bec713942c44219e8bd513c255543c9acda764ccabbe3f940eca696e97e size: 1151
```

Push成功した。
