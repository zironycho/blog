[Home](/README.md)

# docker plugin rexray/s3fs for swarm
By [zironycho](http://github.com/zironycho) :heart: [Neosapience, Inc](http://www.neosapience.com)

**2018.02.28**

## Preparing
* Make s3 full permission on AWS/IAM/users
* Basic for generation Swarm Cluster

## In single node
```bash
# install plugin
$ docker plugin install rexray/s3fs \
  --grant-all-permissions \
  S3FS_REGION=ap-southeast-1 \
  S3FS_ENDPOINT=s3.ap-southeast-1.amazonaws.com \
  S3FS_ACCESSKEY={{ aws access key }} \
  S3FS_SECRETKEY={{ aws secure key }}

# create & mount volume
$ docker volume create \
  -d rexray/s3fs \
  --name my-bucket-t9999

# create file in s3 bucket
$ docker run --rm -t \
  -v my-bucket-t9999:/data \
  alpine \
  touch /data/hello
```

And check your data in s3 bucket on aws console
```bash
$ aws s3 ls my-bucket-t9999 --recursive
2018-02-28 13:42:22          0 data/
2018-02-28 13:42:24          0 data/hello
```

## In swarm cluster mode
cluster일 경우 single node처럼 하면 안된다. 나의 경우에는 두가지를 셋팅해 주고 해결하였다.
* 모든 swarm node에서 plugin install을 해야한다.
* master node에서 service 수행시 `/run/docker/plugins` 볼륨을 추가로 마운트해 준다.

`/run/docker/plugins/{{ plugin id }}` 아래에 rexray.sock이 있는데 이것을 공유해야만 정상적으로 동작한다.


(이미 swarm cluster가 구성되어 있다고 가정하겠다.) master node에서는 아래와 같이 서비스를 수행시킨다.
```
docker service create -d \
  --mount "type=bind,source=/run/docker/plugins,target=/run/docker/plugins" \
  --mount "type=volume,volume-type=rexray/s3fs,source=my-bucket-t9999,target=/data" \
  -p 5000:5000 \
  --replicas=10 \
  --name uuid \
  zironycho/uuid
```

ingress network에 5000포트를 오픈하였으므로 어떤 swarm node에서든 5000번으로 request를 날리면 uuid를 받을 것이다.
```
curl {{your host address}}:/5000
```

그리고 uuid를 확인할 수 있을 것이다.
```
curl {{your host address}}:/5000/{{ uuid }}
```

그리고 s3 console에서도 확인할 수 있을 것이다.
```
aws s3 ls my-bucket-t9999 --recursive
```
