<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。


この記事は[CDP Advent Calendar 2013](http://www.zusaar.com/event/1120006)の12/8分です。


AWS CloudFormationを使ったWordPressの環境構築サービス[JIN-KEI](http://ja.cloudhappy.net/)というのがあります。
それの3つ目の陣形として、CDPから何かやってみてよと頼まれたので、台数・見た目重視でNFS Sharing PatternとAutoScalingを組み合わせてみました。

次のような構成(一部)になっています。

![03_jinkei.png](https://qiita-image-store.s3.amazonaws.com/0/7454/35d42c36-2364-d8e5-607c-a76fab83b777.png "03_jinkei.png")

軽く説明しておくと、

- WordPressのファイルをNFSサーバで共有
- AutoScalling Groupはマウント
- MySQLはRDSで

と、月並みなCloudFormationの構成です。
で、楽をするためEIPは使わない、VPCもつくらないことにしました。


## AWS CloudFormation + Chef-SoloでNFSの設定

陣形で使っているマーケットプレースAMIのAmimotoは、cfn-initの情報次第で自分で勝手に役割を決める仕様になっています。
全部同じイメージです。
そしてその役割を決める時にChef-Soloを実行するようなので、新しく役割を定義してRecipeを書きましたっと。

### NFS Server

Server/ClientはどっちもNFSに関係していることを検知したらとりあえず`nfs_dispatcher`レシピを叩きます。
マスタ役のインスタンスIDはcfn-initでjsonに吐いているので適当に拾ってきます。


```Ruby:recipes/nfs_dispatcher.rb
cfn = JSON.load(::File.read('/opt/aws/cloud_formation.json'))

%w(portmap nfs-utils nfs-utils-lib).each do |pkg|
  package pkg do
    action [:install, :upgrade]
  end
end  ## パッケージインストールは共通。

if node[:ec2][:instance_id] == cfn['nfs']['server']['instance-id']  ## 自分がマスタ役のインスタンスIDかチェック
  include_recipe 'amimoto::nfs_server'  ## Server用レシピへ
else
  include_recipe 'amimoto::nfs_client'   ## Client用レシピへ
end
```

サーバ役は`/etc/exports`を作りゃOKです。exportする場所はどうせログインすればわかるのでそのまま乗っけてます。
オプション的には色々と面倒なので、来たやつ全員nginxユーザとして丸め込みます。

```Ruby:recipes/nfs_server.rb
nginx_uid = `id -u nginx`.chomp
nginx_gid = `id -g nginx`.chomp

file '/etc/exports' do
  action :create
  content [
    "/var/www/vhosts/#{node[:ec2][:instance_id]} *(rw,async,no_acl,root_squash,all_squash,anonuid=#{nginx_uid},anongid=#{nginx_gid})",
    "/var/cache/nginx/proxy_cache *(rw,async,no_acl,root_squash,all_squash,anonuid=#{nginx_uid},anongid=#{nginx_gid})"
    ].join("\n")
end  ## Amimoto関連をエクスポート

%w(rpcbind nfslock nfs).each do |svc|
  service svc do
    action [:enable, :start]
    subscribes :restart, 'file[/etc/exports]'  ## file[/etc/exports]に変更があったらとりあえず関連サービスをリスタートする。
  end
end
```

NFSのポートはAutoScalingが使うセキュリティグループに対して許可を入れてます。


### NFS Client

Server役にElastic IPを付与しないので、ClientはServer役のPrivateIPを取得してからマウントします。

なんでElastic IPにしないのかとちょっと思うかもしれませんので理由を。
まずNFSなんでPublic側を通すと遅いし、そもそもElastic IP x EC2の組み合わせが面倒で嫌いなんです。

それはさておき。

CloudFormationでIAMのリソースとしてEC2のdescribeを許可したロールを作成し、AutoScallingのインスタンスにはそれを割りあてています。
これでEC2のインスタンスはアクセスキー等なしでもEC2のdescribeだけできるので、それを使ってNFSマウントします。

cfnの使い回しはdispatcherの時点で`node.run_state`をつかったほうが綺麗になるんですが、単品でも動くようにあえてここでも取っています。

```Ruby:recipes/nfs_client.rb
cfn = JSON.load(::File.read('/opt/aws/cloud_formation.json'))
describe_cmd = "/usr/bin/aws ec2 describe-instances --instance-id #{cfn['nfs']['server']['instance-id']} --region #{node.ec2[:placement_availability_zone].chop} " ## IAM Roleの権限を使ってServer役のインスタンス情報をゲット
masterserver = JSON.load(`#{describe_cmd}`)
master_ip = masterserver['Reservations'].first['Instances'].first["PrivateIpAddress"]

mount "/var/www/vhosts/#{node[:ec2][:instance_id]}" do
  action [:mount]
  device "#{master_ip}:/var/www/vhosts/#{cfn['nfs']['server']['instance-id']}"
  fstype "nfs"
  options "rw"
end  ## マウントする、fstabには入れない

mount "/var/cache/nginx/proxy_cache" do
  action [:mount]
  device "#{master_ip}:/var/cache/nginx/proxy_cache"
  fstype "nfs"
  options "rw"
end  ## マウントする、fstabには入れない


template '/opt/aws/nfs_client.rb' do
  source 'cfn/nfs_client.rb.erb'
end ## Chef-Applyでマウントする為のファイルを設置

rc_line = '/usr/bin/chef-apply /opt/aws/nfs_client.rb > /dev/null 2>&1'
file '/etc/rc.local' do
  _file = Chef::Util::FileEdit.new('/etc/rc.local')
  _file.insert_line_if_no_match('/opt/aws/nfs_client.rb', [rc_line, "\n"].join)
  content _file.send(:contents).join
  manage_symlink_source true
end.run_action(:create)  ## rc.localで起動時にChef-Apply
```

マウントのアクションを`[:mount, :enable]`にすればfstabにも登録されますが、Server役がSTOPするとPrivateIPが変更されるのでfstabに登録しませんでした。

で、毎回起動時にChef-Applyを実行してくれるように、このレシピが読み込まれた時点でrc.localに追記しておくと。
いきなりrun_actionをしているのはChef::Util::FileEditの都合です、将来ほかのレシピで同じファイルを編集するかもしれないのでこうしてます。


## Chef-ApplyでNFSをマウント

Client役は起動時に毎回`/usr/bin/chef-apply /opt/aws/nfs_client.rb`を実行します、その中身がこちら。

```ruby:nfs_client.rb
cfn = JSON.load(::File.read('/opt/aws/cloud_formation.json'))
describe_cmd = "/usr/bin/aws ec2 describe-instances --instance-id #{cfn['nfs']['server']['instance-id']} --region #{node.ec2[:placement_availability_zone].chop} "
masterserver = JSON.load(`#{describe_cmd}`)
master_ip = masterserver['Reservations'].first['Instances'].first["PrivateIpAddress"]

mount "/var/www/vhosts/#{node[:ec2][:instance_id]}" do
  action [:mount]
  device "#{master_ip}:/var/www/vhosts/#{cfn['nfs']['server']['instance-id']}"
  fstype "nfs"
  options "rw"
end

mount "/var/cache/nginx/proxy_cache" do
  action [:mount]
  device "#{master_ip}:/var/cache/nginx/proxy_cache"
  fstype "nfs"
  options "rw"
end
```

実行時点でのServer役PrivateIPをマウントさせておしまいです。


とりあえずこうしておけば、NFS ClietのAutoScallingインスタンス達は不調になったりマウントが外れたりしたら再起動してもいいし、むしろ適当にTerminateしてしまえば勝手に補充されて復帰するので存分に使い棄てることができます。
