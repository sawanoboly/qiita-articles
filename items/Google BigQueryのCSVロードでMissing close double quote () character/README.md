
S3にCSVファイルが置かれたら、ファイルをBigQueryにアップロードするタスクを仕掛けていたら、ある日を境にこけ始めた。

```
$ bq load --skip_leading_rows=1 --replace --sync --project_id mypj ...[以下省略]
Upload complete.
Waiting on bqjob_r42fb5f95fa2e4847_0000015a213d6a98_1 ... (89s) Current status: DONE   

BigQuery error in load operation: Error processing job 'amimoto-dashboard:bqjob_r42fb5f95fa2e4847_0000015a213d6a98_1': Too many errors encountered.
Failure details:
- /gzip/subrange/file-00000000: Error detected while parsing row
starting at position: 250878765. Error: Missing close double quote
(") character.
```

`Missing close double quote`。なんでやねんと思って調査。

## 改行。

csvファイルの行数は`1,117,796`、pandasでロードしたらレコード数は`1,117,768`。。。??

で、あるカラムに改行を発見。

```
"foobarstring\n"
```

なるほど。


## `--allow_quoted_newlines`をつけて対応できるケースだった

```
$ bq load --skip_leading_rows=1 --allow_quoted_newlines --replace --sync --project_id mypj ...[以下省略]

Upload complete.
Waiting on bqjob_r1680e9a9697238aa_0000015a214b406d_1 ... (168s) Current status: DONE  
```

ちなみに改行が混入したのはAWSの請求データで、`user:Name`とされているカラムでした。リソースタグに`\n`とか入れたら請求のCSVに記述されて、どこかで何かがこけちゃうぞ！
