
ただ、レスポンスを動的に組み立てるだけの話です。

## botではない、単発のSlackのスラッシュコマンド(Slash Commands)

一応簡単に説明しておきます。スラッシュコマンドとは、`/foo [text]`のように、Slackのチャンネルで頭にスラッシュをつけて入力することでなにかしら処理をしてもらうためのシンタックスです。

- [Slash Commands | Slack](https://api.slack.com/slash-commands)



さてこれ、一見Slack Applicationか、botフレームワークの知識が入りそうですが、プリセットのApp`Slash Commands`を使えば次のようなゆるゆる仕様で取り扱うことができます。

- `plain/text`でレスポンスが来たら、Bodyをそのままチャンネルに投稿する。
- `application/json`でレスポンスをしたら、Bodyをパースして規定の形式でチャンネルに投稿する。
- HTTP 200 以外のレスポンスはエラー


![Slash Commands | Slack App Directory 2017-12-18 15-38-11.png](https://qiita-image-store.s3.amazonaws.com/0/7454/ee18fc91-69f8-738e-ed57-afb6652e99ee.png "Slash Commands | Slack App Directory 2017-12-18 15-38-11.png")

これならほぼ絵文字感覚で追加できます。

## テキストでシンプルなレスポンス

ではこんなのを作ります。

- `/hello`コマンドで、"Hello World"のレスポンス。

まずは`ngx_mruby`でのhello-world use_caseそのままです。


- [hello-world | ngx_mruby use_case](https://github.com/matsumotory/ngx_mruby/tree/master/docs/use_case#hello-world)

```ruby:hello.rb
Nginx.echo "Hello World"
```


```nginx.conf
server {
  location /hello {
    mruby_content_handler /path/to/hello.rb;
  }
}
```


```bash:Access
$ curl http://127.0.0.1/hello
Hello World
```

slackの設定をします。今回はGETですね。

![Image 2017-12-18 15-46-20.png](https://qiita-image-store.s3.amazonaws.com/0/7454/8ffd8c6b-59c2-1dfd-fb84-9aef68a10d4d.png "Image 2017-12-18 15-46-20.png")



`/hello`を実行、と。

![Slack - higanworks-idobata 2017-12-18 15-44-20.png](https://qiita-image-store.s3.amazonaws.com/0/7454/8977c451-7f23-a82e-6552-73c7d9a9a16a.png "Slack - higanworks-idobata 2017-12-18 15-44-20.png")


`Only visible to you`とあるように、このメッセージは自分しか確認できませんが、とりあえず反応してもらえました。結果が他に見えない場合、入力したコマンドもチャンネルには表示されません。

見た目などはSlackの設定で好きなように変更しましょう。今回はカレンダーのネタ出しを私に振ってきた[@matsumotory](https://twitter.com/matsumotory)を使います。

## JSONでレスポンス

惜しいことにプレーンテキストでは、コマンドを実行したユーザでしか実行結果が見えません。
このあたりの挙動を変更するのも簡単で、レスポンスを`application/json`にして`response_type`パラメータを`in_channel`(デフォルトの挙動は`ephemeral`)にするなどでOKです。

ついでにスラッシュコマンドの引数によって、すこし挙動を変えたりしてみます。

```ruby:hello2.rb
def args_to_hash(ar)
    h = {}
    ar.each do |a|
        h[a.split('=')[0]] = a.split('=')[1]
    end
    h
end

r = Nginx::Request.new
raw_args = r.args.split('&')
h_args = args_to_hash(raw_args)

resp = {}
resp['response_type'] = 'in_channel'

resp['text'] = h_args['user_name'] + "さん、こんにちは。\n"

if h_args['text'] == nil
    resp['text'] = resp['text'] + "それ、引数がないっていわれませんか？"
else
    resp['text'] = resp['text'] + "ほう、『#{HTTP::URL::decode(h_args['text'])}』、ですか。興味はあります。\n"
end


hout = Nginx::Headers_out.new
hout["Content-type"] = "application/json"

Nginx.echo JSON::stringify(resp)
Nginx.return Nginx::HTTP_OK
```

組み込みのヘルパーっぽい関数としても`args_to_hash`はある(※)んですが、keyに対するvalueが空っぽの場合(例: `&key1=&key2=value2` のようなケースのkey1)にコケるので一旦自前で実装。 

> - ※[ngx_mrubyの裏(不人気)機能(2) - Qiita](https://qiita.com/matsumotory/items/ab6e507d162f447c9c5d)
> 問題は修正して取り込んだので、次のリリースくらいからは`r.uri_args[param_key]`で問題なく取れそうです。 https://github.com/matsumotory/ngx_mruby/pull/327

こんな感じになりますね。かんたん。

![Slack - higanworks-idobata 2017-12-18 16-07-05.png](https://qiita-image-store.s3.amazonaws.com/0/7454/5e25dd1e-4942-934f-def0-9e985baeb510.png "Slack - higanworks-idobata 2017-12-18 16-07-05.png")



## サンプルコマンド、 ngx_mrubyでテキスト翻訳

では、コマンドに渡したテキストの翻訳でもしてもらいましょう。from to で変換する言語を指定するタイプで。


```ruby:hello3.rb
r = Nginx::Request.new

usage = '引数のことなんですけど、3ついるんですよ。 このようにしてください => `/hello en ja Hello World.`'
MS_KEY = 'MicrosoftTranslatorのAPIキー'
V_TOKEN = 'SLACKがパラメータにつけるTOKEN、任意で変更可能'


def get_token
    http = HttpRequest.new()
    res = http.post(
       'https://api.cognitive.microsoft.com/sts/v1.0/issueToken',
        '{}',
        {
            'Content-Type' => 'application/json',
            'Accept'       => 'application/jwt',
            'Ocp-Apim-Subscription-Key' => MS_KEY
        }
    )
    token = res.body
    token
end


## ここで翻訳。
def trans(from, to, text)
    utext = HTTP::URL::encode(text)
    http = HttpRequest.new()
    res = http.get(
       "https://api.microsofttranslator.com/V2/Http.svc/Translate?category=generalnn&from=#{from}&to=#{to}&text=#{utext}",
        nil,
        { 'Authorization' => 'Bearer ' + get_token }
    )
    t_txt = res.body

    t_txt.match(/<string.*?>(.*)<\/string>$/)[1]
end

def args_to_hash(ar)
    {}.tap do |h|
	    ar.each do |a|
    	    h[a.split('=')[0]] = a.split('=')[1]
	    end
    end
end

raw_args = r.args.split('&')
h_args = args_to_hash(raw_args)


## Nginx.returnからのlambdaについては => http://lamanotrama.hateblo.jp/entry/2015/08/02/005930
Nginx.return -> do
    # Slackからのリクエストであるっぽい、に制限するためトークンをチェック。
    if h_args['token'] != V_TOKEN
        Nginx.errlogger Nginx::LOG_WARN, "NO TOKEN ACCESS"
        return Nginx::HTTP_FORBIDDEN
    end

    resp = {}
    resp['response_type'] = 'ephemeral'

    if h_args['text'] == nil
        resp['text'] = usage
    else
        a_args = h_args['text'].split('+')
        if a_args.length < 3
            resp['text'] = usage
        else         
            t_body = a_args[2..(a_args.length)].join('+')
            resp['text'] = h_args['user_name'] + "さん、こんにちは。\n"
            begin
                resp['text'] = resp['text'] + "『#{HTTP::URL::decode(t_body)}』、そうですね。\n"
                resp['text'] = resp['text'] + "翻訳したら、『#{trans(a_args[0], a_args[1], HTTP::URL::decode(t_body))}』、ですかね。"
                resp['response_type'] = 'in_channel'

            # さすがに例外になりやすいところなのでざっくり拾う
            rescue => e
                Nginx.errlogger Nginx::LOG_ERR, e
                resp['text'] = "`Some error occurerd on calling Microsoft Translate API.` (this message only visible to you.)\n maybe unsupported language.\n See https://msdn.microsoft.com/ja-jp/library/hh456380.aspx."
            end
        end
    end

    hout = Nginx::Headers_out.new
    hout["Content-type"] = "application/json"

    Nginx.echo JSON::stringify(resp)
    return Nginx::HTTP_OK
end.call
```



![Slack - higanworks-idobata 2017-12-18 17-01-20.png](https://qiita-image-store.s3.amazonaws.com/0/7454/23bac855-f6ac-6242-2a5d-7ab19914bdbc.png "Slack - higanworks-idobata 2017-12-18 17-01-20.png")


## 今回使った環境と感想

まず環境。

- Amazon EC2のインスタンスにNginx + ngx_mruby
  - ngx_mrubyは自分が保守している [ngx_mruby RPM for amazon linux 1](https://packagecloud.io/amimoto-nginx-mainline/main) があるので。
- [AWS Cloud9](https://aws.amazon.com/jp/cloud9/)
  - nginx + ngx_mruby環境のEC2として。
  - IDEでmrubyスクリプトの編集。
- [ngrok](https://ngrok.com/)
  - slash commands対象のHTTPSホストにした。
  - 本格運用するならElasticIP+適当なEC2とか、ALB+Fargateとか。
- [Translator API - Microsoft Translator](https://www.microsoft.com/ja-jp/translator/translatorapi.aspx)


上記構成にはあまり必然性はなく、Cloud9を触ってるうちに思いついたというだけなので。試してみるならローカル+ngrokで十分です。


さて、たいていこういうのはFaaSでやるのが一般的で、よほどの理由がなければあえてngx_mrubyでやる必要もないでしょう。ただ、slack側のタイムアウトがわりと厳しくて、FaaS側のウォームアップ周り次第でレスポンスが間に合わないことがある、と言う話も聞いたことはあるので意外とありなのかもしれません。

nginxのlocation + rbファイルでわりとやりたい放題、デバッグもまあまあやりやすいので、遊んでるnginxがいたらスラッシュコマンドエンドポイントを生やしても良いかもですよ。

[mruby_add_handler](https://qiita.com/matsumotory/items/f48f41619cf53acdad45)と組み合わせると、ルーズにコマンド追加・管理もできそう。
