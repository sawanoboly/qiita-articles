
DNSのプロバイダがオプションで提供するようなフェイルオーバーはお値段がそこそこする。
今回とりあえずなんらかのClockWorkで自前で回せるようにする理由があった。

各種CLIとShellでやれば小一時間でできるような話だが、勉強がてら単品バイナリを目指してGo言語でやってみた。

目標は環境変数に必要なパラメータをセットして、`healthbased-dnsrr dnsmple`, `healthbased-dnsrr route53`な感じでよろしくやる。


## ゆるい挙動

- ホストリストをHTTPでそれぞれチェック
- 200を返すホストだけAレコードの候補にして、
    - レコードがなければ登録
    - 差分があれば修正
- 200を返すホストがなければ
    - レコードを削除しておく
    - バックアップIPをAレコードにセット(未実装)

こんなもんでいいか。

## 選べるプロバイダ戦略

DNSを使わせてもらう所はDNSimpleとRoute 53。最終的にどちらにするか決めてないので両方対応しておくことにした。

ベタ書きでDNSimpleを書いた所で、まるでGoらしくないような気がしたのでインターフェースを使ってみようと書き直し。
DnsManage型をつくって、`healthBased_rr`というメソッドをもってれば何でもいいという感じにmainを作成。
道中が`fmt.Printf`だらけだったり、色々と汚いのは今日は諦めよう。

```go-healthbased-dnsrr/main.go
package main

import (
	"fmt"
	envmap "github.com/higanworks/go-envmap"
	"os"
	"strings"
)

var envs = envmap.All()
var targetHosts = strings.Split(envs["DNS_TARGET_HOSTS"], ",")
var backupHosts = strings.Split(envs["DNS_BACKUP_HOSTS"], ",")

type DnsManage interface {
	healthBased_rr() int
	//	enableBackup() error
}

func main() {
	os.Exit(realMain())
}

func realMain() int {
	var manager DnsManage

	if len(os.Args) < 2 {
		fmt.Printf("Usage: healthbased-dnsrr [dnsimple|route53]\n")
		return 1
	}

	switch os.Args[1] {
	default:
		fmt.Printf("Usage: healthbased-dnsrr [dnsimple|route53]\n")
		return 1
	case "dnsimple":
		manager = &DnsimpleManage{
			TargetHosts:      targetHosts,
			BackupHosts:      backupHosts,
			TargetRR:         envs["DNS_TARGET_RR"],
			targetDomainName: envs["DNS_TARGET_DOMAIN"],
			apiToken:         envs["DNSIMPLE_TOKEN"],
			email:            envs["DNSIMPLE_USER"],
		}
	case "route53":
		manager = &Route53Manage{
			TargetHosts:      targetHosts,
			BackupHosts:      backupHosts,
			TargetRR:         envs["DNS_TARGET_RR"],
			targetDomainName: envs["DNS_TARGET_DOMAIN"],
		}
	}

	life := manager.healthBased_rr()
	if life == 0 {
		fmt.Printf("All Dead\n")
		//	manager.enableBackup()
	}

	return 0
}
```

引数からmanagerを選択し、あとは同じメソッドで同じことをやればいいはず。。
Goの本ではゴルーチンやチャネルがかっこよさそうに見えた、しかし今日の理解度では無理なのでスルー。

開発中の環境変数はdirenvで入れておくようにした。

```shell:.envrc
.envrc 
layout go
export DNSIMPLE_TOKEN='*********************'
export DNSIMPLE_USER='*********************@higanworks.com'

export DNS_TARGET_DOMAIN='example.net'
export DNS_TARGET_RR='wwwww'
export DNS_TARGET_HOSTS='210.152.xxx.xx1,210.152.xxx.xx2'
export DNS_BACKUP_HOSTS='210.152.xxx.xx3'

export AWS_ACCESS_KEY='*********************'
export AWS_SECRET_ACCESS_KEY='*********************'
```


### DNSimple用の処理

DNSimpleはAレコードを愚直に付け外しするだけなので、ドメインIDを取得したらホストの死活結果を個別に反映させればよかった。

DnsimpleManage型が進むごとにじゃんじゃん太っていった。後でテストを書けば減らせそうだ。


```go-healthbased-dnsrr/dnsmple.go
package main

import (
	"fmt"
	dnsimple "github.com/rubyist/go-dnsimple"
	"os"
)

type DnsimpleManage struct {
	client            *dnsimple.DNSimpleClient
	TargetHosts       []string
	BackupHosts       []string
	TargetRR          string
	targetDomainName  string
	targetDomainId    int
	countAlivedTarget int
	apiToken          string
	email             string
}

func (dm *DnsimpleManage) RecordExists(records []dnsimple.Record, host string) int {
	for _, val := range records {
		if val.Name != dm.TargetRR {
			continue
		}
		if val.Content == host {
			fmt.Printf("Record: %s -> %s\n", val.Name, val.Content)
			return val.Id
		}
	}
	return 0
}

func (dm *DnsimpleManage) getDomainId() int {
	domains, _ := dm.client.Domains()
	for _, domain := range domains {
		if domain.Name == dm.targetDomainName {
			fmt.Printf("Domain: %s %d\n", domain.Name, domain.Id)
			return domain.Id
		}
	}
	return 0
}

func (dm *DnsimpleManage) healthBased_rr() int {
	dm.countAlivedTarget = len(dm.TargetHosts)
	dm.client = dnsimple.NewClient(dm.apiToken, dm.email)

	// Get a list of your domains
	dm.targetDomainId = dm.getDomainId()

	if dm.targetDomainId == 0 {
		fmt.Printf("Exit: Target Domain Not found.\n")
		os.Exit(1)
	}

	//	 Get a list of records for a domain
	records, _ := dm.client.Records(dm.targetDomainName, "", "")

	for _, val := range dm.TargetHosts {
		r_id := dm.RecordExists(records, val)
		status, err := tryHttp(val)
		if err != nil {
			if r_id != 0 {
				fmt.Printf("Dead But Exists!: I'll delete [ %d ]. \n", r_id)
				delRec := dnsimple.Record{Id: r_id, DomainId: dm.targetDomainId}
				err := delRec.Delete(dm.client)
				if err != nil {
					fmt.Printf("Delete returned error: %v\n", err)
				}
			} else {
				fmt.Printf("Result: Dead\n")
			}
			dm.countAlivedTarget--
			continue
		}

		fmt.Printf("Result: %d\n", status)
		// Create a new Record
		//		go func() {
		if r_id == 0 {
			newRec := dnsimple.Record{Name: dm.TargetRR, Content: val, RecordType: "A", TTL: 600}
			rec, _ := dm.client.CreateRecord(dm.targetDomainName, newRec)
			fmt.Printf("RecordID: %d\n", rec.Id)
		} else {
			fmt.Printf("Record Already Exists\n")
		}
		//		}()
	}

	return dm.countAlivedTarget
}

func (dm *DnsimpleManage) enableBackup() int {
	return 0
}
```

DNSimple対応を動かすとこんな感じ。末尾1のホストに到達性が無いので外した時のログ。

```
$ ./build/darwin/amd64/healthbased-dnsrr dnsimple
Domain: example.net 119xxx
Record: wwwww -> 210.152.xxx.xx1
Dead But Exists!: I'll delete [ 39xxxxx ]. 
Record: wwwww -> 210.152.xxx.xx2
Result: 200
Record Already Exists
```


### Route 53用の処理

Route 53は1つのレコードセットに複数のコンテンツを含むので微妙にややこしい。
死活の結果に加えて、現状をとっておいて差分があれば上書きする必要があり、APIコールもバッチ風でけっこうな非同期。UPSERTが無かったらきっとつらい思いをしていた。

今回は共通でドメイン名を使いたかったので、HostedZoneIDの特定にListHostedZonesByNameを使った。
HostedZoneは一度に最大100件取得できるが、対象のアカウントではドメイン200を超えているので全部取ってくるのが面倒だった。

なおRoute 53では同名のドメインを作れるので、実はこのままでは誤爆の可能性はある。すなおにIDを控えたほうがいいんだろう。
ついでに、このままでは`healthBased_rr`が長すぎる。 errがかぶってerr2とか出てきているし。後で関数にしておこう。

```go-healthbased-dnsrr/route53.go
package main

import (
	"fmt"
	"github.com/higanworks/goamz/route53"
	"github.com/mitchellh/goamz/aws"
	"reflect"
	"sort"
	"strings"
)

type Route53Manage struct {
	client            *route53.Route53
	TargetHosts       []string
	currentRRHosts    []string
	updateRRHosts     []string
	BackupHosts       []string
	TargetRR          string
	targetDomainName  string
	targetDomainId    string
	countAlivedTarget int
}

func (dm *Route53Manage) extractDomainId(res_id string) string {
	id := strings.Split(res_id, "/")
	return id[2]
}

func (dm *Route53Manage) RecordExists(records []route53.ResourceRecordSet) route53.ResourceRecordSet {
	fmt.Printf("Finding: " + dm.TargetRR + "." + dm.targetDomainName + ".\n")
	for _, val := range records {
		if val.Name == dm.TargetRR+"."+dm.targetDomainName+"." {
			return val
		} else if val.Name == dm.TargetRR+"."+dm.targetDomainName {
			return val
		}
	}
	return route53.ResourceRecordSet{}
}

func (dm *Route53Manage) getDomainId() string {
	res, _ := dm.client.ListHostedZonesByName(dm.targetDomainName, "", 1)

	for _, zone := range res.HostedZones {
		if zone.Name == dm.targetDomainName+"." {
			fmt.Printf("Found: %s: %s\n", zone.ID, zone.Name)
			return dm.extractDomainId(zone.ID)
		} else if zone.Name == dm.targetDomainName {
			fmt.Printf("Found: %s: %s\n", zone.ID, zone.Name)
			return dm.extractDomainId(zone.ID)
		}
	}
	return "0"
}

func (dm *Route53Manage) healthBased_rr() int {
	dm.countAlivedTarget = len(dm.TargetHosts)

	auth, _ := aws.EnvAuth() // AWS_ACCESS_KEY, AWS_SECRET_ACCESS_KEY
	dm.client = route53.New(auth, aws.USEast)

	// Get a list of your domains
	dm.targetDomainId = dm.getDomainId()

	res, err := dm.client.ListResourceRecordSets(dm.targetDomainId, nil)
	if err != nil {
		panic(err)
	}

	rrr := dm.RecordExists(res.Records)
	if rrr.Name != "" {
		dm.currentRRHosts = rrr.Records
		fmt.Printf("Current RR record: %q\n", dm.currentRRHosts)
	} else {
		fmt.Printf("Current RR record: Nothing\n")
	}

	for _, host := range dm.TargetHosts {
		status, _ := tryHttp(host)
		if err != nil {
			status = 0
		}
		fmt.Printf("Check HTTP: %s %d\n", host, status)
		if status != 200 {
			dm.countAlivedTarget--
			continue
		}
		dm.updateRRHosts = append(dm.updateRRHosts, host)
	}

	sort.StringSlice(dm.updateRRHosts).Sort()
	sort.StringSlice(dm.currentRRHosts).Sort()
	sort.StringSlice(dm.BackupHosts).Sort()
	fmt.Printf("Avaliable targets: %q\n", dm.updateRRHosts)
	if !reflect.DeepEqual(dm.updateRRHosts, dm.currentRRHosts) {
		var req route53.ChangeResourceRecordSetsRequest
		var req_rr route53.ResourceRecordSet
		var change route53.Change

		if dm.countAlivedTarget != 0 {
			fmt.Printf("Found difference, I'll fix it.\n")

			req_rr.Name = dm.TargetRR + "." + dm.targetDomainName + "."
			req_rr.Type = "A"
			req_rr.TTL = 600
			req_rr.Records = dm.updateRRHosts

			change.Action = "UPSERT"
			change.Record = req_rr

			req.Comment = change.Action + " to " + dm.TargetRR + "." + dm.targetDomainName + "."
			req.Changes = append(req.Changes, change)

			res, err2 := dm.client.ChangeResourceRecordSets(dm.targetDomainId, &req)
			if err2 != nil {
				panic(err)
			}
			fmt.Printf("Change Status: %s\n", res.ChangeInfo.Status)
		} else {
			// Delete if BackupHosts Empty
			if dm.targetDomainId != "0" {
				fmt.Printf("All Hosts dead. I'll delete rrset.\n")

				req_rr.Name = dm.TargetRR + "." + dm.targetDomainName + "."
				req_rr.Type = "A"
				req_rr.TTL = 600
				req_rr.Records = dm.currentRRHosts

				change.Action = "DELETE"
				change.Record = req_rr

				req.Comment = change.Action + " to " + dm.TargetRR + "." + dm.targetDomainName + "."
				req.Changes = append(req.Changes, change)

				res, err2 := dm.client.ChangeResourceRecordSets(dm.targetDomainId, &req)
				if err2 != nil {
					panic(err2)
				}
				fmt.Printf("Change Status: %s\n", res.ChangeInfo.Status)
			}
		}
	}

	return dm.countAlivedTarget
}
```

Route 53対応を動かすとこんな感じ。さっき外れた末尾1のホストが復帰した時のログ。

```shell:
$ ./build/darwin/amd64/healthbased-dnsrr route53
Found: /hostedzone/Z1KM1PIxxxxxxx: example.net.
Finding: wwwww.example.net.
Current RR record: ["210.152.xxx.xx2"]
Check HTTP: 210.152.xxx.xx1 200
Check HTTP: 210.152.xxx.xx2 200
Avaliable targets: ["210.152.xxx.xx1" "210.152.xxx.xx2"]
Found difference, I'll fix it.
Change Status: PENDING
```

ログがプロバイダによってまるで不揃いなのは日付をまたいだからだな。


### HTTPの死活チェック

雑に10秒をリミットでつついた。Pingで見るよりいいだろう。

```go-healthbased-dnsrr/http_checker.go
package main

import (
	"net/http"
	"time"
)

var httpClient = &http.Client{Timeout: time.Duration(10) * time.Second}

func tryHttp(host string) (int, error) {
	resp, err := httpClient.Get("http://" + host + "/")
	if err != nil {
		return 0, err
	}
	defer resp.Body.Close()
	return resp.StatusCode, err
}
```

これだけ書いてなんとか目的はだいたいOK。 ほぼわからない所からだったが、`if err != nil`からの`panic(err)`(後でいくらか削除)を適当にいれておくだけでだいぶ違った。


## MacとLinux用にビルド

Rundeckでの実行を想定したので、Linux用にビルドする。goxを使った。

goxで出力ファイル名を調整してビルド。

```go-healthbased-dnsrr/build.sh
gox -osarch="linux/amd64 darwin/amd64" --output="build/{{.OS}}/{{.Arch}}/healthbased-dnsrr"
```

`build.sh`でこの辺ができる。

```tree
$ tree build
build
├── darwin
│   └── amd64
│       └── healthbased-dnsrr
└── linux
    └── amd64
        └── healthbased-dnsrr

4 directories, 2 files
```

Linuxに単品バイナリを運んだ所、ちゃんと動いた。速いし、何より依存が全くないので配備がとっても楽だ。


