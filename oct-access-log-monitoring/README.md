# Access Log Monitoring



## nginx access log monitoring

* ELK + filebeat
  * filebeat + nginx 
    * filebeat 도커 이미지 사이즈가 `328MB`
  * swarm에서는 pod같은 개념이 없다...
    * log 볼륨을 같이 사용하기 위해서는 같은 노드에 띄우거나 
    * 네트워크 볼륨을 만들어야 하는데.. 구지 이짓거리를 할 필요가...
      * aws ebs는 단일 노드에만 접속하고...
      * aws efs는 여러 노드에서 붙을 수 있는데.. 굳이 ….. 이렇게 까지 셋팅을 해야하나...
    * 루트볼륨 하나를 사용해서 내부 노드끼리 공유 할 수도 있긴 하지만 서비스 하는 입장에서.. 비정상임...
  * 빠른포기.



* EK + fluent-bit

  * fluent bit output에 es output이 있음...

  * fluent-bit + nginx alpine

    * fluent-bit는 alpine 제공하지 않음: https://github.com/fluent/fluent-bit-docker-image/issues/12
    * alpine linux에서 빌드하니 되네....

    ```dockerfile
    FROM zironycho/fluent-bit:0.14.4-alpine3.8 as build
    
    FROM nginx:1.15.5-alpine
    ENV timezone Asia/Seoul
    
    RUN apk update && apk --no-cache add \
      libgcc \
      ca-certificates \
      logrotate \
      tzdata \
      && cp /usr/share/zoneinfo/$timezone /etc/localtime \
      && echo "$timezone" > /etc/timezone \
      && apk del tzdata \
      && rm -rf /var/cache/apk/*
    
    COPY --from=build /fluent-bit /fluent-bit
    
    COPY nginx /etc/nginx
    COPY fluent-bit /fluent-bit
    COPY app/dist /usr/frontend
    COPY logrotate.d /etc/logrotate.d
    
    ENV ES_HOST elasticsearch
    ENV ES_PORT 9200
    ENV ES_ENABLE_TLS Off
    ENV ES_USER=
    ENV ES_PASS=
    
    COPY ./entrypoint.sh /usr/entrypoint.sh
    
    ENTRYPOINT [ "/usr/entrypoint.sh" ]
    ```

    * nginx setup

      * log rotation

      ```sh
      # /etc/logrotate.d/nginx
      /var/log/nginx/*.rot.log {
      	hourly
      	rotate 4
      	missingok
      	sharedscripts
      	postrotate
      	  [ ! -f /var/run/nginx.pid ] || kill -USR1 `cat /var/run/nginx.pid`
      	endscript
      }
      ```

      * save log, not default log file

      ```dockerfile
      # /etc/nginx/conf.d/nginx.conf
      # access_log  /var/log/nginx/access.log  main;
      access_log /var/log/nginx/access.rot.log main;
      ```

    * fluent-bit setup 

      * nginx parser 추가 셋업( 기본 셋팅엔 http_x_forwaded_for 변수가 없네...)

      ```ini
      [PARSER]
          Name   nginx-1.15.5
          Format regex
          Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")? "(?<http_x_forwarded_for>[^\"]*)"$
          Time_Key time
          Time_Format %d/%b/%Y:%H:%M:%S %z
      ```

      * fluent-bit config

      ```
      [INPUT]
          Name tail
          Tag  local.nginx.access
          Path /var/log/nginx/access.rot.log
          Parser nginx-1.15.5
          Mem_Buf_Limit 10MB
          Skip_Long_Lines On
          Interval_Sec 10
          
      [OUTPUT]
          Name es
          Match local.nginx.access
          Logstash_Format On
          Index fluentbit
          Generate_ID On
          HTTP_User ${ES_USER}
          HTTP_Passwd ${ES_PASS}
          Host ${ES_HOST}
          Port ${ES_PORT}
          TLS ${ES_ENABLE_TLS}
      ```

    * docker entrypoint

    ```sh
    #!/bin/sh
    nginx -g "daemon on;" && /fluent-bit/bin/fluent-bit -c /fluent-bit/etc/fluent-bit.conf
    ```

  * local EK setup for development environement

  ```yaml
  volumes:
    esdata1:
      driver: local
  networks:
    backend:
      driver: bridge
  
  services:
    elasticsearch:
      image: docker.elastic.co/elasticsearch/elasticsearch:6.4.2
      volumes:
        - esdata1:/usr/share/elasticsearch/data
      networks:
        - backend
    kibana:
      image: docker.elastic.co/kibana/kibana:6.4.2
      networks:
        - backend
      ports:
        - 5601:5601
  ```

  * 스트레스 테스트 같은건 안해봄...
  * 서비스 사용자에게 전달, 처음에 잘 되는 듯 했으나...

    * 502에러 남발....
  * 원인 찾던중 fluent-bit + nginx 이미지가 트레픽좀 들어오면 바로 dead & restart

    * 원인은 fluentbit가 죽음… sigXXX ?
    * 분석할 여력없음...
  * 스트레스 테스를 위해서 wrk로 4 thread 100개 커넥션 물려서 돌려봄

    * dead...
    * 맨붕..
    * 일단은 서비스사용자에게 서비스 자체를 제공해야해서 nginx access log를 버림..



## AWS ALB access log monitoring

* AWS ALB access log

  * aws alb를 사용중이어서 여기서 바로 access log를 추출하기로 결정
  * s3에 남기는 것은 쉬움.. 근데 time series visualization은?
    * aws athena
      * visualization이 안되는듯?
    * redash
      * 대충 되는 줄 알고 이것저것 셋팅해 보았는데 athena와 별 차이 없음.
    * logz.io (ELK)
      * s3로 저장해둔 access log를 time series로 visualization가능
      * limit: 셋팅후 이틀치만 fetch해옴......
        * s3 bucket을 이틀치씩 나눠서 저장… (5일 밖에 안했어서 손으로 copy & paste)
      * community version
        * 3days retention
        * 3GB per day

