# 10Gb network testing
> 연결은 했고, 스위치에서 10G LED도 확인했지만, 실제로 10Gbits/sec 으로 동작하는지 확인하고 싶었음...


* L2 switch: [XS708Ev2 netgear](https://www.netgear.com/images/datasheet/switches/XS708Ev2_XS716E_DS.pdf)
* Leaf node: [DGX workstation](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/dgx-station/nvidia-dgx-station-datasheet.pdf)



## scp

* 초당 170MB/s (1.42Gbits/s)

* 파일 문제인가?? (HDD속도한계인가...)



## bbcp

[download](https://www.slac.stanford.edu/~abh/bbcp/bin/)

- 그닥 속도차이는 없고...
- bbcp 설치및 파일관련되서 좀... 별로인듯.



## iperf(just for testing)

[download](https://iperf.fr/iperf-download.php#more-recent)

* server
  
  ```bash
  $ iperf3 -s -V
  ```
  
* client
  
  ```bash
  $ iperf3 -c $SERVER_IP -V
  ```
  
  * 초당 1.10 GBytes (9.43 Gbits/s)
  
* client with file
  
  ```bash
  $ iperf3 -c $SERVER_IP -V -F large.bin
  ```
  
  * 초당 1.10 GBytes (9.43 Gbits/s)



## fiber vs copper

>  일단 종단 거리는 100M 정도..



### fiber

fiber로 결정하게 될 경우 사야하는게 많고 옵션이 많음...

- Transceiver: 벤더별로 다름...(netgear, dell, cisco, hpe, ...)
  - 2개 사야함(양쪽)
  - type도 다양 (netgear 기준)
    - SR(short range, multimode)
    - LC(long range, singlemode)
    - LRM(long range, multimode)
    - LRLite(long range lite, single mode)
- 반대쪽이 sfp 단자가 없다면, converter도 사야함(fiber - rj45)
- 광케이블



### copper

* 10Gb를 위해서는 cat6a, cat7사면 됨.
  * 100M에 13만원이면 삼...

* cat6는 55m 안에서만 10Gb 가 이론상 됨.






## Ref

https://sola99.tistory.com/374

https://fs.com

https://wbnetworks.com.au/blog/what-is-the-real-difference-between-cat6-and-cat6a