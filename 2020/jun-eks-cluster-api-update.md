# EKS cluster version update 1.14 -> 1.15

현재 1.14 사용중이고, 1.16까지 나와있음.  한번에 업데이트하면 안될것 같아서 하나씩 올리기로 결정

> 일단 1.14 -> 1.15로 업데이트



aws 콘솔들어가면 eks들어가서 생성해둔 클러스터를 보면 업데이트 하라고 나와있음.

업데이트 걸어두고 일단 자고 오면됨... (얼마나 걸리는지 잘 모르겟음)

이렇게 해두면 master node들이 업데이트 됨(워커노드들은 여전히 1.14사용중임)

워커노드의 버젼을 보고 싶다면 `kubectl get node` 해보면 맨 오른쪽에 버젼이 보임.



따로 worker node의 api만 변경은 어떻게 하는지는 모르겠고...

한가지 방법은 현재 있는 워커노드그룹들을 지금 사이즈에 맞춰서 똑같이 일단 하나 더 생성해 둬야함. (돈이 조금 들더라도 심리적으로 편안함, 그리고 나중에 실제로 노드 삭제할건데 그때 준비 안되어 있으면 장애발생 여지가 있음)

아래의 스펙이었다면(num instances는 콘솔보고 현재 사이즈..)

```yaml
node-group-1-v1:
  instance type: r5.xlarge
  num instances: 3
node-group-2-v1:
  instance type: p3.xlarge
  num instances: 2
```



이처럼 다른 이름으로 node group을 만들되 현재랑 같은 사이즈로 만들어둔다.

```yaml
node-group-1-v1:
  instanceType: r5.xlarge
  num instances: 3
node-group-2-v1:
  instanceType: p3.xlarge
  num instances: 2

node-group-1-v2:
  instanceType: r5.xlarge
  minSize: 3
node-group-2-v2:
  instanceType: p3.xlarge
  minSize: 2
```

노드 그룹 생성은 나는 eksctl로 관리하고 있어서 이걸로 생성함...



`kubectl get node` 해보면 1.14와 1.15가 공존해 있음.

이제부터 1.14노드들을 죽여갈거임..

`kubectl delete node $NODE_ID` 를 하게되면 노드가 삭제 되는데 삭제 될때, pod들에게 이벤트를 날림(나 죽을거니 더이상 request받지 말고 종료준비후 종료하라고. SIGTERM일거 같은데 테스트 안해봄.) - k8s에서 node가 삭제 되면 infra에서도 따로 인스턴스를 삭제해야함.

사실 `delete node` 하고 infra에서 따로 노드삭제하는게 귀찮아서 aws콘솔에서 > auto scaling group에서 인스턴스 갯수조절하면서 셧다운 시켜보았는데. 이러면 서비스 망... infra node삭제시 kubernetes쪽으로 이벤트가 안가는것 같음.. 그냥 삭제 되버려서.. 접속되어 있던 http 커넥션들 다 접속이 끊어져버림..... 아아아아악....



이렇게 다 바꾸면 끝..

그런데 문제는 DB들임... DB node가 replica일테니... 아래처럼 구성되어 있으면

```
node-group-1-v1.node-1: master
node-group-1-v1.node-2: slave-1
node-group-1-v1.node-3: slave-2
```

아무 slave(node-group-1-v1.instance-2: `slave-2`)가 속해 있는 node를 지우고, 그 `slave-2` 가 새 노드에 정상적으로 뜨는게 확인되면 `slave-2` 를 마스터로 변경하고. 마스터 변경되는게 확인되면 나머지도 하나씩 지워가면 됨. 근데 master바뀌면 디비 사용하는 쪽에서도 문제가 생기긴함...(이부분더 검증해보아야 하는데... sentry에는 에러가 뜬건 확인, 하지만 유저쪽 에러메세지는 오지 않음 - 좋은건가...)



디비는... 그냥 제공해 주는거 쓰는게 좋은것 같다.. 괜히 k8s에 올리지 말고.. 업데이트 안하면 사실 상관없는데 이러케 업데이트 하면... 뭔가 손으로 해주는게 좀 많은...



사실 메세지큐도 문제임.. replica가 아니라서... ㅠㅠ (이거 replica로 하는건 좀 오바같구... 크지도 않은 메세지큐..) - 잠깐 이슈 생김...

