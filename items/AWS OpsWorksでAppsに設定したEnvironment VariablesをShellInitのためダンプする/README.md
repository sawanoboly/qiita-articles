
OpsWorksのAppsに定義した環境変数をシェルや他の処理からも使いやすくしておきたい。

仕様まわりはこちらに解説があるので割愛。 > [OpsWorksで機能追加された環境変数を試してみて気づいた注意すべきこと - よかろうもん！](http://interu.hatenablog.com/entry/2014/10/27/223954 "OpsWorksで機能追加された環境変数を試してみて気づいた注意すべきこと - よかろうもん！")


Elastic Beanstalkみたいに(※)楽には取れないので、フィードバックから要望を送信しつつ、レシピで対応。

> ※コンテナタイプにもよるが、`/opt/elasticbeanstalk/support/export_envvars`でEnvironment Configurationの環境変数を取れる。


## カスタムクックブック

[OpsRockin/opsworks-export-envs | GitHub](https://github.com/OpsRockin/opsworks-export-envs)

幸いデプロイイベントには必要な情報が揃っているので、deployのランリストに追加すればよし。


```opsworks-export-envs/recipes/default.rb
node.deploy.each_pair do |app_name,vals|
  file File.join(vals[:home], "shellinit-#{app_name}.sh") do
    envs = []
    vals.environment_variables.each_pair {|key, val| envs << "export #{key}=\"#{val}\""} if vals.environment_variables
    owner vals[:user]
    group vals[:group]
    content envs.join("\n")
  end
end
```

## export用ファイルのサンプル

こんな感じで各アプリケーションに設定された環境変数を、読めるように書き出します。

```/home/deploy/shellinit-your_app_name.sh
export HOGE1="piyopiyo1"
export HOGE2="piyopiyo2"
```

使うときは`. /home/deploy/shellinit-your_app_name.sh` や `source /home/deploy/shellinit-your_app_name.sh` で読めばOK。
ファイル自体を置く場所は`/etc/profile.d/`や.bashrcで自動で読ませるとか、各自使いやすいように調整してもいいですね。
