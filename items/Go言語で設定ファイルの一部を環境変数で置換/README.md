

Go製ツール[walter](https://github.com/walter-cd/walter "walter-cd/walter")の設定ファイル`pipeline.yml`で値を環境変数から取りたいなあと思ったのでトライ。

とりあえず"text/template"で環境変数を使える体まで書いたのでメモ。


## まず環境変数をまるごとマップに

Goでは環境変数の一覧を`os.Environ()`で取れるんだけど、キーバリュー込みで1つの単位になるテキストのスライスをもらえるだけなので、そのままでは扱いづらい。

じゃあまずマップに変換してみようと。

[github.com/higanworks/envmap](https://github.com/higanworks/envmap)

参考にしたのはこちらの記事 [Get environment variables as a map in golang - coderwall](https://coderwall.com/p/kjuyqw/get-environment-variables-as-a-map-in-golang "coderwall.com : establishing geek cred since 1305712800")

ライブラリの内容はこちら。

```
package envmap

import (
	"os"
	"strings"
)

// 全部入りのマップを返す
func All() map[string]string {
	data := os.Environ()
	items := make(map[string]string)
	for _, val := range data {
		splits := strings.SplitN(val, "=", 2)  // `=`で分割、分割数を2にして値側の`=`を無視
		key := splits[0]
		value := splits[1]
		items[key] = value
	}
	return items
}

// キーの一覧を返す
func ListKeys() []string {
	data := os.Environ()
	var keys []string
	for _, val := range data {
		splits := strings.SplitN(val, "=", 2)  // `=`で分割、分割数を2にして値側の`=`を無視
		keys = append(keys, splits[0])
	}
	return keys
}
```

事前の仕込みはこれでよし。


## yamlの文字列を`text/template`で処理

変換したいところをヒゲで囲んだ`hoge.yml`を用意した。今回はドットのあとに代入したい環境変数。

```hoge.yml
message:
 hoge: Hello {{.HOME}}
```

で、[template - The Go Programming Language](http://golang.org/pkg/text/template/ "template - The Go Programming Language")を見ながら処理を書いた。


```main.go
package main

import (
	"github.com/higanworks/envmap"
	"io/ioutil"
	"os"
	"text/template"
)

func main() {
	envs := envmap.All() // 環境変数マップを取得

	rawdata, err := ioutil.ReadFile("hoge.yml")
	if err != nil {
		panic(err)
	}

	tmpl, err := template.New("test").Parse(string(rawdata))
	if err != nil {
		panic(err)
	}

	err = tmpl.Execute(os.Stdout, envs) // 置換して標準出力へ
	if err != nil {
		panic(err)
	}

}
```

実行して、置換後の出力。

```yaml:hoge.yml(output)
---
message:
 hoge: Hello /Users/sawanoboly
```

できたわー。


ちなみにコードこのままだと、環境変数が無かったら`<no value>`で埋められちゃう。


```yaml:hoge.yml(output_with_no_value)
---
message:
 hoge: Hello <no value>
```

もしかすると"text/template"の`Actions`だけでも何とかなりそうだけど、yaml側に色々書くのも変だし。


## おすすめGo

この2つにお世話になりました。

- [Amazon.co.jp： 基礎からわかる Go言語: 古川 昇: 本](http://www.amazon.co.jp/dp/4863541171 "Amazon.co.jp： 基礎からわかる Go言語: 古川 昇: 本")
- [fatih/vim-go](https://github.com/fatih/vim-go "fatih/vim-go")
