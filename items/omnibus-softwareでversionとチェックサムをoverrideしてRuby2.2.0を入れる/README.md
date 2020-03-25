

[chef/omnibus](https://github.com/chef/omnibus)で依存ソフトをビルドする際に、ソースアーカイブのチェックサムを要求するものがある。例えば[Ruby](https://github.com/chef/omnibus-software/blob/master/config/software/ruby.rb)。

バージョン番号だけなら簡単に指定できるけど、チェックサムが必要な場合の書式がなかなか分からなかったので調べた。

[project overrides of software definitions #94](https://github.com/chef/omnibus/pull/94)

上記のIssueで様々な議論がなされたあと、次のような書式にひとまず落ち着いたようだ。


```projects/mysoftware.rb
override :ruby,
version: "2.2.0",
source: {
md5: "cd03b28fd0b555970f5c4fd481700852"
}
dependency "ruby"

override :rubygems, version: "2.4.5"
dependency "rubygems"
override :bundler, version: '1.8.2'
dependency "bundler"
```

変更時の差分を見るとブロックでもいけそうな気がする。
