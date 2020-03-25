
DatadogのDockerホスト向けAgentはD4xにちょうどいいかもしれないと思ったので起動してみる。

## run => serviceに変換する

Dockerホスト向けAgentのインストラクションからはこう来るので、

```
docker run -d --name dd-agent \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  -e API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx datadog/docker-dd-agent:latest
```

globalモードのサービスとして動かせるように変換と。こうしておけばノードが増減しても全体でAgentが稼働する状況を保てる。

```
docker service create --name global-dd-agent \
  --mode global \
  --limit-cpu 1 \
  --mount type=bind,source=/var/run/docker.sock,target=/var/run/docker.sock,readonly=true \
  --mount type=bind,source=/proc/,target=/host/proc/,readonly=true \
  --mount type=bind,source=/sys/fs/cgroup/,target=/host/sys/fs/cgroup,readonly=true \
  -e API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx \
  datadog/docker-dd-agent:latest
```

そういえばglobalにも対象絞りができるんだろうか？まあ今回はいいや。

## Serviceの確認

全部のNodeでAgentが動きだしたね。

```
$ docker service ps global-dd-agent
ID                         NAME                 IMAGE                           NODE                            DESIRED STATE  CURRENT STATE           ERROR
b2m3d5lvyng0dn0p20ubpktpj  global-dd-agent      datadog/docker-dd-agent:latest  ip-192-168-xxx-xxx.ec2.internal  Running        Running 25 seconds ago  
b71c81rhk8k1qzybme6on0csl   \_ global-dd-agent  datadog/docker-dd-agent:latest  ip-192-168-xxx-xxx.ec2.internal  Running        Running 20 seconds ago  
0tkbe30m150unwuh58hw4svb2   \_ global-dd-agent  datadog/docker-dd-agent:latest  ip-192-168-xxx-xxx.ec2.internal  Running        Running 23 seconds ago  
arc0tozq3ypoltqntli23eo4a   \_ global-dd-agent  datadog/docker-dd-agent:latest  ip-192-168-xxx-xxx.ec2.internal  Running        Running 12 seconds ago  
ev76xdu7vc8r1pgldd1g5ui1f   \_ global-dd-agent  datadog/docker-dd-agent:latest  ip-192-168-xxx-xxx.ec2.internal  Running        Running 22 seconds ago  
8atpxhwzhf4lie9xs50rq5geb   \_ global-dd-agent  datadog/docker-dd-agent:latest  ip-192-168-xxx-xxx.ec2.internal  Running        Running 20 seconds ago
```

Datadog側にもガサっと情報が上がってきました。

![Events___Datadog.jpg](https://qiita-image-store.s3.amazonaws.com/0/7454/610a0c14-e933-89d8-0f5f-18c11a0c4600.jpeg "Events___Datadog.jpg")

Dockerコンテナの作られたりする様子もあがってきて色々と丁度いいかも。


## おわりに

globalのサービスは、一発タスクにも使える。例えば事前にdocker imageをpullしておきたいとか。
