for async webapp

* [Quart](https://gitlab.com/pgjones/quart)
* [Sanic](https://github.com/channelcat/sanic)





### for celery

```bash
celery worker -A task -P eventlet -c 1000
```



### benchmark

* tool: [wrk](https://github.com/wg/wrk)
* task:
  * threads: 12
  * connections: 400
  * 30sec

|                          | sleep 10 sec | w/o sleep |
| ------------------------ | ------------ | --------- |
| gunicorn sync            | 3            | 13,284    |
| gunicorn async(eventlet) | 814          | 35,448    |
| gunicorn async(gevent)   | 842          | 37,746    |
| sanic                    | 817          | 126,538   |
| sanic with gunicorn      |              | 51,573    |



### request with sleep

sync

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.00s     0.00us  10.00s   100.00%
    Req/Sec     0.00      0.00     0.00    100.00%
  3 requests in 30.09s, 462.00B read
  Socket errors: connect 0, read 565, write 2, timeout 2
Requests/sec:      0.10
Transfer/sec:      15.36B
```

async

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.16s   186.02ms  10.54s    73.10%
    Req/Sec    51.00     56.71   210.00     80.56%
  814 requests in 30.09s, 126.39KB read
  Socket errors: connect 0, read 11, write 4, timeout 0
Requests/sec:     27.06
Transfer/sec:      4.20KB
```

gevent

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.06s    70.97ms  10.21s    76.48%
    Req/Sec    52.07     69.17   276.00     79.31%
  842 requests in 30.11s, 131.04KB read
  Socket errors: connect 0, read 67, write 0, timeout 0
Requests/sec:     27.97
Transfer/sec:      4.35KB
```



sanic

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.05s    26.15ms  10.09s    57.04%
    Req/Sec    80.70    104.86   313.00     77.78%
  817 requests in 30.10s, 88.56KB read
  Socket errors: connect 0, read 37, write 19, timeout 0
Requests/sec:     27.14
Transfer/sec:      2.94KB
```



quart

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:5000
Running 30s test @ http://127.0.0.1:5000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    10.28s    87.23ms  10.42s    66.92%
    Req/Sec    62.15     61.42   215.00     69.70%
  792 requests in 30.09s, 100.55KB read
  Socket errors: connect 0, read 153, write 3, timeout 0
Requests/sec:     26.32
Transfer/sec:      3.34KB
```



### request without sleep

sync

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   284.17ms  105.93ms 849.14ms   73.24%
    Req/Sec    39.16     28.47   250.00     68.47%
  13284 requests in 30.10s, 1.95MB read
  Socket errors: connect 0, read 2632, write 57, timeout 0
Requests/sec:    441.31
Transfer/sec:     66.37KB
```



async eventlet

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   241.71ms    1.22s   13.18s    95.58%
    Req/Sec     1.00k   415.96     1.37k    83.43%
  35448 requests in 30.09s, 5.38MB read
  Socket errors: connect 0, read 49, write 7, timeout 136
Requests/sec:   1178.10
Transfer/sec:    182.94KB
```

gevent

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   594.73ms    2.16s   14.98s    92.61%
    Req/Sec     1.15k   386.46     1.43k    86.15%
  37746 requests in 30.07s, 5.72MB read
  Socket errors: connect 0, read 38, write 8, timeout 46
Requests/sec:   1255.10
Transfer/sec:    194.89KB
```



sanic

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:8000
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    90.16ms   21.27ms 239.17ms   79.89%
    Req/Sec   355.86    134.46     0.99k    73.31%
  126538 requests in 30.10s, 13.40MB read
  Socket errors: connect 0, read 386, write 1, timeout 0
Requests/sec:   4203.73
Transfer/sec:    455.68KB
```



sanic with gunicorn

```bash
Running 30s test @ http://127.0.0.1:8000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   229.44ms   49.56ms 499.78ms   83.31%
    Req/Sec   153.22     68.98   673.00     77.98%
  51573 requests in 30.09s, 5.46MB read
  Socket errors: connect 0, read 65, write 11, timeout 0
Requests/sec:   1713.88
Transfer/sec:    185.78KB
```



quart

```bash
zironycho$ wrk -t12 -c400 -d30s --timeout 15  http://127.0.0.1:5000
Running 30s test @ http://127.0.0.1:5000
  12 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   451.31ms  153.36ms 838.22ms   80.86%
    Req/Sec    80.20     66.21   320.00     66.67%
  25670 requests in 30.09s, 3.18MB read
  Socket errors: connect 0, read 244, write 4, timeout 0
Requests/sec:    853.00
Transfer/sec:    108.29KB
```

