癖で普段から沢山ターミナルを開いています。しかし、イベントでの発表とか、[ちょっとした都合](https://www.twitch.tv/nobolycloud)で自分の端末を映像配信・録音に使うなどのため、できるだけ負担を減らす目的で全部閉じることがあります。

で、閉じる前何やってたっけ。。があるのでダンプしておくことにしよう。

lsofでcwdつき出力して、あとは良くあるフィルタのワンライナーで実施。

```bash
$ for x in $(pidof bash); do lsof -a -d cwd -p $x ; done |  grep bash | awk '{print $9}' | sort | uniq
/Users/sawanoboly
/Users/sawanoboly/develop/src/git.example.com/kafka-docker
/Users/sawanoboly/develop/src/git.example.com/zookeeper-docker/stack
/Users/sawanoboly/develop/src/git.example.com/sawanoboly/kafka-miscs
/Users/sawanoboly/develop/src/git.example.com/sawanoboly/kafka-miscs/landloop
/Users/sawanoboly/develop/src/github.com/confluentinc/cp-docker-images
/Users/sawanoboly/develop/src/github.com/getshifter/base
/Users/sawanoboly/sandbox/kafka-leaning/fluentd
/Users/sawanoboly/sandbox/spot/opencv
/Users/sawanoboly/sandbox/spot/opencv/face_recognition/examples
/Users/sawanoboly/sandbox/spinnaker
```

脳内コンテキスト復旧の手助けになるよう。
