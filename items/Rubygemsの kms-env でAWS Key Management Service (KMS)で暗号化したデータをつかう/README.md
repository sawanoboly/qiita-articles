
AWS KMSを経由した暗号をRuby/Railsからしれっと使いたいので、Gemでも作るかと思ったら[kms-env](https://github.com/Fullscreen/kms-env)というのがあった。
これがなかなか利用シーンを想定した工夫がなされていたので紹介。


## 準備: 末尾に`_KMS`をつけた環境変数をセットする

たとえば`MY_TEXT1`という環境変数に目的の文字列を入れたい場合、`MY_TEXT1_KMS`という環境変数にCiphertextを入れておく。

```bash:
export MY_TEXT1_KMS="CiCIvMyU68+VaZSTfI3932heQDZdvp+5+HrByuWVyEcTRRKZAQEBAgB4iLzMlOvPlWmUk3yN/d9oXkA2Xb6fufh6wcrllchHE0UAAABwMG4GCSqGSIb3DQEHBqBhMF8CAQAwWgYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAxbSecwaBUfYce/0MQCARCALftL6ZVE9fAMgpohPz0A8hfylDx3wGg64M4J47GK2CTrI11pIylpKHnklMcIag=="
export MY_TEXT2_KMS="CiCIvMyU68+VaZSTfI3932heQDZdvp+5+HrByuWVyEcTRRKcAQEBAgB4iLzMlOvPlWmUk3yN/d9oXkA2Xb6fufh6wcrllchHE0UAAABzMHEGCSqGSIb3DQEHBqBkMGICAQAwXQYJKoZIhvcNAQcBMB4GCWCGSAFlAwQBLjARBAwgFJzQeMJU3a7NF5YCARCAMDE2ngCZllnIKUq9S5qdS9FuxWmlEW8OlmjSoE3chqmLgTuoXvwzwX7AltZg7jecmQ=="
```



## 使用: `KmsEnv::load`を実行すると、`_KMS`が取り除かれたキーでENVにセットされる

手元でやるならprofile、その他ならアクセスキーをつかう。もちろんEC2などAWS上ならロール/STSでいい。

```
export AWS_PROFILE=myprofile
export AWS_REGION=ap-northeast-1
```

動作の様子をPryで。

```ruby:pry-console
> ENV['MY_TEXT1']
=> nil  ## まだカラッポ

> require 'kms-env'
=> true


> KmsEnv.load
=> ["MY_TEXT2_KMS", "MY_TEXT1_KMS"]  # 処理対象が返される。


> puts ENV['MY_TEXT1']          # MY_TEXT1に復号されたデータがセットされた
Is this your pen?
=> nil

> puts ENV['MY_TEXT2']          # MY_TEXT2に復号されたデータがセットされた
Yes, this is my pen.
=> nil

## === ついでに既存の値は？

> ENV['MY_TEXT2'] = 'hoge'

> ENV['MY_TEXT2']          # MY_TEXT2を上書きしてからの...
=> hoge


(pry)> KmsEnv.load
=> ["MY_TEXT2_KMS", "MY_TEXT1_KMS"]  # もう一度ロードしてみる

(pry)> puts ENV['MY_TEXT2']          # またMY_TEXT2に復号されたデータがセットされた
Yes, this is my pen.
```

こいつはよい。initializeで`KmsEnv.load`した後は通常のENVとして使い放題だ。
対象(*_KMS)がなければ特に何もせず、既存の値を上書きしないのでローカルは平文、デプロイ先ではCiphertextという運用が可能なのも素敵。



## まとめ

サフィックスから自動でやるってのは考えつかなかった、副作用もすくなかろうし応用が効きそうだ。
自前でライブラリつくろうかなっと思った時はCLIで暗号化もできたらいいかもと思ったが、KMSでデータを変換するインターフェースはlamvery(encrypt/decrypt)があるからいいや。
