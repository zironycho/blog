[Home](/README.md)

# sub directory를 위한 S3 프록시 만들어보기

### Problem: Blob으로 zip 파일을 저장해서 <a>태그를 이용해서 다운로드 하면 특정 PC에서 다운로드가 안됨...

그래서 아래처럼 download attribute 를 주고 audio/zip파일을 직접 url로 링크하려고함.

```html
<a href="/images/myw3schoolsimage.jpg" download="w3logo">
```

일반적인 s3: https://my-bucket.s3.amazon.com/dsdd

타캐: https://typecast.ai/

Same origin이 아님.



### same origin이 아닐경우 이슈:

* 이름변경이 안됨
* 창 닫을때 popup message지 출력이 있는 상태에서 다운로드시 팝업이 뜸
* 위에 것을 피해가려고, `_blank` 옵션
  * 팝업차단시 팝업허용해줘야 하나 눈에 안띰
  * 작전: 팝업을 임시로 해보고 안뜨면 계속해서 하라고 강요할수 있음.



### 간단하게 subdirectory로 proxy 서버 운영하기로 결정.

s3리소스를 same origin으로 주기 위해서는 frontend와 동일한 hostname을 사용해야함. [링크](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy)

ex) https://typecast.ai/my-s3-bucket/...

단점: 기존의 s3로 트래픽을 줄때는 우리가 트래픽 부하를 신경안써도 되었음. 근데 프록시를 통해야 하니 s3에 가는 트래픽이 모두 프록시가 받게 되어서 이부분에 대한 추가 서비스 운영해야함

#### Nginx docker: 1.19.0를 이용해 운영

* docker 지원해주려고.. 컨테이너 런타임시 env컨버팅해주는것 넣어줌.. 
* 간단하게 한번 만들어봄: https://github.com/neosapience/s3-proxy
* s3에 presigned url을 사용할 것이기에 aws secret을 proxy에 넣을 필요는 없음
* 환경변수로 `AWS_BUCKET`만 넣어주면 일단 동작하는데는 문제 없음.
* 하지만 대부분 sub directory(ex. https://typecast.ai/sub/)로 사용할 것이기에 `PATH_PREFIX` 환경변수에 /sub/를 같이 넣어주면 문제없이 subdomain에서 사용가능

