
お仕事の関係上、たまにLet's EncryptのAPIエンドポイントである[letsencrypt/boulder](https://github.com/letsencrypt/boulder)のソースを見ます。
組んだシステムがぶっ壊れないよう、事前にstagingへの変更を確認したり軽く結合テストをする目的ですね。

新機能フラグとかは、なるべくONでーという感じでコンフィグをみたら。

```shell:
$ cat test/config-next/ca.json  | jq .ca.features
{
  "WildcardDomains": true,
  "AllowTLS02Challenges": true,
  "GenerateOCSPEarly": true
}
```

おや、`WildcardDomains`フラグですってよ。

この調査をした時点の[stagingブランチ(521e27...)](https://github.com/letsencrypt/boulder/tree/521e27acb383fb792e3ebc90be7e4d97c9483974)をベースに、軽くどのように展開されるのか見てみました。

## 先に見通しの結論

実際リリースの時にこうなるかは保証しませんよ。

- Authorization(authz)の`domain`に`*.`で始まるドメインを渡せばワイルドカード判定
  - 内部的にWildcardフラグが立つ
  - `*.`を除いた部分がチャレンジ対象として扱われる
  - `*.`を除いた部分でこれまでと同じポリシー([PublicSuffix](https://qiita.com/sawanoboly/items/8e3c57aa2e30fc58c4e3#制限のかかるドメインの単位---public-suffix)など)判定
- チャレンジ方式は`dns-01`のみ

となるっぽい。


## ソース

興味があったり、誤読を見つけたい方はこの先にどうぞ。

前述のとおり[stagingブランチ(521e27...)](https://github.com/letsencrypt/boulder/tree/521e27acb383fb792e3ebc90be7e4d97c9483974)を見た感想です。ついでに言うと、まだフラグだけでは作れない模様(ACME v2実装部分のwfe2に行かない)。

まずWebFrontEnd(wfe)のテストから。 [boulder/wfe2/wfe_test.go](https://github.com/letsencrypt/boulder/blob/521e27acb383fb792e3ebc90be7e4d97c9483974/wfe2/wfe_test.go#L2399-L2423)


```go:boulder/wfe2/wfe_test.go
func TestPrepAuthzForDisplayWildcard(t *testing.T) {
	wfe, _ := setupWFE(t)

	// Make an authz for a wildcard identifier
	authz := &core.Authorization{
		ID:             "12345",
		Status:         core.StatusPending,
		RegistrationID: 1,
		Identifier:     core.AcmeIdentifier{Type: "dns", Value: "*.example.com"},
		Challenges: []core.Challenge{
			{
				ID:   12345,
				Type: "dns",
			},
		},
	}

	// Prep the wildcard authz for display
	wfe.prepAuthorizationForDisplay(&http.Request{Host: "localhost"}, authz)

	// The authz should not have a wildcard prefix in the identifier value
	test.AssertEquals(t, strings.HasPrefix(authz.Identifier.Value, "*."), false)
	// The authz should be marked as corresponding to a wildcard name
	test.AssertEquals(t, authz.Wildcard, true)
}
```

APIにリクエストする際、ドメイン部分に`"*.example.com"`を指定すると、(Authorizationの受け口から)[`prepAuthorizationForDisplay`](https://github.com/letsencrypt/boulder/blob/521e27acb383fb792e3ebc90be7e4d97c9483974/wfe2/wfe.go#L848-L869) によって `*.`が除去されて、Wildcardフラグを立てているのがわかりました。


このあと色々追いかけてあれこれ実装をみた([ValidationAuthority](https://github.com/letsencrypt/boulder/blob/521e27acb383fb792e3ebc90be7e4d97c9483974/va/va.go#L882-L886)とか、[Only provide a DNS-01-Wildcard challenge](https://github.com/letsencrypt/boulder/blob/521e27acb383fb792e3ebc90be7e4d97c9483974/policy/pa.go#L318-L372)の記述とか)のですが、結局ポリシーのテストがわかりやすかったので端折ります => [boulder/policy/pa_test.go](https://github.com/letsencrypt/boulder/blob/521e27acb383fb792e3ebc90be7e4d97c9483974/policy/pa_test.go#L201-L273521e27acb383fb792e3ebc90be7e4d97c9483974/policy/pa_test.go#L201-L273)

CSRで証明書をオーダーする途中でチェックしています。


```go:boulder/policy/pa_test.go
func TestWillingToIssueWildcard(t *testing.T) {
	bannedDomains := []string{
		`zombo.gov.us`,
	}
	pa := paImpl(t)

	bannedBytes, err := json.Marshal(blacklistJSON{
		Blacklist: bannedDomains,
	})
	test.AssertNotError(t, err, "Couldn't serialize banned list")
	f, _ := ioutil.TempFile("", "test-wildcard-banlist.txt")
	defer os.Remove(f.Name())
	err = ioutil.WriteFile(f.Name(), bannedBytes, 0640)
	test.AssertNotError(t, err, "Couldn't write serialized banned list to file")
	err = pa.SetHostnamePolicyFile(f.Name())
	test.AssertNotError(t, err, "Couldn't load policy contents from file")

	makeDNSIdent := func(domain string) core.AcmeIdentifier {
		return core.AcmeIdentifier{
			Type:  core.IdentifierDNS,
			Value: domain,
		}
	}

	testCases := []struct {
		Name        string
		Ident       core.AcmeIdentifier
		ExpectedErr error
	}{
		{
			Name:        "Non-DNS identifier",
			Ident:       core.AcmeIdentifier{Type: "nickname", Value: "cpu"},
			ExpectedErr: errInvalidIdentifier,
		},
		{
			Name:        "Too many wildcards",
			Ident:       makeDNSIdent("ok.*.whatever.*.example.com"),
			ExpectedErr: errTooManyWildcards,
		},
		{
			Name:        "Misplaced wildcard",
			Ident:       makeDNSIdent("ok.*.whatever.example.com"),
			ExpectedErr: errMalformedWildcard,
		},
		{
			Name:        "Missing ICANN TLD",
			Ident:       makeDNSIdent("*.ok.madeup"),
			ExpectedErr: errNonPublic,
		},
		{
			Name:        "Wildcard for ICANN TLD",
			Ident:       makeDNSIdent("*.com"),
			ExpectedErr: errICANNTLDWildcard,
		},
		{
			Name:        "Forbidden base domain",
			Ident:       makeDNSIdent("*.zombo.gov.us"),
			ExpectedErr: errBlacklisted,
		},
		{
			Name:        "Valid wildcard domain",
			Ident:       makeDNSIdent("*.everything.is.possible.at.zombo.com"),
			ExpectedErr: nil,
		},
	}

	for _, tc := range testCases {
		t.Run(tc.Name, func(t *testing.T) {
			result := pa.WillingToIssueWildcard(tc.Ident)
			test.AssertEquals(t, result, tc.ExpectedErr)
		})
	}
}
```

Non-DNSは全てダメ、かつ`Valid〜`のケースだけワイルドカード発行OK、ということですね。

## おわりに

ドメイン所有=認証基盤利用可能、はそもそも初期からあってもよい組み合わせだったのではないかなあ。と思いつつ、これでサーバ認証には事欠かなくなりそうです。
