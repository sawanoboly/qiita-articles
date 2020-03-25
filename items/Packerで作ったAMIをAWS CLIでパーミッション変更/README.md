<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

まあPacker限定ではないんですがメモ書き程度で。
作ったAMIを他所に使わせる場合に使う、例えばAWS MarketPlaceとか。

とりあえずpackerでAMIのIDができたとします。

```
==> amazon-ebs: Stopping the source instance...
==> amazon-ebs: Waiting for the instance to stop...
==> amazon-ebs: Creating the AMI: Example AMI
    amazon-ebs: AMI: ami-xxxxxxxx
==> amazon-ebs: Waiting for AMI to become ready..
```


`ec2 describe-images`でSnapShotのIDをいただきます。

```shell:
$ aws ec2 describe-images --image-ids 'ami-xxxxxxxx' [--profile name]

{
    "Images": [
        {
            "VirtualizationType": "hvm", 
            "Name": "Example AMI", 
            "Hypervisor": "xen", 
            "ImageId": "ami-xxxxxxxx", 
            "State": "available", 
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/sda1", 
                    "Ebs": {
                        "DeleteOnTermination": true, 
                        "SnapshotId": "snap-xxxxxxxx",    ## SnapshotのID
                        "VolumeSize": 8, 
                        "VolumeType": "gp2", 
                        "Iops": 24
                    }
                }, 
                {
                    "DeviceName": "/dev/sdb", 
                    "VirtualName": "ephemeral0"
                }, 
                {
                    "DeviceName": "/dev/sdc", 
                    "VirtualName": "ephemeral1"
                }
            ], 
            "Architecture": "x86_64", 
            "ImageLocation": "6xxxxxxxx/Example AMI", 
            "RootDeviceType": "ebs", 
            "OwnerId": "6xxxxxxx", 
            "RootDeviceName": "/dev/sda1", 
            "Public": false, 
            "ImageType": "machine"
        }
    ]
}

```

jqでフィルタの例、スクリプトに組み込むなら。

```shell
$ aws ec2 describe-images --image-ids 'ami-xxxxxxxx'  --profile mp  | jq '.Images[0].BlockDeviceMappings[0].Ebs.SnapshotId'
"snap-cxxxxxxx"

```

## オーナー変更

AMIとSnapshotにAWS MarketPlaceさん向けのパーミッションを付けます。

AMIは`ec2 modify-image-attribute`で`--launch-permission`にAddします。


```shell:
$ aws ec2 modify-image-attribute --image-id ami-xxxxxxxx --launch-permission "{\"Add\": [{\"UserId\":\"67xxxxxxxxxx\"}]}" [--profile name]
{
    "return": "true"
}
```

Snapshotは`ec2 modify-snapshot-attribute`で`--create-volume-permission`にAddします。

```shell:
$ aws ec2 modify-snapshot-attribute --snapshot-id snap-xxxxxxxx --create-volume-permission "{\"Add\": [{\"UserId\":\"67xxxxxxxxxx\"}]}"  [--profile name]
{
    "return": "true"
}
```

これでOK。

----

追記：これを応用したBashスクリプトをGistにおきました。


https://gist.github.com/sawanoboly/b0a6d7289ad5e88177d7

<script src="https://gist.github.com/sawanoboly/b0a6d7289ad5e88177d7.js"></script>
