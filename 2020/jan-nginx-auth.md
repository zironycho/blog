## nginx Ingress with auth

### 왜 썻나요??

개발용, 내부용 대쉬보드 만들때 auth를 하나하나 만들기 귀찮을 때가 있음....

예를들면:

https://jaeger.example.com/

- 예거를 그냥 올려두면 아무나 접속할 수 있기 때문에 auth를 셋팅해야함.

- 기본애플리케이션에 내장 안되어 있는 개발용 대쉬보드들이 많음...

### Prerequisites

* Kubernetes

* nginx ingress controller

  

### nginx ingress를 사용하는 이유.

AWS 로드벨런스 하나에 여러 namespace에 있는 서비스에 연결할수 있음

이 nginx ingress에서 basic auth, oauth2, external basic auth를 사용할 수 있도록 셋팅할 수 있음.

https://kubernetes.github.io/ingress-nginx/examples/auth/basic/



* ingress recipe (annotation 부분)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # basic auth part
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'

spec:
  rules:
  - host: jaeger.example.com
    http:
      paths:
      - backend:
        ...
```

* htpasswd

```
bar:$apr1$duJqCXdR$mAS1v8kw.ggLUamgjSos2.
foo:$apr1$HygaQFyV$Bst5BsjCy/u3855wM8TaH0
```



### 다른회사들은 어떻게 하고 있는지?

* vpn, firewall
  * `+`인증(sso)
* acl: 허용하는 ip들만



## ref

https://httpd.apache.org/docs/2.4/programs/htpasswd.html