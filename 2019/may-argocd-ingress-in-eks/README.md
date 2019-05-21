# argocd for aws eks alb-ingress 

## argocd

기본 argocd를 설치하는건 [링크](https://github.com/argoproj/argo-cd/blob/master/docs/getting_started.md) 를 따라가면 설치할 수 있다. 다만 여기에 amazon eks에 alb ingress를 설치해야할 경우 아래와 같은 셋업들을 해야한다.

위의 상태까지 쿠버네티스에 배포하게되면, 서비스는 퍼블릭하게 오픈되지 않는다. 서비스를 퍼블릭하게 오픈하고 특정 도메인에 연결해 주기 위해서는 [alb-ingress](https://github.com/kubernetes-sigs/aws-alb-ingress-controller)나, [external-dns](https://github.com/kubernetes-incubator/external-dns) + service(spec.type: LoadBalancer)를 통해서 가능하다. 나는 alb-ingress를 통해서 사용했다.

위의 install.yaml에는 full recipe가 다 있다. 하지만, alb-ingress에 연동하기 위해서는 recipe를 몇개 수정해야한다. argocd-server deployment, svc를 수정해야하고 ingress를 추가해야한다.  [링크](https://gist.github.com/zironycho/86ab13e952281ab2d1383843b9cafadf) 참조.

* argocd-server deployment
  * spec.template.spec.container.comman에 마지막에 `- —insecure` 추가하기
    * 이부분 안하면 무한 redirect된다. alb에서 https에서 오더라도 backend에 http로 전달한다.
* argocd-server service
  * spec.type: `NodePort` 를 할당한다. alb-ingress 사용하기 위해서는 서비스 타입이 nodeport여야 한다.


그리고 ingress recipe를 만들어 줘야한다.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: argocd
  name: argocd-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80,"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: {{ your-acm-arn }}
    alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'

  labels:
    app: argocd-ingress
spec:
  rules:
    - host: {{ argocd-service-host }}
      http:
        paths:
        - backend:
            serviceName: ssl-redirect
            servicePort: use-annotation
        - backend:
            serviceName: argocd-server
            servicePort: 80
```

* `alb.ingress.kubernetes.io/listen-ports` 를 통해서 alb의 80,443을 오픈해주고
* `alb.ingress.kubernetes.io/certificate-arn` 를 통해서 acm arn을 할당해주고
* `alb.ingress.kubernetes.io/actions.ssl-redirect: '{"Type": "redirect", "RedirectConfig": { "Protocol": "HTTPS", "Port": "443", "StatusCode": "HTTP_301"}}'` 이부분을 통해서 http는 항상 https로 redirect 해준다.

* `{{ argocd-service-host }}` 이부분에 route53에 등록되어 있는 도메인을 연결하면, 알아서 route53에 등록해준다.


## argocd cli
argocd cli는 grpc를 사용하기에 alb-ingress를 사용하면 동작하지 않는다. argocd cli에서 alb-ingress를 통해 접속하기 위해서는 로그인시 `--grpc-web`을 추가로 넣어주면 동작한다.


## Refs:
* https://github.com/kubernetes-sigs/aws-alb-ingress-controller
* https://argoproj.github.io/argo-cd/
* https://argoproj.github.io/argo-cd/operator-manual/ingress/
* https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/tasks/ssl_redirect/
* https://blog.argoproj.io/how-to-eat-the-grpc-cake-and-have-it-too-77bc4ed555f6
