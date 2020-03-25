
EC2などをプロバイダにした時、2回目以降の同期対象がおかしいアレ。

[Problem reloading Chef shared folders · Issue #5199 · mitchellh/vagrant](https://github.com/mitchellh/vagrant/issues/5199)


で、とりあえずの回避策。
`.vagrant/machines/default/aws/synced_folders`ファイルがなければ毎回rsyncに行くので、Vagrantfileで消す。

```Vagrantfile

# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

if Vagrant::VERSION == "1.7.2"
  synced_folder_files = Dir.glob(File.expand_path("../.vagrant/machines/**/synced_folders", __FILE__))
  synced_folder_files.map do |synced_folder_file|
    File.unlink(synced_folder_file) if File.exists?(synced_folder_file)
  end
end

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

## -- ship --
```

vagrantコマンドを何かしら実行するたびに`synced_folders`ファイルが消えちゃうけど、実害はないのでいいっしょ。
