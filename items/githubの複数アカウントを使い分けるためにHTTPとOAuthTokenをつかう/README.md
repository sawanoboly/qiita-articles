<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

とある状況下では、複数のGithubのアカウントが必要になる。

例えば以下の様なケース、

- 普段使うの：個人Githubアカウント
- 特定Projectの開発：管理上の都合(笑)で共通のGithubアカウント

Githubがいくら個人指向の`Organization` & `Team` な作りでも、企業間でのやりとりではそこに合わせられないということも多いでしょう。

そんな時に役立つTips.


## 個人アカウントは`SSH public key`、その他は`OAuthのトークン＋HTTP URI` でいこう

聡明な面倒臭がりなら`ssh-agent`であったり`credential-cache`を導入し、快適なgithubライフをおくっていることでしょう。

ここではみなまで述べませんが、SSH-Keyによるアカウント識別の仕組みやhttpアクセス時のアレコレを考えると、Githubのアカウント使い分けは結構難がある仕組みになっています。

OAuth Tokenを使ったリポジトリClone手法を覚えておくとアカウントを使い分けつつほぼシームレスに全関係者用リポジトリにアクセスすることができます。

[Creating an OAuth token for command-line use](https://help.github.com/articles/creating-an-oauth-token-for-command-line-use)

とりあえず上記を参考に共通アカウント用のTokenを発行しましょう、Roleは`repository`のみで十分です。


## Tokenを使った`via https`なクローン

Tokenを手に入れたら、HTTP認証の文字列としてRemoteURIに組み込みましょう。

    git clone https://{OAuth Token}@github.com/{org or user}/repository.git

または既存の`git repository`にリモートを追加したいとき。

    git remote add upstream https://{OAuth Token}@github.com/{org or user}/repository.git

これらは嬉しいことに`ssh-agent`や`credential-cache`に優先します。

Githubアカウントの使い分けに不便を感じたら是非試してみてください。
