
[Amazon Elasticsearch Service](https://aws.amazon.com/jp/elasticsearch-service/)はElasticsearchです。
普通にElasticsearchとしてエンドポイントを使用でき、アクセスポリシーは次の中から選びます。

- Any
- 送信元IPアドレス
- IAM(STS)

さて、ちょっと都合でS3に置かれたファイルを軽くパースしてElasticsearchに突っ込もうかと、IAMベースでLambdaからのリクエストを思ったところ、現状では署名リクエストを自前でつくって放り込むっぽい。

結果。やってはみたが、今は一旦CloudWatch Logsのストリームに流してからプリセットのファンクションを使うほうが結局楽に済みそう。ES用にboto3のresourceが実装されれば苦労しなくてよさそうだ。

とりあえず動いたのでコードをメモしておこう。

## Elasticsearchドメイン側のポリシー

- IAMテンプレートでLambdaに適用したロールを許可する
   - `es:*`なり、`es:ESHttpGet`など

## Lambdaファンクション

EventSourceのことはちょっと置いといて、とりあえずESの`/`にGETを送るだけ。
次の資料が参考になった。

- [完全なバージョン 4 署名プロセスの例（Python） - アマゾン ウェブ サービス](http://docs.aws.amazon.com/ja_jp/general/latest/gr/sigv4-signed-request-examples.html "完全なバージョン 4 署名プロセスの例（Python） - アマゾン ウェブ サービス")
    - Authorizationヘッダを作成するための関数を持ってきた。
- CloudWatch Logsで作成できる、`LogsToElasticsearch`のJavascript
    - Authorization以外ヘッダや環境変数の参考にした。
    - だいたいコレを移植した感じ。

ほとんどがヘッダを作るためのコード。。。

```function.py
import urllib2, datetime, os, sys
import hashlib, hmac

endpoint = 'search-sandbox01-xxxxxxxxxxxxxxxxxxxx.ap-northeast-1.es.amazonaws.com'
url = "https://" + endpoint
region = 'ap-northeast-1'
service = 'es'
method = 'GET'

access_key = os.environ.get('AWS_ACCESS_KEY_ID')
secret_key = os.environ.get('AWS_SECRET_ACCESS_KEY')
if access_key is None or secret_key is None:
    print 'No access key is available.'
    sys.exit()

def sign(key, msg):
    return hmac.new(key, msg.encode('utf-8'), hashlib.sha256).digest()

def getSignatureKey(key, dateStamp, regionName, serviceName):
    kDate = sign(('AWS4' + key).encode('utf-8'), dateStamp)
    kRegion = sign(kDate, regionName)
    kService = sign(kRegion, serviceName)
    kSigning = sign(kService, 'aws4_request')
    return kSigning


def lambda_handler(event, context):
    t = datetime.datetime.utcnow()
    amzdate = t.strftime('%Y%m%dT%H%M%SZ')
    datestamp = t.strftime('%Y%m%d') # Date w/o time, used in credential scope
    
    canonical_uri = '/'
    canonical_querystring = ""
    canonical_headers = 'host:' + endpoint + '\n' + 'x-amz-date:' + amzdate + '\n'
    signed_headers = 'host;x-amz-date'
    payload_hash = hashlib.sha256('').hexdigest()
    canonical_request = method + '\n' + canonical_uri + '\n' + canonical_querystring + '\n' + canonical_headers + '\n' + signed_headers + '\n' + payload_hash
    
    algorithm = 'AWS4-HMAC-SHA256'
    credential_scope = datestamp + '/' + region + '/' + service + '/' + 'aws4_request'
    string_to_sign = algorithm + '\n' +  amzdate + '\n' +  credential_scope + '\n' +  hashlib.sha256(canonical_request).hexdigest()
    
    signing_key = getSignatureKey(secret_key, datestamp, region, service)
    signature = hmac.new(signing_key, (string_to_sign).encode('utf-8'), hashlib.sha256).hexdigest()
    
    authorization_header = algorithm + ' ' + 'Credential=' + access_key + '/' + credential_scope + ', ' +  'SignedHeaders=' + signed_headers + ', ' + 'Signature=' + signature
    
    headers = {
          'Content-Type': 'application/json',
                'Host': endpoint,
                'X-Amz-Security-Token': os.environ.get('AWS_SESSION_TOKEN'),
                'X-Amz-Date': amzdate,
                'Authorization': authorization_header
    }
    
    req = urllib2.Request(url, None, headers)
    
    try:
        print urllib2.urlopen(req).read()
    except urllib2.URLError, e:
        print e.read()
```

このヘッダ、botocoreのモジュール使えばさくっと作るんだろうか？ と思いつつ。

> 追記：その、『botocoreのモジュール使えばさくっと作る』の例はこちら。
> [[小ネタ] botocoreのAWS APIリクエストの署名プロセスのみを利用する ｜ Developers.IO](http://dev.classmethod.jp/cloud/aws/botocore-signed-process/ "[小ネタ] botocoreのAWS APIリクエストの署名プロセスのみを利用する ｜ Developers.IO")


## You Know, for Search

一応手元からESにcURLしてみると、ちゃんとブロックされている。

```
$ curl search-sandbox01-xxxxxxxxxx.ap-northeast-1.es.amazonaws.com
{"Message":"User: anonymous is not authorized to perform: es:ESHttpGet on resource: arn:aws:es:ap-northeast-1:xxxxxxxxx:domain/sandbox01/"}
```

Lambdaを開き、前述のコードを`Save and Test`すると、無事にESからの返事がきたね。

```
START RequestId: b6ce7399-85eb-11e5-b2c3-99a74c2c1765 Version: $LATEST
{
  "status" : 200,
  "name" : "Chief Examiner",
  "cluster_name" : "xxxxxxxxxxx:sandbox01",
  "version" : {
    "number" : "1.5.2",
    "build_hash" : "62ff9868b4c8a0c45860bebb259e21980778ab1c",
    "build_timestamp" : "2015-04-27T09:21:06Z",
    "build_snapshot" : false,
    "lucene_version" : "4.10.4"
  },
  "tagline" : "You Know, for Search"
}

END RequestId: b6ce7399-85eb-11e5-b2c3-99a74c2c1765
REPORT RequestId: b6ce7399-85eb-11e5-b2c3-99a74c2c1765	Duration: 64.87 ms	Billed Duration: 100 ms 	Memory Size: 128 MB	Max Memory Used: 16 MB
```


