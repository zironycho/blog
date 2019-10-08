[Home](/README.md)

# Outline

> Team wiki - https://www.getoutline.com/







Free wiki site구축을 위해 찾아보고 있었음.

slack / gsuite으로 계정연동이 가능한 점이 메리트. 

![](img/outline.png)



embed로 사용될 다양한 플랫폼이 integration 되어 있음

![](img/outline-2.png)



### docker compose로 간단하게 구성

* s3, outline app, redis, postgres 를 띄우면 됨

  ```yaml
  # https://raw.githubusercontent.com/outline/outline/master/docker-compose.yml
  version: "3"
  services:
    redis:
      image: redis
      ports:
        - "6380:6379"
    postgres:
      image: postgres
      ports:
        - "5434:5432"
      environment:
        POSTGRES_USER: user
        POSTGRES_PASSWORD: pass
        POSTGRES_DB: outline
    s3:
      image: lphoward/fake-s3
      ports:
        - "4569:4569"
      volumes:
        - ./fakes3:/fakes3_root
    outline:
      image: outline:v001
      command: yarn dev
      build:
        context: .
        dockerfile: Dockerfile
        args:
          pull: 1
      ports:
        - "3000:3000"
      volumes:
        - .:/opt/outline
      depends_on:
        - postgres
        - redis
        - s3
  ```

  

* 필요한 config는 다 환경변수로 들어가있음

  ```ini
  # https://raw.githubusercontent.com/outline/outline/master/.env.sample
  # 
  # Copy this file to .env, remove this comment and change the keys. For development
  # with docker this should mostly work out of the box other than setting the Slack
  # keys (for auth) and the SECRET_KEY.
  #
  # Please use `openssl rand -hex 32` to create SECRET_KEY
  SECRET_KEY=generate_a_new_key
  
  DATABASE_URL=postgres://user:pass@postgres:5432/outline
  DATABASE_URL_TEST=postgres://user:pass@postgres:5432/outline-test
  REDIS_URL=redis://redis:6379
  
  URL=http://localhost:3000
  PORT=3000
  
  # enforce (auto redirect to) https in production, (optional) default is true.
  # set to false if  your SSL is terminated at a loadbalancer, for example
  FORCE_HTTPS=true
  
  DEPLOYMENT=self
  ENABLE_UPDATES=true
  SUBDOMAINS_ENABLED=false
  WEBSOCKETS_ENABLED=true
  DEBUG=cache,presenters,events
  
  # Third party signin credentials (at least one is required)
  SLACK_KEY=get_a_key_from_slack
  SLACK_SECRET=get_the_secret_of_above_key
  
  GOOGLE_CLIENT_ID=
  GOOGLE_CLIENT_SECRET=
  
  # Comma separated list of domains to be allowed (optional)
  # If not set, all Google apps domains are allowed by default
  GOOGLE_ALLOWED_DOMAINS=
  
  # Third party credentials (optional)
  SLACK_VERIFICATION_TOKEN=PLxk6OlXXXXXVj3YYYY
  SLACK_APP_ID=A0XXXXXXX
  SLACK_MESSAGE_ACTIONS=true
  GOOGLE_ANALYTICS_ID=
  BUGSNAG_KEY=
  GITHUB_ACCESS_TOKEN=
  
  # AWS credentials (optional in development)
  AWS_ACCESS_KEY_ID=get_a_key_from_aws
  AWS_SECRET_ACCESS_KEY=get_the_secret_of_above_key
  AWS_REGION=xx-xxxx-x
  AWS_S3_UPLOAD_BUCKET_URL=http://s3:4569
  AWS_S3_UPLOAD_BUCKET_NAME=bucket_name_here
  AWS_S3_UPLOAD_MAX_SIZE=26214400
  
  # Emails configuration (optional)
  SMTP_HOST=
  SMTP_PORT=
  SMTP_USERNAME=
  SMTP_PASSWORD=
  SMTP_FROM_EMAIL=
  SMTP_REPLY_EMAIL=
  ```





### 이슈 #1 - 내부 공유로만 변경

* before:

  ```javascript
  // https://github.com/outline/outline/blob/4164fc178cbbfac1d4ce1112103db0de8876ac3a/app/routes.js#L37-L47
  export default function Routes() {
    return (
      <Switch>
        <Route exact path="/" component={Home} />
        <Route exact path="/share/:shareId" component={KeyedDocument} />
        <Authenticated>
          <SocketProvider>
            <Layout>
              <Switch>
                <Route path="/dashboard/:tab" component={Dashboard} />
                <Route path="/dashboard" component={Dashboard} />
  ```

* after

  ```javascript
  // https://github.com/neosapience/outline/blob/8d34dcf099eb0008f96313417d07fe71fb1fb7b9/app/routes.js#L38-L47
  export default function Routes() {
    return (
      <Switch>
        <Route exact path="/" component={Home} />
        <Authenticated>
          <SocketProvider>
            <Layout>
              <Switch>
                <Route exact path="/share/:shareId" component={KeyedDocument} />
                <Route path="/dashboard/:tab" component={Dashboard} />
  ```

  

### 이슈 #2 - 이미지 빌드이슈

* https://hub.docker.com/r/outlinewiki/outline/build

* 배포 하면 될걸로 생각하고, k8s에서 리소스 제약을 걸고 시작함.

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  ...
  spec:
    template:
      spec:
        containers:
          resources:
            requests:
              cpu: 200m
              memory: 500Mi
  ...
  
  ```

* 자꾸 안올라감...

* yarn build && yarn start

  ```dockerfile
  # https://raw.githubusercontent.com/outline/outline/master/Dockerfile
  
  FROM node:12-alpine
  
  ENV PATH /opt/outline/node_modules/.bin:/opt/node_modules/.bin:$PATH
  ENV NODE_PATH /opt/outline/node_modules:/opt/node_modules
  ENV APP_PATH /opt/outline
  RUN mkdir -p $APP_PATH
  
  WORKDIR $APP_PATH
  COPY . $APP_PATH
  
  RUN yarn install --pure-lockfile
  RUN cp -r /opt/outline/node_modules /opt/node_modules
  
  RUN yarn build
  
  CMD yarn start
  
  EXPOSE 3000
  ```

* docker container 배포를 위해서는 빌드후, `yarn start`만 하는 도커 이미지를 만들어서 배포해야함



### 이슈 #3 - ALB SSL Termination

![](img/ssl-termination-diagram.png)

ALB이후로 ssl은 사용하지 않고, http로만 접속. 

하지만,  outline router코드에서 enforce https가 설정되어 있어서 http로 전달될 경우 다시 https로 요청. 

결국 `client` -> **https** -> `alb ssl termination` -> **http** -> `outline server request https` -> **https** -> ...

infinity chain이 되어버림

해결법중 하나는 `x-forwarded-proto=https` 를 셋팅해서 웹서버로 전달해주면 됨.

다른하나는 enforce https옵션을 옵션으로 설정할 수 있게함

issue: https://github.com/outline/outline/issues/1041

pr: https://github.com/outline/outline/pull/1042/files





## Ref:

* https://avinetworks.com/glossary/ssl-termination/