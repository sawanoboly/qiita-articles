
Azure WebAppのまとまったドキュメントが見当たらなくて、CLIの通信をベースに調べることにした。

調査には [Burp Proxy](http://portswigger.net/burp/proxy.html) を使い、`export HTTP_PROXY=http://127.0.0.1:8080`した後地道に取得。

## 共通項

リソースの共通部分を次のようにして記述を省略します。

エンドポイント: https://management.core.windows.net/${サブスクリプションID}

データセンタの場所にあたるwebSpaceNameがURIに含まれるので、適宜次の中から埋めます。

ロケーション(webSpaceName)一覧: 

```
[
  {
    "name": "North Central US",
    "webSpace": "northcentraluswebspace"
  },
  {
    "name": "North Europe",
    "webSpace": "northeuropewebspace"
  },
  {
    "name": "West Europe",
    "webSpace": "westeuropewebspace"
  },
  {
    "name": "South Central US",
    "webSpace": "southcentraluswebspace"
  },
  {
    "name": "East US",
    "webSpace": "eastuswebspace"
  },
  {
    "name": "East Asia",
    "webSpace": "eastasiawebspace"
  },
  {
    "name": "Southeast Asia",
    "webSpace": "southeastasiawebspace"
  },
  {
    "name": "West US",
    "webSpace": "westuswebspace"
  },
  {
    "name": "Central US",
    "webSpace": "centraluswebspace"
  },
  {
    "name": "Japan West",
    "webSpace": "japanwestwebspace"
  },
  {
    "name": "Japan East",
    "webSpace": "japaneastwebspace"
  },
  {
    "name": "East US 2",
    "webSpace": "eastus2webspace"
  },
  {
    "name": "Brazil South",
    "webSpace": "brazilsouthwebspace"
  }
]
```

## CLIとURIの対応表

とりえあず自分が使用すると思うリソースと、CLIでだいたいまともに動いてる感じのリクエストをまとめました。全部じゃないけど。

| azure cli | method/rescode | 対応するURI |一言|
|----|----|----|----|
|site location list| GET(200) | /services/WebSpaces| - |
|site list | GET(200) | /services/WebSpaces/{webSpaceName}/sites | - |
|site create [name]|POST(200)| /services/WebSpaces/{webSpaceName}/sites| Body例あり。 |
|site show [name]|*GET(200)| - | config,instanceidsなどの複合リクエストでした。 |
|site showの途中で呼ばれているinstance_id | GET(200)|/services/WebSpaces/{webSpaceName}/sites/{SiteName}/instanceids |CLIでは直接これだけ叩くコマンドはない。  |
|site stop [name]|PUT(200)| /services/WebSpaces/{webSpaceName}/sites/{SiteName} |Body例あり。  |
|site domain list [name] | GET(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName} | - |
|site domain add [dn] [name] | PUT(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName} |Body例あり、事前にCNAME登録必須。 |
|site domain delete [dn] {name} | PUT(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName} |Body例あり。 |
|site appsetting list [name]|GET(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName}/config | JSONで返ってくる。|
|site appsetting show <key> [name] | GET(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName}/config |すべてとってCLI側でパースしていた。リクエストとしてはlistと同じ。 |
|site appsetting add <keyvaluepair> [name] | PUT(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName}/config |Body例あり、JSONだ。  |
|site appsetting delete <key> [name] | PUT(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName}/config |Body例あり、JSON。リクエストとしてはaddと同じ。  |
|site config *| - | - |古い、appsetting推奨なのでカット。 |
|site scale mode <mode> [name]| PUT(200) | /services/WebSpaces/{webSpaceName}/ServerFarms/{ServerFarmId} |Body例あり、ServerFarmIdはsiteをGETした際の情報に含まれている。 (`Default4`など) |
|site scale instances <instances> [name] | - | - | 成功してない。 |
|site defaultdocument list [name] |GET(200) |/services/WebSpaces/{webSpaceName}/sites/{SiteName}/config |JSONで返ってくる。リクエストとしてはappsetting listと同じ。 |
|site defaultdocument add <document> [name] | PUT(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName}/config |Body例あり、JSON。 |
|site defaultdocument delete <document> [name] | PUT(200) | /services/WebSpaces/{webSpaceName}/sites/{SiteName}/config  |Body例あり、JSON。リクエストとしてはaddと同じ。 |



### SCM関連のエンドポイントとCLI対応

SCM関連は普通のAPIエンドポイントに送るリクエストとサイト専用のSCMエンドポイントにリクエストが分かれます。

| azure cli | method/rescode | 対応するURI |一言|
|----|----|----|----|
|site deployment github [name]|POST(200) |/services/WebSpaces/{webSpaceName}/sites/{SiteName}/repository |POSTのBODYが空で投げられて、サイト側にベアリポジトリを作成する。。いわゆる`ローカルGitリポジトリ`で作られる設定。コマンド名となんか違う。 |
|site deployment user set [username] [pass]|PUT(??) |/services/WebSpaces?properties=publishingCredentials |検証していたアカウントがco-adminsだったので400で蹴られた。あまりAPIからつつくものでは無いかも。  |


一部のリクエストでサイトごとに割り振られたSCMエンドポイント(`{SiteName}.scm.azurewebsites.net` )が使用される。
hogehoge31なら`hogehoge31.scm.azurewebsites.net`だった。

次のリクエストはSCMエンドポイントに送る。


| azure cli | method/rescode | 対応するURI |一言|
|----|----|----|----|
|site deployment list [name]|GET(200)|/api/deployments/ | JSONのアレイで返ってくる。コミットIDを含む。 |
|site deployment redeploy [options] <commitId> [name] |PUT(204) |/api/deployments/{commit_id(sha1)} |BODYは空。任意のコミットIDをデプロイできる。これ204。  |
|site log tail [name]|GET(200)|/logstream |ログがストリーミングで渡ってくる。 |
|site log download [name]|GET(200)|/dump |CLIではdiagnostics.zip一式としてダウンロード。 |


## POST/PUT時のリクエスト集

以下、Bodyのサンプル。

### site create

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Site xmlns="http://schemas.microsoft.com/windowsazure">
  <Name>hogehoge31</Name>
  <ServerFarm/>
  <WebSpaceToCreate>
    <GeoRegion>Japan East</GeoRegion>
    <Name>japaneastwebspace</Name>
    <Plan>VirtualDedicatedPlan</Plan>
  </WebSpaceToCreate>
</Site>
```

### site stop/start

サイト`hogehoge31`をstopした時のリクエスト。

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Site xmlns="http://schemas.microsoft.com/windowsazure">
  <State>Stopped</State>
</Site>
```

Startの場合は`Running`。CLIのrestartはStoppedとRunningを続けて送信する。

### site domain add

hogehoge31に`hogehoge31.higan.works`を付与した時。

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Site xmlns="http://schemas.microsoft.com/windowsazure" xmlns:a="http://schemas.microsoft.com/2003/10/Serialization/Arrays">
  <HostNames>
    <a:string>hogehoge31.azurewebsites.net</a:string>
    <a:string>hogehoge31.higan.works</a:string>
  </HostNames>
</Site>
```

### site domain delete

hogehoge31から`hogehoge31.higan.works`を削除した時。

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<Site xmlns="http://schemas.microsoft.com/windowsazure" xmlns:a="http://schemas.microsoft.com/2003/10/Serialization/Arrays">
  <HostNames>
    <a:string>hogehoge31.azurewebsites.net</a:string>
  </HostNames>
</Site>
```

### site appsetting add/delete

`setting add MYCONF="HOGEHOGE" hogehoge31` とした時のリクエスト。delete時は該当のキーバリューを削除して送る。


```
{
   "DefaultDocuments" : [
      "Default.htm",
      "Default.html",
      "Default.asp",
      "index.htm",
      "index.html",
      "iisstart.htm",
      "default.aspx",
      "index.php",
      "hostingstart.html"
   ],
   "RoutingRules" : [],
   "LogsDirectorySizeLimit" : 35,
   "DetailedErrorLoggingEnabled" : false,
   "ConnectionStrings" : [],
   "AlwaysOn" : false,
   "WebSocketsEnabled" : false,
   "NumberOfWorkers" : 1,
   "PhpVersion" : "5.4",
   "NetFrameworkVersion" : "v4.0",
   "HttpLoggingEnabled" : false,
   "HandlerMappings" : [],
   "AppSettings" : [
      {
         "Name" : "WEBSITE_NODE_DEFAULT_VERSION",
         "Value" : "0.10.32"
      },
      {
         "Value" : "HOGEHOGE",
         "Name" : "MYCONF"
      }
   ],
   "RemoteDebuggingEnabled" : false,
   "SiteAuthEnabled" : false,
   "RequestTracingEnabled" : false,
   "Metadata" : [],
   "ScmType" : "None",
   "ManagedPipelineMode" : 0,
   "Use32BitWorkerProcess" : true
}
```


### site scale mode

site `hogehoge31`をbasicにした時のリクエスト。

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<ServerFarm xmlns="http://schemas.microsoft.com/windowsazure">
  <SKU>basic</SKU>
</ServerFarm>
```


### site defaultdocument delete/add

hogehoge31からdefault.aspxを取り除いたリクエスト例。

```
{
   "DetailedErrorLoggingEnabled" : false,
   "WebSocketsEnabled" : false,
   "HandlerMappings" : [],
   "SiteAuthEnabled" : false,
   "DefaultDocuments" : [
      "Default.htm",
      "Default.html",
      "Default.asp",
      "index.htm",
      "index.html",
      "iisstart.htm",
      "index.php",
      "hostingstart.html"
   ],
   "NumberOfWorkers" : 1,
   "LogsDirectorySizeLimit" : 35,
   "HttpLoggingEnabled" : false,
   "RoutingRules" : [],
   "RemoteDebuggingEnabled" : false,
   "Use32BitWorkerProcess" : true,
   "ManagedPipelineMode" : 0,
   "RequestTracingEnabled" : false,
   "AppSettings" : [
      {
         "Name" : "WEBSITE_NODE_DEFAULT_VERSION",
         "Value" : "0.10.32"
      }
   ],
   "PhpVersion" : "5.4",
   "ConnectionStrings" : [],
   "RemoteDebuggingVersion" : "VS2012",
   "NetFrameworkVersion" : "v4.0",
   "Metadata" : [],
   "ScmType" : "None",
   "AlwaysOn" : false
}
```

### site deployment user set

`user: sawaploy`, `pass: yourpassword`で共通デプロイユーザを作るときのXML。
これは管理者でしか作れないのと、全サイトにまたがる設定になるので最初にポータルから作れば十分。

```
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<User xmlns="http://schemas.microsoft.com/windowsazure">
  <PublishingPassword>yourpassword</PublishingPassword>
  <PublishingUserName>sawaploy</PublishingUserName>
</User>
```


## CLI単独では直接取れないリソースのレスポンス

CLIには対応するサブコマンドが無いんだけども、`site config`などの複合出力が読んでいるのを見つけた。とりあえず１つ。

### ...{SiteName}/instanceids

Appのホスティングに割り当てられているインスタンスのIDが取れた。

```
<ArrayOfstring xmlns="http://schemas.microsoft.com/2003/10/Serialization/Arrays" xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
  <string>c316b60c7926f2ebXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX</string>
  <string>1da71d92d8f9da7dXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXc</string>
</ArrayOfstring>
```


