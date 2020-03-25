
Railsはいくつかの値を設定ファイルだけでなく、環境変数としても指定できる。
設定ファイルが必須でないなら、アプリコンテナとかも作りやすくなって嬉しい。

そもそも環境設定が.rbだし、yamlもERBで処理されるのでいかようにでもできるとはいえ、`DATABASE_URL`なんかは便利だ。

使えるものをざっくり書き出しておくだけでも意外な発見があるかもしれんと、せっかくだから抽出してみた。
4-2-stable(63a80998e34b314ce0e96faaace3895fe35f4a8c) が対象。

調べ方は`ag 'ENV\[' 対象ディレクトリ/lib`で引っかかったのを列挙、上書きするためでは無いものや、generate関連、doc出力関係などはスルー。

思ったよりは少なかったのと、だいたいがgenerate後のテンプレートに使い方を案内されているので、意外な発見は無かった。
列挙した分、自分らでつかうキーが被らないようにするくらいには役に立つかも。


## railtiesで見かけた環境変数

まずrailties。お役立ち系はだいたいここに。

|環境変数|サンプル値|ファイル(1つとは限らない)|寸評|
|----|----|----|----|
|DATABASE_URL |`'postgresql://foo:bar@localhost:9000/foo_test?pool=5&timeout=3000'`, `sqlite3:#{database_url_db_name}`など  |railties/lib/rails/application/configuration.rb | database.ymlめんどいのでかなり頼れる。 |
|SECRET_KEY_BASE | hexの文字列 |railties/lib/rails/application.rb | productionであると便利。 |
|RAILS_RELATIVE_URL_ROOT |'/subdir'など |railties/lib/rails/application/configuration.rb |サブディレクトリを指定できる。  |
|RAILS_ENV |'production', 'test'など |railties/lib/rails/commands/runner.rb |説明不要。 |
|RACK_ENV |'production', 'test'など |railties/lib/rails/commands/runner.rb | RAILS_ENVが無い時にRACK_ENVがあれば使われる。 |
|RAILS_SERVE_STATIC_FILES |nilやemptyと、その他何でもを区別 |railties/lib/rails/generators/rails/app/templates/config/environments/production.rb.tt |/public以下をRailsが返すかを切り替えできる。 |
|BUNDLE_GEMFILE |`Gemfile_ruby22`,`Gemfile_edge`など。 |ailties/lib/rails/generators/rails/app/templates/config/boot.rb |Bundlerに実装されている、Gemfileを切り替えしたいときに。 |
|LOGS |`test,development,sidekiq`など |railties/lib/rails/tasks/log.rake |Rakeタスクでログのクリアで対象を絞り込みたい時に使う。 |


## activejobで見かけた環境変数

activejobはバックエンド次第なので、Railsというよりはバックエンド側で環境変数からデータストアのURLを受け取る実装があるものを使える感じ。
なので`test/`下にのみある環境変数ばかり。バックエンドが大抵実装しているので、そっちのソースを確認するのがよさげ。

|環境変数|サンプル値|ファイル(1つとは限らない)|寸評|
|----|----|----|----|
|AJ_ADAPTER |inline delayed_job, sidekiqなど |activejob/Rakefile |activejobのアダプタを指定できそうな気がするけど、使用されているのはテストでのみ。 |
|REDISTOGO_URL|"tcp://127.0.0.1:6379/12" |activejob/test/support/integration/adapters/qu.rb |Herokuでお馴染み。一応Quの設定例として登場。 |
|QUE_DATABASE_URL |'postgres:///active_jobs_que_int_test' |activejob/test/support/integration/adapters/que.rb |Sequelの設定例。 |
|QC_DATABASE_URL |'postgres:///active_jobs_qc_int_test' |activejob/test/support/integration/adapters/queue_classic.rb |QC(QueueClassic)の設定例。 |



## activerecordで見かけた環境変数

たいていRakeタスクで使われるものばっかりだった。ほぼRakeの引数としてしか使わなさそう。

|環境変数|サンプル値|ファイル(1つとは限らない)|寸評|
|----|----|----|----|
|FIXTURES|'hogehoge.yml,piyopiyo.yml'など |activerecord/lib/active_record/railties/databases.rake|ややこしいのでdescより`Load specific fixtures using FIXTURES=x,y. Load from subdirectory in test/fixtures using FIXTURES_DIR=z. Specify an alternative path (eg. spec/fixtures) using FIXTURES_PATH=spec/fixtures.` | 
|FIXTURES_PATH |'test/fixtures'など |activerecord/lib/active_record/tasks/database_tasks.rb |FIXTURESに同じ。環境別テストに。 |
|FIXTURES_DIR | |activerecord/lib/active_record/railties/databases.rake |FIXTURESに同じ。環境別テストに。 |
|SCHEMA | |activerecord/lib/active_record/railties/databases.rake |`db:schema:dump`のファイル名を指定できる。比較とかでつかうのかな？|
|DB_STRUCTURE | |activerecord/lib/active_record/railties/databases.rake |`db:structure:dump`の時のファイル名を指定できる。 |
|STEP |整数 |activerecord/lib/active_record/railties/databases.rake |migrationを rollback/forward する時のステップ数。 |
|VERSION |整数 |activerecord/lib/active_record/railties/databases.rake |migration、 (up/down含む) の目標バージョン。|
|SCOPE |'blog' |activerecord/lib/active_record/tasks/database_tasks.rb |migrationのスコープを絞りたい時につかう。 |
|CHARSET |'utf8'など |activerecord/lib/active_record/tasks/mysql_database_tasks.rb | MySQLのみ。db:create時のDEFAULT_CHARSETを指定できる。 |
|COLLATION |'utf8_unicode_ci' |activerecord/lib/active_record/tasks/mysql_database_tasks.rb |MySQLのみ。db:create時のDEFAULT_COLLATIONを指定できる。 |


## activesupportで見かけた環境変数

RAILS_CACHE_IDは意識しておくと役に立つケースがあるかも。

|環境変数|サンプル値|ファイル(1つとは限らない)|寸評|
|----|----|----|----|
|RAILS_CACHE_ID |'c99' |activesupport/lib/active_support/cache.rb |ActiveSupportのキャッシュ保存先につけるPrefix。バージョン混在を避けたいときに。 |
|RAILS_APP_VERSION |'rails3' |activesupport/lib/active_support/cache.rb | RAILS_CACHE_IDと動作は同じ。RAILS_CACHE_ID とは排他っぽい。|
|TZ |`rake time:zones:all`のなにか。 |activesupport/lib/active_support/time_with_zone.rb |Time.zoneに影響する。 詳しくは [Ruby - Railsと周辺のTimeZone設定を整理する (active_record.default_timezoneの罠) - Qiita](http://qiita.com/joker1007/items/2c277cca5bd50e4cce5e "Ruby - Railsと周辺のTimeZone設定を整理する (active_record.default_timezoneの罠) - Qiita") |


## 未分類やガイドで見つけたもの

他にも色々細々とした環境変数の記述はあったけど、テストのループで使用するものやドキュメント、ガイドラインのフラグくらいだった。

|環境変数|サンプル値|ファイル(1つとは限らない)|寸評|
|----|----|----|----|
|VERBOSE |trueとか文字列 | - |詳しく出力。どっちかって言うとRake用。 |
|CDN_HOST |`config.action_controller.asset_host = ENV['CDN_HOST']` |guides/source/asset_pipeline.md  | Asset PipelineのCDNホスト設定例。直接書くより環境変数から取ろうよ、と。|




