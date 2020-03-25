<!-- permanent -->

VagrantでVMを作ってプロビジョニングをテストする際に、起動ディスク以外おｎディスクがアタッチされている状況を再現したい事もあります。
対象がPublic/PrivateなIaaSならEC2(EBS)だったり、CloudStack, OpenStackでも結構見られる状況かなと。

確認したのは`Vagrant 1.6.2`。


ざっとこんな感じで記述します。

```ruby:Vagrantfile
# -- snip --
    config.vm.define machine do |config|

      config.vm.provider :virtualbox do |vb|
        at_disk = 'tmp/name_of_file.vdi'
        unless File.exists?(at_disk)
          vb.customize ['createhd', '--filename', at_disk, '--size', 5 * 1024]
        end
        vb.customize ['storageattach', :id, '--storagectl', 'IDE Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', at_disk]
        end
      end

    end
# -- snip --
```

この記述で、次のような挙動になります。

- vagrant up時に仮想ディスクファイルを(作成して)アタッチ
- reset/reload時は内容残したままリブート
- vagrant destroy時に一緒に削除

デフォルトではdestroy時、一緒に消えることに注意です。


ついでに、`customize`についてはVBoxManageに渡すオプションを記述すればよいので、他のカスタマイズは`VBoxManage`本体のヘルプが参考になるはず。


例えば`storageattach`。

```
VBoxManage storageattach    <uuid|vmname>
                            --storagectl <name>
                            [--port <number>]
                            [--device <number>]
                            [--type dvddrive|hdd|fdd]
                            [--medium none|emptydrive|additions|
                                      <uuid|filename>|host:<drive>|iscsi]
                            [--mtype normal|writethrough|immutable|shareable|
                                     readonly|multiattach]
                            [--comment <text>]
                            [--setuuid <uuid>]
                            [--setparentuuid <uuid>]
                            [--passthrough on|off]
                            [--tempeject on|off]
                            [--nonrotational on|off]
                            [--discard on|off]
                            [--bandwidthgroup <name>]
                            [--forceunmount]
                            [--server <name>|<ip>]
                            [--target <target>]
                            [--tport <port>]
                            [--lun <lun>]
                            [--encodedlun <lun>]
                            [--username <username>]
                            [--password <password>]
                            [--initiator <initiator>]
                            [--intnet]

```

