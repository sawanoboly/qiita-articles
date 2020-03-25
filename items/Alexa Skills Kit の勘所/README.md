このエントリは2016-07-27（水）開催の
[Alexa meetup #01 | JAWS-UG Kobe](https://jaws-ug-kobe.doorkeeper.jp/events/47410)
での発表スライドです。

※表示にはQiitaのスライドモード全画面を推奨。(モードが残っていれば)

---

## このエントリで扱う範囲

- Skills Kitの作成と公開に限定。
- Smart Home対応デバイス連携はまた別の話なのでゴメン。
- Getting Startedでは微妙にわかりづらい事を補う感じです。

---

## Alexa Skills Kit ?

![Alexa_Skills_Kit__ASK__-_Amazon_Apps___Games_Developer_Portal.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/d242a4cb-6c95-75ed-1564-54d7ff7a94ef.jpeg)

- Alexaデバイス用の自作APIです。
- 実体はAmazon Lambdaのfunction.
- Echoに話した文章を聞き取って、アクションを実行します。


---

## Alexa Amimoto Ninja

[![Alexa_AMIMOTO_Ninja___High_Performance_WordPress_AWS_AMI.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/43283d03-dcee-983b-4d56-1a441f44754f.jpeg)](https://amimoto-ami.com/amimoto-ninja/)

---

# Amimoto Ninjaとは

EUのWordCampブース設置用に(<del>ネタで</del>)作成。(※ Alexa本来の利用法とは相当かけ離れています。)

- 名前を聞く(セッション中のみ記憶)。
- Amimotoに関する質問をしてもらい、回答する。
    - 聞き取れなかったらランダムにお勧めを紹介する。 
- 感想を聞いて、『名前 + 感想』をWordpress及びTwitterにポストする。
    - ついでにKinesis firehoseでs3にログを保存する。

ソースは公開： [amimoto-ami/amimoto-amazon-alexa | Github](https://github.com/amimoto-ami/amimoto-amazon-alexa)

---

## 反響はひっそりそこそこ。

![Takayuki_MiyauchiさんはInstagramを利用しています_「Alexaと口喧嘩をする某有名エンジニア。」.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/264cfc56-8898-3b71-eb43-48b7af674ff0.jpeg)


[https://www.instagram.com/p/BHCiP-rjzJf/](https://www.instagram.com/p/BHCiP-rjzJf/)

---

## Skills Kitのフロー

- AlexaにSkill起動を指示(`ask amimoto ninja`)。
- セッションを初期化して、起動したSkillが何か案内を流す。
- Skill側の案内に応じて、人が何かを話す。
- 話した内容をAlexa(サービス側)が言語解析、 *定義済の* どの意図にあたるかを自動判別する。
- 意図＋変数のセットでLambda functionがキックされる
- Lambda functionでなんらかの処理を行い、レスポンスを作る。
- レスポンスがEchoデバイスから発音される。

---

## Echoデバイスで使えるようにするには？

- Lambdaファンクションのデプロイ
- Amazon Developer console [https://developer.amazon.com/alexa-skills-kit]でSkillの登録

の2系統を処理する必要があります。

---

## Skills Kitの要素

- **Intent**
    - 定義する意図(Intent)、メソッドみたいなもの。組み込みとユーザ定義がある。Lambdaではこれを判断してDispatchする。
- **Utterances**
    - 意図(Intent)と自然言語のマッピング。変数としてあつかう場所を指定できる。
- **Slot**
    - 言葉の選択肢をリストとして定義、組み込みもある。フリーワード(聞き取れた通りの単語を使用する)として扱わせるように指定も可能。フロー説明での変数にあたる。
- **Intent Schema**
    - このSkillがもつ意図とSlotの一覧。ユーザ定義も組み込みも、これに書いたものだけが使用できる。
- **Session**
    - Skills起動時に初期化され、IDが振られる。セッション中は少々の変数をEchoデバイスと共有できる。

---

## 要素を踏まえて、Skills Kitの具体的フロー 

Skill例『GymLeader Spy Go』 質問:『XXXのジムリーダーは誰？』

- Alexa(サービスが)言語解析。`ジムリーダーは誰？`にマッチするので、XXXを変数と判断。
    - Lambdaを`Intent=WhoisLeader`, `Location(Slot)=XXX`でキック
    - XXXがフリーワードで判断しづらい場合、頑張ってリストをつくってSlotにする。
- Lambda内で他所のAPIにジムリーダー情報を問い合わせる。
    - Echoデバイスへのレスポンスを組み立てる。
- 『XXXのジムリーダーはBさんです。』などとEchoデバイスがしゃべる。

※ Alexaユーザ名とゲームのScreenNameから、『あなた様です、流石です』などと言わせたい場合は、別途ひも付けのDBを用意する必要がある。

---

## Intent & Utterances

---

### 組み込み Intent

![Implementing_the_Built-in_Intents_-_Amazon_Apps___Services_Developer_Portal.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/c1ca7633-8a34-68de-ff4f-3208c1d0653c.jpeg)

- [Implementing the Built-in Intents - Amazon Apps & Services Developer Portal](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/implementing-the-built-in-intents)

---

### ユーザー定義 Utterances

Amimoto Ninjaで使用しているIntentとUtterancesの例。`{例文|Slot名}`で変数。

```
MyNameIsIntent my name is {john smith|VisitorName}
MyNameIsIntent i am {john smith|VisitorName}
MyNameIsIntent i'm {john smith|VisitorName}
WhatIsIntent what is {amimoto name|AskedQuestion}
WhatIsIntent what's {amimoto name|AskedQuestion}
WhatIsIntent what about {amimoto name|AskedQuestion}
WhatIsIntent tell me about {amimoto name|AskedQuestion}
CanIUseIntent can i use {free trial|AskedQuestion}
CanIUseIntent do you have {free trial|AskedQuestion}
CanIUseIntent how about {free trial|AskedQuestion}
...
```

----

### Intent & Utterances (& Session)勘所

- 言語解析の対応部分は基本的に **小文字** で記述しましょう。
    - 人間は大文字や、アラビア数字を発声しているわけではありません。
    - `Wordpress 4.1`と聞き取りたい > `wordpress four point one` として渡される。
- 各Intentはどのタイミングでも送信されることを念頭に置く。
    - Amimoto Ninjaのように、想定した応答シナリオがあるとわりとつらい。
    - セッション中は`session_attribute`という格納領域があるので、名前を格納したり状態の管理につかった。
    - Alexa本体開発者からのお勧めは？ => DynamoDBを使えばいいじゃん。
- Utterancesの`{例文|Slot名}`の記述。
    - 例文の単語がひとつなら、単語として聞き取られる。
    - 2語以上なら、文章として聞き取ってくれる。
- YesIntent, NoIntentは精度が高くて便利。
    - ただ、これもシナリオ上のどのシチュエーションかによって更に内部でDispatchする必要がある。
- Skill公開申請時にはHelpIntentが必須です。

---

## Slot

このようにTypeを指定。

```
    {
      "intent": "MyNameIsIntent",
      "slots": [
        {
          "name": "VisitorName",
          "type": "AMAZON.LITERAL"
        }
      ]
    },
```


----

## Slotの大雑把な種類わけと勘所

- 組み込み Slot (次頁で紹介)
    - Typeから選んで、内容を丸めたりLambdaに渡す前に自動変換を行える。
    - 便利なんだけど、英語のみ対応。
- リスト Slot
    - テキスト形式でリスト(ASCII文字のみ)を登録しておく。
    - 聞き取った内容をある程度丸めることができる(のかも)？
    - リストから見つからない場合の対応も必要。
- フリーワード Slot
    - `Type: AMAZON.LITERAL`でフリーワード
    - (英語ネイティブでなければ)基本的に地獄

----

### 組み込みSlot紹介

![Alexa_Skills_Kit_Custom_Interaction_Model_Reference_-_Amazon_Apps___Services_Developer_Portal.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/5487150e-2322-b53a-e498-04ab608d74cc.jpeg)

[Alexa Skills Kit Custom Interaction Model Reference - Amazon Apps & Services Developer Portal](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/alexa-skills-kit-interaction-model-reference)

----

### フリーワードSlotの地獄

日本人は言いました...『wordpress』

> Alexa heard...
> `the valkyries` , `worth press`, `well press`, `was the price`, `why the press`, `what the press`, `what race`

--- 

### Amimoto Ninjaでは別途リストを作って、無理やり丸めて対応。

```
---
wordpress: # (これら全部Wordpressと聞こえたことにする)
  - the valkyries
  - worth press
  - well press
  - was the price
  - why the press
  - what the press
  - what race

amimoto: # (これら全部AMIMOTOと聞こえたことにする)
  - an immortal
  - a motel
  - available
  - a moto
  - it i'm a mother
  - ammy moto
```

---

## レスポンススピーチ

---

### レスポンスのスピーチで用意するテキストは3種類

- outputSpeech
    - まずしゃべる内容。はじめの案内や、聞かれたことへの回答が主。
- reprompt
    - 数秒応答がなかったらしゃべる内容。
    - 繰り返してもよいし、ヘルプっぽい案内でもよい。
- card
    - AlexaユーザーのWebコンソールに表示する内容
    - 音声でやり取りできない事情への対応かな？

---

## Cardが(自分には)分かりづらかったので例

ここにこのように出ます。

<img width="980" alt="スクリーンショット_2016-07-26_13_59_56.jpg" src="https://qiita-image-store.s3.amazonaws.com/0/7454/869a66ec-99ec-c716-56d5-1fe8ed2a4880.jpeg">

---

## レスポンススピーチ勘所

- SSML(Speech Synthesis Markup Language)の一部に対応している！
    - ブレスを入れたり、Echoに自然に話してもらうためにはプレーンテキストよりSSMLをつかおう。
    - 発音記号でAMIMOTOもなめらかに発音することができる。
    - `<p>Hi, I'm  <phoneme alphabet="ipa" ph="amimoʊtoʊ">AMIMOTO</phoneme> Ninja.</p>`
- Cardのテキストも使い回しで済ませたい場合はSSMLタグを除去する必要がある。
    - AMIMOTO NinjaではSSMLテキストをXMLパースしてテキストだけ抜き出すようにした。
    - ただ、Cardやrepromptを真面目に組み立てるのは、Skill公開申請する場合だけでよい。

---

## 聞き取れなかった際向けには、やさしい返事を作りましょう。 

---

## 回答を『Pardon?』だけとかにするとだとテストする人の心が折れていくようです。


---

## Lambda function

---

## Lambda function (Pythonの場合)

まずはLambdaに最初からあるサンプルを参考にしましょう。

- [Sample Interaction Model for the Color Expert Blueprint | Amazon Developer](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/developing-an-alexa-skill-as-a-lambda-function#Sample%20Interaction%20Model%20for%20the%20Color%20Expert%20Blueprint)

一応コツ？

- イベントにはAlexaを指定します(Webコンソールからのみ可)。
- 最初のみ`['session']['new']`が`true`でくるので、判別して初期化処理します。
- 最初のハンドラ(lambda_handler)は、LaunchかEndか、または何らかのインテント、という3種類を振り分けておきます。

---

## あとはとにかく地道にDispatchします

※勘所とは言いづらいのですが。。こんな感じに作りました。

```python:
...
    # Dispatch to your skill's intent handlers
    if intent_name == "MyNameIsIntent":
        return set_visitor_name_from_session(intent, session)
    elif intent_name == "WhatIsIntent" or intent_name == "CanIUseIntent":
        return dispatch_question(intent, session)
    elif intent_name == "ImpressionIntent":
        return collect_impression(intent, session)
    elif intent_name == "AMAZON.YesIntent":
        return dispatch_yes_intent(intent, session)
    elif intent_name == "AMAZON.NoIntent":
        return dispatch_no_intent(intent, session)
    elif intent_name == "AMAZON.HelpIntent":
        return return_help_response(intent, session)
    elif intent_name == "AMAZON.CancelIntent" or intent_name == "AMAZON.StopIntent":
        return handle_session_end_request(intent, session)
    else:
        raise ValueError("Invalid intent")
...
```

---

## Skillのデバッグ

まずはWebの管理画面で行います。Echoデバイスを持っていれば、そのまま実機テストも可能。

![Amazon_Apps___Services_Developer_Portal_と_「Alexa_Skills_Kit_の勘所」を編集_-_Qiita.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/7e979fe8-adf6-954b-ee84-c9217ff6686c.jpeg)


---

## デバッグ勘所

- Lambdaのログを見やすいように、`awslogs`コマンドまたは関連ツールをつかいましょう。
- 実機では、聞き取った内容がユーザのSkills管理画面に出てきます、確認につかいましょう。
- 実機確認、できればネイティブを連れてきましょう。。

---

## デプロイ

- Skillの定義(Interaction Model)は、まだAPIがありません。
    - `Intent Schema`, `Custom Slot`, `Utterances`はおとなしくブラウザから入力しましょう。
    - テキストなので、手元でGit管理にはしておきます。
- Lambda functionのデプロイは、lamveryをおすすめします。
    - [marcy-terui/lamvery: User-friendly deployment and management tool for AWS Lambda function](https://github.com/marcy-terui/lamvery)

---

### Amimoto Ninjaのデプロイ

1. CircleCIでPython文法、Yaml Syntaxのチェック
2. 自動デプロイでLambda functionを更新

[amimoto-ami/amimoto-amazon-alexa | CircleCI](https://circleci.com/gh/amimoto-ami/amimoto-amazon-alexa)

![amimoto-ami_amimoto-amazon-alexa__235_-_CircleCI.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/ebc2cabf-f42c-bdbd-8179-c7d7e9aaaafe.jpeg)


---

## ちょっと役に立つ情報

- 公開を前提にするなら、ベストプラクティスを遵守しましょう。審査でハネられます。
    - [Alexa Skills Kit Voice Design Best Practices - Amazon Apps & Services Developer Portal](https://developer.amazon.com/public/solutions/alexa/alexa-skills-kit/docs/alexa-skills-kit-voice-design-best-practices)

---

## おわり
