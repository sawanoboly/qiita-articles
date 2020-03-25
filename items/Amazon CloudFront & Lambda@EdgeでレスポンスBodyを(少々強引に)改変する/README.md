Amazon CloudFrontのLambda@Edgeは、HTTPリクエスト/レスポンスに対して任意のタイミングでLambdaをカマす機能です。

さて、今回はレスポンスのHTMLをちょっと加工したいなと思って調べました。

実装例のドキュメントはこちら。

- [リクエストトリガーでの HTTP レスポンスの生成 - Amazon CloudFront](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/lambda-generating-http-responses.html)
- [origin-response トリガーでの HTTP レスポンスの更新 - Amazon CloudFront](https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/lambda-updating-http-responses.html)

> 追記: bodyさわれるようになりましたね。以下はもうoutdated

----

これらによると、以下の制限があるとのこと。

> HTTP レスポンスを使用する場合、Lambda@Edge は、オリジンサーバーから返された HTML 本文を origin-response トリガーに公開しません。


bodyに手を出せないのかよ！ オリジンからのレスポンスに対する通常の利用範囲内では、bodyはまるごと上書きのみ可能、とのこと。
ちょっと内容変更して配信したい、というケースに対応するには。。？

そこでヒントにしたのがこちら。

- [Amazon CloudFront & Lambda@Edge で画像をリサイズする | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/resizing-images-with-amazon-cloudfront-lambdaedge-aws-cdn-blog/)

わりとそのまんまの解決策だった。これをすこし方針変更してテキスト処理にすればよさそう。

## 方針

- 改変したいのはHTMLだけ
- オリジンはAmazon S3

## How to

- origin-responseの時点でLambdaコール
  - ここで書き換えればキャッシュにのるので
- STATUS=200かつ`.html`のリクエストを判定
  - ヘッダは使い回す
  - bodyには、S3からオブジェクトを直接取得してから改変したコンテンツを突っ込む
- IAM Roleに割り当てるPolicyではgetObjectを許可

## コード

で、origin-responseイベントのところで動作するLambdaスクリプトをこのようにしました。
content-lengthの再計算いるかなと思ったけど、このイベントの時点ではcontent-lengthヘッダはついてなかったので、レスポンス最終段階で付与されるんだろうと。

サンプルではBody中の`WordPress`という単語を`getShifter`に置換してます。


```javascript:origin_response.js
'use strict';
const util = require('util');
const AWS = require('aws-sdk');
const S3 = new AWS.S3({
  signatureVersion: 'v4',
});

exports.handler = (event, context, callback) => {
  console.log(util.inspect(event.Records[0].cf));
  let response = event.Records[0].cf.response;

  if (response.status == 200) {
    let request = event.Records[0].cf.request;
    let bucket = request.origin.s3.domainName.split('.')[0];
    let path = request.uri;

    if (path.endsWith('\.html')) {
      let key = path.substring(1);
      S3.getObject({ Bucket: bucket, Key: key }).promise()
        // perform the replace operation
        .then(data => data.Body.toString()
          .replace(/WordPress/g, 'getShifter')
        )
        .then(buffer => {
          // console.log(buffer);
          response.body = buffer;
          callback(null, response);
          return;
        })
        .catch( err => {
          console.log("Exception while reading source :%j",err);
        });
    } else {
      callback(null, response);
    }
  } else {
    callback(null, response);
  }
};
```

一応Edgeの厳し目なタイムアウト制限(5s)の範囲には収まっているっぽく、置換されたコンテンツを取得できました。

パススルー用のcallbackをelseの外に置いてたらうまくいかなかったのはなんでやろうなあ。。


## 使用前/使用後

![before.png](https://qiita-image-store.s3.amazonaws.com/0/7454/36ec7843-8bc6-7fae-5838-a4ff64c97a57.png)

↓↓↓変わった。

![after.png](https://qiita-image-store.s3.amazonaws.com/0/7454/15cd1d9d-0b7c-b7fe-aacd-ea31a2168a23.png)

初回ちょっと遅いけど、キャッシュには改変後が乗るのでよいかな。
