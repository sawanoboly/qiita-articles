<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

Redmineを好みのRackupを使いたい、プラグインで使うgemを追加したい。

Gemfileに直接書いたらマージが発生するので、ローカル用・プラグイン用のGemfileをインクルード出来るようにしてくれている。


```Ruby:Gemfile(redmine)
-- snip --
local_gemfile = File.join(File.dirname(__FILE__), "Gemfile.local")
if File.exists?(local_gemfile)
  puts "Loading Gemfile.local ..." if $DEBUG # `ruby -d` or `bundle -v`
  instance_eval File.read(local_gemfile)
end

# Load plugins' Gemfiles
Dir.glob File.expand_path("../plugins/*/Gemfile", __FILE__) do |file|
  puts "Loading #{file} ..." if $DEBUG # `ruby -d` or `bundle -v`
  instance_eval File.read(file)
end
```

rack関係や、適当に追加したいユーティリティは`Gemfile.local`に記述。
プラグイン独自のgemは`plugins/{pluginname}/Gemfile`に記述。

という管理をしてbundle。
