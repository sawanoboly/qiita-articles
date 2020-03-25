<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Chef-Clientがfailで終了したらロールバックしたいという要望をよくもらう、そもそもChefってそういうもんじゃあないよと思いながらも。

バックアップをとるファイル系など、例外ハンドラでリカバリさせることもできるリソースもあるっちゃあるなとサンプルを書いてみた。


## Failするレシピの例

例えばこんなレシピだと、`tmp/hoge`と`/tmp/piyo`が更新されたあとにChef-ClientがFailするので、すべての定義に対して中途半端にノードが更新された状態がつくれる。

```
file '/tmp/hoge' do
  content lazy {Time.now.to_s + "\n"}
  owner 'root
end

template '/tmp/piyo' do
  local true
  source File.expand_path('../time.erb', __FILE__)
end


## Failするリソース(親パスが無い)
file '/tmp/tmp/tmp/hoge' do
  content 'mission failure...'
end
```

※ レシピ中のテンプレートソース

```erb:time.erb 
<%= Time.now.to_s %>
```


> ちなみに`--why-run`ではこういうFailケースはスルーされる、仕方ないね。
> 
> ```
> $ sudo FAIL=true chef-client -z recipe.rb --why-run
> 
> ...
> 
> Chef Client finished, 3/3 resources would have been updated
> ```



## リカバリサンプルのRecipeはこんな感じ

じゃあ半端な状態のノードになったら、任意のリソースだけChef-Client実行前に一応戻してみよう。

例外ハンドラを記述して、`recipe.rb`はこうなった。


```ruby:recipe.rb
## ファイルのリストアをする例外ハンドラ
class Chef::Handler::RollBacker < ::Chef::Handler
  def report
    run_data = data

    ## 更新済みリソースの列挙、ハンドラ共通処理
    Chef::Log.warn '======= Update Resources are following...'
    run_data[:updated_resources].each.with_index do |r,idx|
      Chef::Log.warn [idx, r.to_s].join(':')
    end

    ## Chef-Clientが例外で終わった時の処理
    if exception
      ## 更新済みのリソースに対して順番に処理する
      run_data[:updated_resources].each do |r|
        case r.resource_name
        ## リソースタイプがファイル系リソースだった場合はリストアを試みる
        when :file, :template, :cookbook_file
          ## backupにfalseがセットされているものはスキップ
          next Chef::Log.warn "==== Skkipped restore #{r.to_s} due to no backup" unless r.backup
          restore_file_from_backup(r)
        end
      end
    end
  end

  def get_last_backup(resource, num = 1)
    ## バックアップ先のディレクトリから指定した世代前のファイルを取得
    prefix = File.join(Chef::Config[:file_backup_path], resource.path)
    Dir.glob(prefix + '**').sort[-num]
  end

  def restore_file_from_backup(r)
    backup_file =  get_last_backup(r)
    ## 初回など、バックアップファイルが存在しないケースではスキップ
    return Chef::Log.warn "==== Skkipped restore #{r.to_s} due to backup file not found" unless backup_file

    Chef::Log.warn "=== Restore: #{r.to_s} from Backup..."
    begin
      ## バックアップパスから元の場所にファイルをコピーする
      FileUtils.cp(backup_file, r.path, {:verbose => true, :preserve => true })
      Chef::Log.warn "=== Restore Success!! #{r.to_s}"
    rescue
      Chef::Log.warn "=== Restore Fail!! #{r.to_s}"
    end
  end
end

## 例外ハンドラの登録
Chef::Config[:report_handlers] << Chef::Handler::RollBacker.new
Chef::Config[:exception_handlers] << Chef::Handler::RollBacker.new


## このへんから普通のレシピ
file '/tmp/hoge' do
  content lazy {Time.now.to_s + "\n"}
  owner ['root', 'fluentd'].sample
  backup false  ## 動作サンプル用にバックアップをfalseにしています。
end

template '/tmp/piyo' do
  local true
  source File.expand_path('../time.erb', __FILE__)
end


## Failするリソース(Parent nothing)
if ENV['FAIL']
  file '/tmp/tmp/tmp/hoge' do
    content 'mission failure...'
  end
end
```

## 実行してみる

ひとまず`tmp/hoge`と`/tmp/piyo`を作るため、Failしないように普通に実行させてみる。


```shell:
$ sudo chef-client -z recipe.rb

Starting Chef Client, version 11.14.6

...

Converging 2 resources
Recipe: @recipe_files::/root/rollbacker/recipe.rb


  * file[/tmp/hoge] action create
    - create new file /tmp/hoge
    - update content in file /tmp/hoge from none to c9b647
    --- /tmp/hoge	2014-09-01 xx:04:18.065927966 +0000
    +++ /tmp/.hoge20140901-29657-1vejvln	2014-09-01 xx:04:18.069927890 +0000
    @@ -1 +1,2 @@
    +2014-09-01 xx:04:18 +0000
    - change owner from '' to 'fluentd'


  * template[/tmp/piyo] action create
    - create new file /tmp/piyo
    - update content in file /tmp/piyo from none to c9b647
    --- /tmp/piyo	2014-09-01 xx:04:18.077927738 +0000
    +++ /tmp/chef-rendered-template20140901-29657-eq7f91	2014-09-01 xx:04:18.101927281 +0000
    @@ -1 +1,2 @@
    +2014-09-01 xx:04:18 +0000

Running handlers:

## ハンドラのレポート
[2014-09-01Txx:04:18+00:00] WARN: ======= Update Resources are following...
[2014-09-01Txx:04:18+00:00] WARN: 0:file[/tmp/hoge]
[2014-09-01Txx:04:18+00:00] WARN: 1:template[/tmp/piyo]
  - Chef::Handler::RollBacker
Running handlers complete
Chef Client finished, 2/2 resources updated in 3.268531754 seconds
```

次にChef-ClientがFailするリソースを含めたレシピを実行した。


```shell:
$ sudo FAIL=true chef-client -z recipe.rb 

Starting Chef Client, version 11.14.6

Converging 3 resources

Recipe: @recipe_files::/root/rollbacker/recipe.rb

  * file[/tmp/hoge] action create
    - update content in file /tmp/hoge from c9b647 to 6317af
    --- /tmp/hoge	2014-09-01 xx:04:18.069927890 +0000
    +++ /tmp/.hoge20140901-30004-m7jw0g	2014-09-01 xx:06:06.975857156 +0000
    @@ -1,2 +1,2 @@
    -2014-09-01 xx:04:18 +0000
    +2014-09-01 xx:06:06 +0000
    - change owner from 'fluentd' to 'root'

  * template[/tmp/piyo] action create
    - update content in file /tmp/piyo from c9b647 to 6317af
    --- /tmp/piyo	2014-09-01 xx:04:18.101927281 +0000
    +++ /tmp/chef-rendered-template20140901-30004-1v8gk6n	2014-09-01 xx:06:06.983857004 +0000
    @@ -1,2 +1,2 @@
    -2014-09-01 xx:04:18 +0000
    +2014-09-01 xx:06:06 +0000

  * file[/tmp/tmp/tmp/hoge] action create
    * Parent directory /tmp/tmp/tmp does not exist.
    ================================================================================
    Error executing action `create` on resource 'file[/tmp/tmp/tmp/hoge]'
    ================================================================================
    
    Chef::Exceptions::EnclosingDirectoryDoesNotExist
    ------------------------------------------------
    Parent directory /tmp/tmp/tmp does not exist.      ## 親パスが無いので失敗した

...    
    

## 例外ハンドラ起動
Running handlers:
[2014-09-01Txx:06:07+00:00] ERROR: Running exception handlers


## Chef-ClientはFailで終わったけど、更新したリソースが２つある
[2014-09-01Txx:06:07+00:00] WARN: ======= Update Resources are following...
[2014-09-01Txx:06:07+00:00] WARN: 0:file[/tmp/hoge]
[2014-09-01Txx:06:07+00:00] WARN: 1:template[/tmp/piyo]

## このへんからリストア処理

### 1つ目はバックアップをとって無いからスキップ
[2014-09-01Txx:06:07+00:00] WARN: ==== Skkipped restore file[/tmp/hoge] due to false is set to backup


### 2つ目はバックアップから新しいのを見繕って戻す
[2014-09-01Txx:06:07+00:00] WARN: === Restore: template[/tmp/piyo] from Backup...
cp -p /root/.chef/local-mode-cache/backup/tmp/piyo.chef-20140901100606.990466 /tmp/piyo
[2014-09-01Txx:06:07+00:00] WARN: === Restore Success!! template[/tmp/piyo]

  - Chef::Handler::RollBacker
Running handlers complete

[2014-09-01Txx:06:07+00:00] ERROR: Exception handlers complete
[2014-09-01Txx:06:07+00:00] FATAL: Stacktrace dumped to /root/.chef/local-mode-cache/cache/chef-stacktrace.out
Chef Client failed. 2 resources updated in 3.268363143 seconds
[2014-09-01Txx:06:07+00:00] ERROR: file[/tmp/tmp/tmp/hoge] (@recipe_files::/root/rollbacker/recipe.rb line 58) had an error: Chef::Exceptions::EnclosingDirectoryDoesNotExist: Parent directory /tmp/tmp/tmp does not exist.
[2014-09-01Txx:06:07+00:00] FATAL: Chef::Exceptions::ChildConvergeError: Chef run process exited unsuccessfully (exit code 1)
```

## リストアされたかチェック

`/tmp/piyo`は次の記述どおりなら`2014-09-01 xx:06:06 +0000`という中身に変わっているはず。
ただ、ファイルが置き換えられる際にバックアップが取られている。

```
  * template[/tmp/piyo] action create
...
    -2014-09-01 xx:04:18 +0000
    +2014-09-01 xx:06:06 +0000
```

例外ハンドラによって、バックアップからとりあえず一番新しいっぽいファイルを元の場所に置いたので元通りにはなっている。

```
$ cat /tmp/piyo 
2014-09-01 xx:04:18 +0000
```

場合によってはファイル系リソースと一緒に`service`あたりを拾って適切な処理を書けば十分ロールバックになる気もする。packegeなど、その他のリソースはそれこそ混乱のタネになるので気にしないほうがよさそう。

そういえば、同じファイルパスに対するアクションが２回以上ある場合は、その回数前のバックアップを取らないといけないね。


## ひとこと


Chef-Containerなどでコンテナ単位が対象とかなら、例えば例外ハンドラ内で前回のコンテナをデプロイし直すとかすればリカバリできるので例外ハンドラの処理も結構やりやすいのかもね。
