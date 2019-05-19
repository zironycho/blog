# `nginx:alpine` docker for AWS ELB SSL
By [zironycho](http://github.com/zironycho) :heart: [Neosapience, Inc](http://www.neosapience.com)

**2018.03.07**


<br><br><br>
## 목차
* 어떤 사람을 위한 문서인가?
* 시작하게 된 배경
* Directory skeleton
* Dockerfile 작성

<br><br><br><br><br><br><br><br><br><br><br><br>
## 어떤 사람을 위한 문서인가?
* docker를 활용한 aws사용
* nginx 초보
* 라이브서버 이외에 개발서버, 로컬서버까지 모두 커버하기 위한 nginx구성하고 싶은 사람

<br><br><br><br><br><br><br><br><br><br><br><br>
## 시작하게 된 배경
처음에는 elb에 http만 붙여서 사용했었다.
```
[http url] ---> [elb:80] ---> [nginx:80] --> [app]
```
<br><br>
그러다가 ssl까지 포함시켜 보니 아래처럼 두개의 별도의 서버가 되었다.
```
[http url] ---> [elb:80] ---> [nginx:80] --> [app]
[https url] --> [elb:443] --> [nginx:80] --> [app]
```
<br><br>
내가 원하는 것은 이런게 아니었다. 아래처럼 http로 접속하게 되면 https로 redirection시켜 줘서 항상 https로 접속하게 만들고 싶었다.
```
[http url] --------> [elb:80] ---> [nginx:80]
[ * ] <--------------------------- [301, https url]
[https redirect] --> [elb:443] --> [nginx:80] ------> [app]
```
<br><br>
이 문제를 해결하기는 매우 간단하다. http로 들어오면 https로 redirection시키면 끝이다.
```nginx
server {
    listen 80;
    
    server_name app.example.com;
    if ($http_x_forwarded_proto = 'http') {
        return 301 https://$server_name$request_uri;
    }
    
    location / {
        ...
        ...
    }

    ...
    ...
}
```
<br><br>
그런데, 라이브서버만 있는 것은 아니라서 이렇게 하면 안된다. 나에게는 라이브서버와 개발서버 그리고 로컬서버가 있다(나중에는 CI용으로도 더 생기겠지만...). docker-compose와 swarm을 이용해서 개발하고 있기 때문에 이정도는 있어야한다. 로컬서버와 개발서버가 접속하는 경로가 다르기 때문에 위처럼 해결할 수 없다. 개발서버와 라이브서버를 합치기 위해 `server_name`에 `*.example.com` 를 넣어봤지만.. 안된다. redirection 주소에 `*`가 들어가 버린다. 

<br><br>
하위에 location들은 같지만 주소들이 다르고, elb가 없는 것도 있다.
```
live server:  [app.example.com:http/https] -----> [elb:80,443] -> [nginx port:80] -> [app]
dev server:   [dev-app.example.com:http/https] -> [elb:80,443] -> [nginx port:80] -> [app]
local server: [localhost:http] ---------------------------------> [nginx port:80] -> [app]
```
* 해결책은 결국 location부분만 빼서 include 시키고 각각의 서버이름으로 server block을 만들어야 한다.

<br><br><br><br><br><br><br><br><br><br><br><br>
## Directory skeleton
아래의 구조를 통으로 이미지에 복사해 넣을 것이다. 이미지를 빌드할 때, 디렉토리 전테를 복사하면 base image에 있던 것은 유지되고 추가된 것만 복사가 이뤄진다(`docker run` 시킬때 --mount 옵션과는 다르다).
```bash
nginx/
└── conf.d/
    ├── .location.conf
    ├── default.conf
    ├── forward-dev.conf
    └── forward-live.conf
```


### nginx/conf.d/default.conf
`ngnix:alpine` 기본 이미지에 `default.conf`가 들어 있어서 이름 하나는 어쩔수 없이 `default.conf`로 만들어서 기본 이미지의 것을 덮어써야 한다. 나는 default.conf는 로컬서버를 위해서 셋팅했다. 그리고 location block을 include해 주었다. location block 파트는 아래쪽에 나온다.

```nginx
server {
    listen 80;
    
    include /etc/nginx/conf.d/.location.conf;
}
```

### nginx/conf.d/forward-live.conf
app.example.com에 http로 들어오는 트래픽을 전부 https로 바꿔준다. 그리고 여기서도 공통으로 사용하는 location block을 include 해준다.

```nginx
server {
    listen 80;
    
    server_name app.example.com;
    if ($http_x_forwarded_proto = 'http') {
        return 301 https://$server_name$request_uri;
    }
    
    include /etc/nginx/conf.d/.location.conf;
}
```

### nginx/conf.d/forward-dev.conf
`foward-live.conf`와 셋팅이 같지만, `server_name`이 있는 한 줄만 수정하면 된다.
```nginx
server_name dev-app.example.com;
```

### nginx/conf.d/.location.conf
기본적인 nginx의 location block 셋팅들이다. 위의 세가지 서버가 모두 공유하기 위해서 이렇게 따로 분리하였다. 파일이름을 `.`으로 시작함으로써 http server block에 추가하지 못하도록 하였다(/etc/nginx/nginx.conf를 들여다 보면 http block에서 /etc/nginx/conf.d/\*.conf를 include하는 부분이 있다).

```nginx
location /api {
    ...
}

location / {
    ...
}

...
...

```

<br><br><br><br><br><br><br><br><br><br><br><br>
## Dockerfile 작성
Dockerfile을 아래와 같이 작성하고 빌드하게 되면, 기존의 이미지의 /etc/nginx를 덮어 씌우지 않고 추가된 부분만 복사한다.

```Dockerfile
FROM nginx:1.13-alpine
COPY ./nginx /etc/nginx

...
...

```

도커 실행은 다들 아실테니... pass..

<br><br><br><br><br><br><br><br><br><br><br><br>
## References
* https://aws.amazon.com/premiumsupport/knowledge-center/redirect-http-https-elb/
* https://serverfault.com/questions/373578/how-can-i-configure-nginx-locations-to-share-common-configuration-options
* https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms
