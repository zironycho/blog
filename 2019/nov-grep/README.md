# grep command

k8s쓰면서 grep을 많이 쓰는데, 단순하게 파이프라닝해서 있는지 없는지만 볼 때써서.. 좀더 잘쓸수 있을것 같아서 조금만 알아봄..



Example)

```bash
kubectl describe po -n neos-dev prosody-engine-56bbf6dc48-r8dvl | grep Requests

    Requests:
```

근처의 데이터를 볼수 가 없음..



## grep으로 잡힌부분 앞뒤로 더 보기.

* -A {num} : 찾고나서 다음 {num}라인갯수만큼 출력
* -B {num}: 찾고나서 이전 {num}라인갯수만큼 출력
* -C {num}: 찾고나서 다음 {num}라인, 이전 {num}라인 만큼 출력

 

## grep시 글자관련

* -i: 대소문자 무시
* -w: 워드 풀서치



## grep시 util
* -c: 카운팅
* -e: searching word multiple,  `-e Requests -e Limits`
* -n: 라인넘버출력











