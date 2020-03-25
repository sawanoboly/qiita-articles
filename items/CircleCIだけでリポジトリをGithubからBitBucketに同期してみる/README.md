
普段Githubを使っているがたまにメンテなので、とりあえずのざっくりミラーをBitBucketに作るというタスクをCircieCIのDeploymentにやらせてみた。なんともスクリプトがカオスに。

他に直接デプロイしないという前提で、とりあえず`circle.yml`。

```yaml:circle.yml
deployment:
  all_branches:
    branch: /.*/
    commands:
      - ci/99_sync_bitbucket.sh
  all_tags:
    tag: /.*/
    commands:
      - ci/99_sync_bitbucket.sh
```



## BitBucket側準備

- org/name が同じリポジトリを用意する
- 同期専用のユーザをひとつ用意する
    - ssh_keyにCircleCIのデプロイキーから作った公開鍵をセット
    - リポジトリにはこのユーザを対象にwrite権限を付ける

CircleCIはGithubのリポジトリが公開/非公開にかかわらずGithubにDeployキー(プロジェクト単位のR/Oキー)を登録する。プロジェクト単位でユニークかつ、どうせオーナーにしかわからないので、使い回しました。



## 同期用スクリプト

Gitリポジトリ同期でありがちな次の2点になんとか対応しようと思った。

- force push
- tagの削除、または同名で別Commitへの付け替え

なのでBitBucket側での歴史の一貫性などは無視するということで書いたスクリプトが次。


```shell:99_sync_bitbucket.sh
if [ "${CI}" == "true" ]; then

set -e

cat << EOL >> ~/.ssh/config

Host bitbucket.org
IdentitiesOnly yes
IdentityFile /home/ubuntu/.ssh/id_circleci_github
EOL

if ! git remote | grep -q bitbucket ; then
  git remote add bitbucket `git remote get-url origin | sed 's/github.com/bitbucket.org/'`
fi
git fetch origin --prune
for tag in `git tag` ; do git tag -d $tag ; done
git fetch origin --tags --prune
for branch in `git branch -a | grep remotes/origin | grep -v HEAD | grep -v master` ; do
  git branch --track ${branch#remotes/origin/} $branch || true
done
git push bitbucket --mirror --prune --follow-tags
git push bitbucket --mirror --prune --follow-tags

fi
```

フェッチプルーンフェッチプルーン。継ぎ足しで適当に確認しながら作ったのでどれか無駄かもしれない。

CircleCIは基本的にgitリポジトリをキャッシュするので、tagやらremoteの情報を保持していたり、ブランチによってはしていなかったりする。タグは結局pruneで追えないっぽいので、一回ローカルのを破棄することで対応。

最後のpushは1回めでBitBucket側がさっぱり消えて、2回目でブランチ同期、という挙動になった。これ全く個別のディレクトリで作業するほうがマシかも。


`--mirror`の使い方として合っているのか多少気がかりだが、ブランチ、タグ等々これで追えたようなので、ゆるいミラーとしては一旦これでよし。ただリポジトリのサイズが大きいと時間がかかりそうな気もする。

ブランチ/タグを削除するだけ、というイベントではCIが動かない(tag pushはOK)とか、Deploymentでやるとテストがこけたら(テストがある場合)同期しない点に注意と。


