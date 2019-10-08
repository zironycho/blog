[Home](/README.md)

# KubeDB

>  쿠버네티스에서 database provisioning, patching, backup, recovery, failure detection, repair를 간단하게 해줄 수 있음. - https://kubedb.com/

현재버젼: 0.13.0-rc.0



최근 mongodb의 transaction기능을 사용하려고 하니 문제발생.

4.0이상으로 업그레이드 해야하고, replicaset으로 구성해야함.

지금까지는 3.6에 single로 배포하고 사용중이었음.



MongoDB replicaset 구성하려다가 귀찮아서 찾아봄. mongodb 회사 솔루션도 있으나 가격이 보여서 일단 버림

* https://www.mongodb.com/kubernetes  



## 지원되는 데이터베이스

* Elasticsearch
* Memcached
* MongoDB
* MySQL
* PostgreSQL
* Redis



Clustering - Sharding, Replication을 recipe하나로 간단하게 구성할 수 있음.



### ReplicaSet

> storage class를  gp2로 바꿔야함 (AWS EKS)

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo
  namespace: demo
spec:
  version: "4.0.5-v1"
  storageType: Durable
  replicas: 3
  replicaSet:
    name: rs0
  storage:
    storageClassName: "gp2"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```



### Auth 변경

```yaml
apiVersion: kubedb.com/v1alpha1
...
spec:
  databaseSecret:
    secretName: mgo-auth
---

apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: mgo-auth
  namespace: demo
data:
  user: dXNlcg==
  password: cGFzcw==
```



### service 추가

> 기존에 몽고 서비스 이름을 사용하는 곳이 많으면 이런식으로 추가해 주는게 나음..

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: demo
  name: my-mgo-svc
spec:
  ports:
  - port: 27017
    targetPort: 27017
    protocol: TCP
  selector:
    kubedb.com/kind: MongoDB
    kubedb.com/name: mgo
```



### Shard

> 안해봄...

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mongo-sh
  namespace: demo
spec:
  version: 3.6-v3
  shardTopology:
    configServer:
      replicas: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: gp2
    mongos:
      replicas: 2
      strategy:
        type: RollingUpdate
    shard:
      replicas: 3
      shards: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: gp2
```



pvc가 바로 지워지지 않아서 다시 사용할 수 있음.

