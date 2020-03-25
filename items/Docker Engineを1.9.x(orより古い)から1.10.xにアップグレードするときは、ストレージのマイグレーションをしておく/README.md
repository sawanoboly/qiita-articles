
まるっと破棄して作りなおす場合には問題ないが、更新をする場合は事前にマイグレーションしておいたほうがよいみたい。

> このマイグレーションなしでも、更新後の起動時に自動実行されるようです。ダウンタイムを節約したい場合にこの手順が有効。

インストーラで更新しようとしたら、警告。

```
If you installed the current Docker package using this script and are using it
again to update Docker, we urge you to migrate your image store before upgrading
to v1.10+.

You can find instructions for this here:
https://github.com/docker/docker/wiki/Engine-v1.10.0-content-addressability-migration
```

こちらを参照しろとのことだ。あーそんな感じのこと、ブログで言うてたな。。

https://github.com/docker/docker/wiki/Engine-v1.10.0-content-addressability-migration


マイグレーションツールもDockerで動かせる、更新前に起動中のDocker1.9系を使い、マイグレーションを行なう。

```
docker run --rm -v /var/lib/docker:/var/lib/docker docker/v1.10-migrator
```

その後、1.10系に更新すればOK。
