
Docker for AWS(以下D4a)はGAの際、永続化ストレージをサポートするCloudstorというプラグインを添付するようになりました。

https://docs.docker.com/docker-for-aws/persistent-data-volumes/

D4aの`17.06.0-ce`でどのように使うのか試して見たので使用感など。

## Cloudstorがサポートするストレージのタイプ

- EBSにデバイスを作成してマウント
  - k8sでのPersistent Volumesのような感じ。
  - 複数コンテナからの同時利用ははなし。
- EFSで任意のディレクトリをマウント
  - 対象のEFSはテンプレートパラメータで自動作成。
  - dockerdが環境変数`EFS_ID_REGULAR`, `EFS_ID_MAXIO`で対象を決めるので、一応既存のEFSも使えなくはない。
  - 複数コンテナからの同時利用OK。


## つかってみる


ドキュメントではswarmモード(service)での使い方しか出ていませんが、単発の`docker run`でも普通に機能します。

次のようにvolume-driverに`cloudstor:aws`を指定、デフォルトはEFS。

```bash
$ docker run -it --rm \
  --volume-driver=cloudstor:aws \
  -v sharedvol1:/shareddata \
  alpine:latest
```

これで、EFSに`sharedvol1`ディレクトリが作成され、Dockerコンテナからは`/shareddata`で参照できます。

EFS側に親ディレクトリがない場合もソース指定可能です。

```bash
$ docker run -it --rm \
  --volume-driver=cloudstor:aws \
  -v sharedvol5/1/2/3:/shareddata \
  alpine:latest touch /shareddata/touchfile
```

この場合、Dockerコンテナ起動時にEFSに対して`sharedvol5/1/2/3`というディレクトリを作ります。

なおマウント時にサブディレクトリを指定した場合も、VOLUME名はトップのディレクトリが使われます。

```bash
$ docker volume ls
DRIVER              VOLUME NAME
cloudstor:aws       sharedvol1
cloudstor:aws       sharedvol2
cloudstor:aws       sharedvol5
local               sshkey
```

挙動はこんな感じです。基本的には`docker service/volume`で作成し、全ノードで参照できるようにしておくとよいでしょう。

### このとき、EFSの中身

他のEC2インスタンスから`/mnt/efs`にEFSをマウントして、中の様子を見てみます。
VOLUME名のディレクトリが作られているだけでした。シンプルですね。

```bash
$ find /mnt/efs
/mnt/efs
/mnt/efs/sharedvol2
/mnt/efs/sharedvol1
/mnt/efs/sharedvol1/testfile
/mnt/efs/sharedvol5
/mnt/efs/sharedvol5/1
/mnt/efs/sharedvol5/1/2
/mnt/efs/sharedvol5/1/2/3/touchfile
/mnt/efs/fs-ef07xxxx:sharedvol1
```

一番下にあるやつは、EFSのID指定できないかなと考えたけど、とりあえずやってみて失敗に終わったディレクトリです。
Cloudstorはソースが非公開なので、`-v fs-ef07xxxx:sharedvol1:/shareddata`などと実行したらこんなことに。



### docker volume rm で、データも消える

volumeの詳細をチェック。

```bash
$ docker volume inspect sharedvol5
[
    {
        "Driver": "cloudstor:aws",
        "Labels": null,
        "Mountpoint": "/mnt/efs/reg/sharedvol5",
        "Name": "sharedvol5",
        "Options": {},
        "Scope": "local"
    }
]
```

いたって普通です。EFSレギュラー/MAXIO用の親ディレクトリを作り、そこにマウントと。
D4aではちょっとわからないけども、ホストから見るとEFSをマウントしっぱなのでしょうね。

最後に消したらどうなるのかと、`docker volume rm sharedvol5`してみました。
まずdocker管理下のvolumeは消えます、そしてEFS側も綺麗さっぱりです。


```bash
# 他インスタンスからEFSの中身を参照
$ find /mnt/efs
/mnt/efs
/mnt/efs/sharedvol2
/mnt/efs/sharedvol1
/mnt/efs/sharedvol1/testfile
/mnt/efs/fs-ef07xxxx:sharedvol1
```

ここ連動して消えちゃうのか。。Docker視点ならそれでよいけど、コンフィグの共有とかでマウントするとかのNFSな視点だと違和感はあります。
マウントが外れるだけという選択肢があってもよいとおもって、[フォーラムに投稿](https://forums.docker.com/t/how-to-keep-files-on-efs-after-volume-rm/35234)しておきました。

## 終わりに

Docker Swarmが安定していればこの作りでも維持していけそうな気はしますが、なんだかんだでまだ不安定になったりします。
なにか手当をして回復するよりたいてい作り直した方が早いのですが、そういう運用ではvolumeのエントリーが巻き込まれてデータを消しそうな予感がしているため、(自分の用途では)フィードバックしつつもう少し様子見かなあ。


## EBSも少しだけ触って見る

volume-optに`backing=relocatable`を渡すと単体コンテナ用のEBSボリュームを作れます。

```bash
$ docker volume create -d cloudstor:aws \
  --opt backing=relocatable \
  --opt size=8 ebs1

=> ebs1
```

コマンド実行すると、ちゃんとEBSができていました。最初の紹介の通り、k8sのpvみたい。
こちらも`docker volume rm ebs1`などとすると、EBSごと消えます。
