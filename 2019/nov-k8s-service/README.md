[Home](/README.md)

# kubernetes - service



## Before..

**node**: worker machine in Kubernetes, part of cluster

**pod**: the basic execution unit of a Kubernetes application (smallest and simplest unit in the Kubernetes object model)

* 팟 안에는 여러개의 다커 컨테이너가 실행될수 있고, localhost로 file및 네트워크를 공유

**controller**: pod을 관리하기 위한 컨트롤러

* replicaset
* Deployments
* StatefulSets
* DaemonSets
* Jobs
* CronJob

**service**:

실행되고 있는 팟 셋들을 접근가능하게 하는 추상적 방법.  팟 셋들에 DNS이름을 부여한다.

`{{ service name }}.{{ namespace }}.svc.cluster.local`

**팟 셋(set of pods)**: 하나의 디플로이먼트/스테이트풀셋 등으로 생성되는 같은 meta.name안에 있는 팟들.



팟은 생성되고, 삭제되며, 한번 생성됐던 팟이 다시 같은 아이피로 사용되지 않는다. 생성될 때마다 같은 아이피를 받는다는 보장이 없다. 팟들이 어떤 아이피를 할당받을 지 알수 없다. 만약 어떤 팟 셋 (-프론트엔드라고 불리우는)이 다른 팟셋(-백엔드라고 불리는)의 api를 사용한다면 프론트엔드는 백엔드에 있는 팟들의 아이피를 알아야 한다. 그 아이피를 바로 접근할 수도 있지만, 그렇게 되면 아이피가 바뀔때마다(팟이 생성 될 때마다 - ex.API를 버젼업해서 릴리즈 할 때마다) 아이피를 업데이트 해줘야 한다. 때문에 이 팟 셋을 묶는 추상적인 개념이 필요하고 그게 서비스다.

서비스가 어떤 팟 셋들에게 향할지는 [selector](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)를 통해 결정된다.

### 서비스 정의하기

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

`app=MyApp` 이라고 어딘가 정의 되어 있는 팟 셋 들의 TCP 9376포트를 80포트로 접근가능하게 하는 서비스로 이름은 my-service이다. 같은 네임스페이스에 있다면 `my-service:80` 으로 접근이 가능하다.

쿠버네티스는 위의 서비스에 아이피(cluster IP)를 할당하고 이것은 [서비스 프록시](virtual ip and service proxies)에서 사용하고 있다. 



#### selector 없는 서비스

* 외부 디비사용
* 외부 API사용
* 다른 네임스페이스 사용

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```

엔드포인트 오브젝트를 만들고 여기에 외부에서 사용하는 IP를 붙일 수 있다.  Endpoints의 IP는 외부 쿠버네티스의 cluster IP를 사용할 수 없다.



#### Publishing Services(ServiceType)

Type을 지정하지 않으면 ClusterIP로 동작

* **ClusterIP**: 클러스터 내부 아이피로 노출시킴. 클러스터 안에서만 이 주소로 접근이 가능

  ![](https://miro.medium.com/max/1700/1*I4j4xaaxsuchdvO66V3lAg.png)

  ```
  light-redis ClusterIP 10.100.91.203 <none> 6379/TCP 211d
  ```

* **NodePort**: 노드에 특정포트(기본 30000-32767)를 자동으로 할당해서 클러스터 외부에서 접근이 가능하게 함

  ![](https://miro.medium.com/max/2094/1*CdyUtG-8CfGu2oFC5s0KwA.png)

  ```bash
  light-api NodePort 10.100.134.179 <none> 80:30427/TCP 211d
  ```

* **LoadBalancer**: 서비스 프로바이더(aws, gcp, azure, ...)에서 제공하는 로드발란서에 이 서비스를 바로 붙임

  ![](https://miro.medium.com/max/1826/1*P-10bQg_1VheU9DRlvHBTQ.png)

  ```
  ingress-nginx LoadBalancer 10.100.72.159 xxxxxx.ap-northeast-2.elb.amazonaws.com 80:31930/TCP,443:32120/TCP 58d
  ```

* **ExternalName**: (사용해 보지 않음..) CNAME 레코드에 `spec.externalName`의 값을 할당. 

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: my-service
    namespace: prod
  spec:
    type: ExternalName
    externalName: my.database.example.com
  ```

  이처럼 있을 때,  `http://my-service`를 같은 네임스페이스에서 콜하게 되면 my.database.example.com 이쪽을 호출하게 됨





### 서비스타입은 아니지만 외부에 Endpoint로 사용할 수 있는 Ingress

클러스터 외부에서 http, https로 접근해서 내부의 서비스로 연결시켜주는 오브젝트. 

```bahsh
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

![](https://miro.medium.com/max/3970/1*KIVa4hUVZxg-8Ncabo8pdg.png)



example cata ingress:

```
cata-ingress-nginx cata.neosapience.xyz xxxxx.ap-northeast-2.elb.amazonaws.com
80 59d
```



L7 (application layer)로 동작

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080
```

```
foo.bar.com -> 178.91.123.132 -> /foo    service1:4200
                                 /bar    service2:8080
```



```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80
```

```
foo.bar.com --|                 |-> foo.bar.com service1:80
              | 178.91.123.132  |
bar.foo.com --|                 |-> bar.foo.com service2:80
```





ingress가 동작하기 위해서는 ingress controller가 필요함

현재 우리는 2가지 사용중

* aws alb ingress: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
  * 이걸 쓰기위해서는 service가 NodePort여야함.
* nginx ingress: https://www.nginx.com/products/nginx/kubernetes-ingress-controller

다음엔 [Ambassador](https://www.getambassador.io/) 써보고 싶음.






## ref

https://kubernetes.io/docs/concepts/services-networking/service/

https://kubernetes.io/docs/concepts/services-networking/ingress/

https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0


