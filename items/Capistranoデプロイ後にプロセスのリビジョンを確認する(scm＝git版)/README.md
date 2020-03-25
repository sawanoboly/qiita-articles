<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Capistranoでデプロイすると、コードを設置した後に新旧コードをシンボリックリンクで入れ替える。
この方式はロールバックが簡単で良いのだが、リスタートしたつもりのプロセスは本当に`Current`のものか気になるので確認する。


## revisionチェックのタスク
scmにgitを指定した際、capistranoは`#{current_path}/REVISION`にコミットのハッシュを置くので、`/proc`から実行中プロセスのワーキングディレクトリを覗く。


```ruby:deploy.rb
namespace :deploy do
  task :print_revision do
    run "cat #{current_path}/REVISION"
    run "/proc/`cat #{shared_path}/pids/myapp.pid`/cwd/REVISION"
  end
end
```

## 実行サンプル

```
  * executing "cat /opt/deploy/myapp/current/REVISION"
 ** [out :: xx.xx.xx.xx] 7f36e97e7d4fa16b3556b0d2a2c73f86fb108871
  * executing "cat /proc/`cat /opt/deploy/myapp/shared/pids/myapp.pid`/cwd/REVISION"
 ** [out :: xx.xx.xx.xx] 7f36e97e7d4fa16b3556b0d2a2c73f86fb108871
```

ハッシュがあっていればよし。
