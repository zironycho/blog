[Home](/README.md)

# AWS EKS에서 ingress사용하기



### 들어가기전… 심심풀이

- kubectl: 큐브커를
- eksctl: 이케이에스커를
- helm: 헮
- kops: 캅스
- etcd: 엩씨디
- docker: 다커



## 쿠버네티스에서 ingress는 무엇?

외부의 인터넷과 통신하는 입구역할을 하는 로드벨런서 같은 서비스타입은 두가지 종류가 있다.

하나는 `LoadBalancer`이고 다른 하나는 `Ingress` 이다.



## 왜써야 함?

* `LoadBalancer` 를 사용하게 되면 AWS기준으로 ALB(정확히 안봐서 기억이 안난다 ALB인지 NLB인지)가 하나 생성되기 때문에, `deployment` 에 외부 인터넷의 접근을 허용할 때 마다 만들게 되면, 비용이 많이 잡히게 된다. (AWS에서는 ALB하나당 비용이 있다) 별일 안해도 약 `18,000원`정도 나왔던것 같다.
* `Ingress` 를 사용하게 되면 layer7 라우팅을 지원하기 때문에 여러개의 서비스를 묶어서 사용할 수 있다. 그렇기 때문에 비용절감이 가능하다.

![](https://cdn-images-1.medium.com/max/1600/1*P-10bQg_1VheU9DRlvHBTQ.png)

![](https://cdn-images-1.medium.com/max/2400/1*KIVa4hUVZxg-8Ncabo8pdg.png)

Thanks to [Ahmet Alp Balkan](https://medium.com/@ahmetb) for the diagrams





## 어떻게 사용함?

* https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/ 이 링크에서 deploy part 있는데 이걸 deploy해 두고.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
  labels:
    app: my-ingress
spec:
  rules:
    - http:
        paths:
          - path: /api/*
            backend:
              serviceName: "my-api"
              servicePort: 80
          - path: /share/*
            backend:
              serviceName: "my-api"
              servicePort: 80
          - path: /*
            backend:
              serviceName: "my-frontend"
              servicePort: 80

```

* aws alb에도 셋업확인: https://ap-northeast-2.console.aws.amazon.com/ec2/v2/home?region=ap-northeast-2#LoadBalancers:sort=desc:createdTime

* 그리고 route53을 연동하기 위해서는 [external-dns](https://github.com/kubernetes-incubator/external-dns) 를 kubernetes에 설치해야함.
* 아래와 같이 ingress config를 업데이트 해줘야함. 

```yaml
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: "arn:...:certificate/..."
    external-dns.alpha.kubernetes.io/hostname: "example.com."
    ...
spec:
  ...
```



## Ref.

* https://aws.amazon.com/blogs/opensource/kubernetes-ingress-aws-alb-ingress-controller/
* https://kubernetes.io/docs/concepts/services-networking/ingress/
* https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0

