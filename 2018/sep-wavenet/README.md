# Implement efficient wavenet using c++
By [zironycho](http://github.com/zironycho) :heart: [Neosapience, Inc](http://www.neosapience.com), and OSEU
**2018.09.28**

## Estimate Computing power
하나의 CPU로 작업이 가능한가? 

* cpu core (intel skylake, included avx-512)
  * `Lower Bound: 28 cores * 1.7 GHz * 32 FLOPS/Hz = 1523.2 GFLOPS`
  * `1523.2 / 28cores = 54.4 GFLOPS/s ` == `0.054 TFLOPS/s`
* gpu core: titanx maxwell
  * `6 TFLOPS/s`
* R=64, S=256, A=256
  * `0.042 TFLOPS/s`

## Implement base code
1. pytorch코드에서 layer별로 python 테스트코드 작성
2. `python` 사용하는 layer의 일부를 떼어서 데이터 추출용으로 구현
   * 디버깅을 위해서 각각 input, output, weight, bias들을 `사람이 판별하기 쉬운 값`으로 셋팅
     * 0, 1, 2, ...
     * 1, 1, 1, .. 
     * 0, 0, 0, ...
   * 각각의 input, output, weight, bias들을 `text file`로 저장
3. `C/C++` GoogleTest 를 이용해서 각각 layer를 unittest 코드와 함께 작성
   * 각각의 layer들은 forward, set_weight 함수를 지님
   * weight를 로딩하는 부분, 실제로 forward를 하는 부분을 구현
   * matrix계산이나, math function들은 naive하게 구현
   * 메모리를 최대한 한 번만 생성하게 구현
4. `python` random weight, input들을 넣고 output을 얻어내서 저장
5. `C/C++` 테스트가 정상적으로 돌아가는지 체크
   * float형태의 계산결과가 다를 수 있으니, isclose같은 함수를 사용하여 비교

### TIP: shared vs static
* 하나의 binary만 독자적으로 배포하려고 하기 때문에, shared로 할 경우 전체 docker 이미지 사이즈가 커짐.
* static으로 하게 되면 binary사이즈는 커지게 되지만, docker image사이즈는 작음

## References:
* [tasks for fast wavenet implementation](http://on-demand.gputechconf.com/gtc/2017/presentation/s7544-andrew-gibiansky-efficient-inference-for-wavenet.pdf)
* [nv wavenet](https://github.com/NVIDIA/nv-wavenet)
* [deep voice paper](https://arxiv.org/abs/1702.07825)
* [c5 instance with avx-512](https://www.intel.com/content/www/us/en/architecture-and-technology/avx-512-overview.html)
* [c5 instance](https://aws.amazon.com/ec2/instance-types/c5/)
* [max FLOPS on Intel xeon Skylake](https://software.intel.com/en-us/forums/software-tuning-performance-optimization-platform-monitoring/topic/761046)
* [Extending Python with C or C++](https://docs.python.org/3/extending/extending.html)
* [pybind11](https://github.com/pybind/pybind11)
* [mkl vs openblas](https://software.intel.com/en-us/articles/performance-comparison-of-openblas-and-intel-math-kernel-library-in-r)

## Development Envioronments:
- CLion / CMake / C++
- Pycharm / Python
- Docker for linux build and distribution
