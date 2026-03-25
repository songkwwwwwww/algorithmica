---
title: GPU 프로그래밍 (GPU Programming)
weight: 5
---

이 문서는 HTML로 렌더링된 주피터 노트북(Jupyter notebook)입니다. 여기서 직접 실습을 해보고 싶다면 [Colab]()에서 열거나 [다운로드]()하여 로컬에서 편집하세요. 전자의 경우, 약간의 단계를 거쳐 CUDA와 파이썬 바인딩인 PyCuda를 설치해야 합니다. 데비안 기반 머신(Debian-based machine)에서는 다음 명령어로 충분할 것입니다: 
* `apt-get install nvidia-cuda-dev nvidia-cuda-toolkit`
* `pip install pycuda`

선수 지식: 파이썬과 C에 대한 기초 지식, 기초 알고리즘, 그리고 컴퓨터가 일반적으로 어떻게 작동하는지에 대한 이해.

## 무어의 법칙의 미묘한 점들

다음은 CPU 세계에서 일어나고 있는 일을 대략적으로 나타내는 그래프입니다:

<img width='600px' src='https://www.karlrupp.net/wp-content/uploads/2015/06/35years.png'>

**무어의 법칙(Moore's law)**은 마이크로프로세서의 트랜지스터 수가 약 2년마다 두 배로 증가한다는 관찰 결과입니다. 이는 대략적으로 성능도 두 배가 된다는 것을 의미합니다. 

2005년경에 설계의 변화가 있었음을 알 수 있습니다.

코어들은 어느 정도 독립적입니다.

현대적인 GPU는 2000년대 초반에 등장했습니다. GPU는 작동하는 특정 영역을 활용합니다.

코어 속도에는 물리적인 한계가 있습니다.

그 중 하나는 빛의 속도입니다. 적어도 전자기파(이 또한 빛입니다)가 마더보드의 한쪽 끝에서 다른 쪽 끝으로 이동하는 데 걸리는 시간이 필요합니다.

그들 중 일부는...

구글 코랩(Google Colab)에서 제공하는 기본 무료 GPU는 [꽤 강력합니다](https://www.nvidia.com/content/dam/en-zz/Solutions/Data-Center/tesla-t4/t4-tensor-core-datasheet-951643.pdf). 저자는 왜 구글이 이런 일을 하는지 모르겠지만, 정말 멋진 일입니다.

## 왜 멀티프로세싱인가?

클록 주파수 — 예를 들어, 인텔 코어 i7은... 이것이 상한선을 제공합니다.

두 가지 유형의...

## 범용 GPU (General-purpose GPU)

헤지펀드들이 게임 회사의 컴퓨터 그래픽 전문가들을 그들의 계산 능력 때문에 고용하던 시절이 있었습니다.

여러 가지가 있습니다.

이는 윈도우와 리눅스의 관계와 비슷합니다.

우리는 CUDA를 사용할 것입니다. 왜냐하면 딥러닝과 같이 사람들이 크게 신경 쓰지 않는 분야에서 더 널리 퍼져 있기 때문입니다.

## 이기종 컴퓨팅 (Heterogeneous computing)

CUDA 프로그래밍은 하나 이상의 CPU가 있는 호스트 시스템과 하나 이상의 GPU라는 두 개의 서로 다른 플랫폼에서 코드를 동시에 실행하는 것을 포함합니다.

## CPU와의 차이점

### 스레드 (Threads)

CPU의 스레드는 일반적으로 무거운 개체(heavyweight entities)입니다. 운영체제는 멀티스레딩 기능을 제공하기 위해 CPU 실행 채널에서 스레드를 교체(swap)해야 합니다. 따라서 컨텍스트 스위치(Context switch)는 느리고 비용이 많이 듭니다.

이에 비해 GPU의 스레드는 매우 가볍습니다. 일반적인 시스템에서는 수천 개의 스레드가 작업을 위해 대기하며, 각각 32개의 스레드로 구성된 워프(warp) 단위로 처리됩니다. GPU가 한 워프의 스레드를 기다려야 하는 경우, 단순히 다른 워프의 작업을 실행하기 시작합니다. 모든 활성 스레드에 별도의 레지스터가 할당되므로, GPU 스레드 간을 전환할 때 레지스터나 다른 상태를 교체할 필요가 없습니다. 리소스는 실행이 완료될 때까지 각 스레드에 할당된 상태로 유지됩니다.

요약하자면, CPU 코어는 한 번에 하나 또는 두 개의 스레드에 대한 지연 시간(latency)을 최소화하도록 설계된 반면, GPU는 처리량(throughput)을 최대화하기 위해 수많은 동시 경량 스레드를 처리하도록 설계되었습니다.

### 메모리 (Memory)

호스트 시스템과 디바이스는 각각 별도로 부착된 물리적 메모리를 가집니다. 호스트와 디바이스 메모리는 PCI 익스프레스(PCIe) 버스로 분리되어 있으므로, 호스트 메모리의 아이템은 가끔 버스를 통해 디바이스 메모리로 전달되어야 하며 그 반대의 경우도 마찬가지입니다. (What Runs on a CUDA-Enabled Device? 에서 설명됨)

이런 식으로 생각하면 성능의 98%를 쉽게 버리게 될 수 있습니다.

## PyCUDA 설치하기

CUDA는 많은 언어에서 사용할 수 있습니다.

좋은 문서는 여기서 찾을 수 있습니다: https://documen.tician.de/pycuda/index.html

코랩(Colab)을 사용 중이라면, 런타임 -> 런타임 유형 변경 -> 하드웨어 가속기로 이동하여 "GPU"로 설정하세요.


```python
# 설치 후 이 셀의 출력을 지우는 것이 좋습니다.
from IPython.display import clear_output
 
# 시간이 좀 걸릴 수 있습니다.
!pip install pycuda

clear_output()
```


```python
import numpy as np

from pycuda.compiler import SourceModule
import pycuda.driver as drv
import pycuda.autoinit
```

## 기초 (The basics)

간단한 예제로 시작해서 더 깊이 들어가 보겠습니다.

## 커널 (Kernels)

몇 가지 커스텀 내장 함수와 지정자를 사용한다는 점을 제외하면 C 또는 C++와 거의 같습니다.

CUDA는 일반적인 C와 거의 비슷하지만, 특정 함수가 실행되도록 지정할 수 있습니다. 구현에 따라 워크플로우는 다음과 같이 진행됩니다:

컴퓨터를 이기종 머신으로 생각해야 합니다. 호스트 데이터와 디바이스 데이터가 있습니다.

* 입력 데이터를 디바이스 메모리로 이동합니다.
* 디바이스에서 계산을 수행합니다.
* 이 데이터를 다시 가져옵니다.

사실 커널 실행은 병렬로 이루어집니다. 커널 실행이 완료될 때까지 프로그램이 차단(block)되지 않습니다. 최신 디바이스는 이런 식으로 여러 커널을 동시에 실행하고 그 결과를 기다릴 수도 있습니다.

## 유명한 $A + B$ 문제

테스트 및 호스트와의 조정을 위해 **NumPy** 패키지를 사용하겠습니다. 설치되어 있지 않다면 `pip install numpy`로 설치하세요.

NumPy는 파이썬에서 선형 대수와 배열 조작을 위한 패키지입니다. C로 작성되어 매우 효율적이지만 오직 CPU에서만 실행되므로, 이를 벤치마크 대상으로 삼겠습니다.


```python
# 테스트 데이터를 생성합니다: 랜덤하게 채워진 두 개의 float 배열
a = numpy.random.randn(100).astype('float32')
b = numpy.random.randn(100).astype('float32')
# randn의 기본 타입은 float64이지만 CUDA는 이를 알지 못하므로 타입을 명시해야 합니다.

# 커널이 결과를 기록할 공간을 만듭니다.
dest = numpy.zeros_like(a)

# 이것이 커널 자체입니다.
mod = SourceModule("""
    __global__ void add(float *dest, float *a, float *b) {
        const int i = threadIdx.x;
        dest[i] = a[i] + b[i];
    }
""")

# 소스 코드를 지정하면 PyCUDA가 이를 컴파일합니다.
add_kernel = mod.get_function("add")

add_kernel(
    drv.Out(dest),  # 이 메모리가 쓰기 가능함을 지정합니다.
    drv.In(a),  # 이 메모리가 읽기 가능함을 지정합니다.
    drv.In(b),
    block=(100,1,1)  # 이에 대해서는 잠시 후에 설명하겠습니다.
)

assert np.allclose(dest, a + b), 'WA'  # 두 값이 같은지 확인합니다.
print('OK')
```


      File "<ipython-input-27-afc857479fe4>", line 19
        %%time
        ^
    SyntaxError: invalid syntax



### 메모리 관리 (Memory management)

CUDA C API에서는 메모리를 명시적으로 할당해야 합니다. 이것은 사실 꽤 괜찮은 방식입니다.

`drv.InOut` 함수도 있는데, 이는 읽기와 쓰기 모두 가능하게 합니다. 하지만 이 튜토리얼에서는 코드 테스트를 위해 사용하지 않겠습니다.

여기서 대부분의 연산은 메모리 연산이므로 성능 측정은 의미가 없습니다. 걱정하지 마세요. 곧 더 복잡한 예제를 다룰 것입니다.

GPU는 매우 특정한 연산을 수행합니다. 하지만 NVIDIA GPU의 경우 관리가 매우 간단합니다. 카드에는 *컴퓨팅 능력(compute capabilities)* (1.0, 1.1, 1.2, 1.3, 2.0 등)이 있으며, 버전 $x$에서 추가된 모든 기능은 이후 버전에서도 사용할 수 있습니다. 이는 실행 시간이나 컴파일 시간에 확인할 수 있습니다.

차이점은 이 위키피디아 문서에서 확인할 수 있습니다: https://en.wikipedia.org/wiki/CUDA#Version_features_and_specifications

## 동기화 (Synchronization)

**리덕션(Reduction)**은 모든 배열 단위 연산을 의미합니다.

다음 문제를 가정해 봅시다:


## 동적 계획법 (Dynamic programming)

다음 점화식을 고려해 보세요:


```python
## 문제: 동적 계획법
```

## 작업량 vs 지연 시간 (Work vs. Latency)

이제 작업량(work)과 단계 복잡도(step complexity)를 모두 고려해야 합니다.

일부 작업, 특히 암호학 분야의 작업은 병렬화할 수 없습니다. 하지만 가능한 작업들도 있습니다.

## $O(\log n)$ 시간에 배열 합 구하기

$n$개의 원소를 가진 배열에 대해 어떤 결합 법칙(associative, 즉 $A*(B*C) = (A*B)*C$)이 성립하는 연산을 수행하고 싶다고 가정해 봅시다. 예를 들어 합계를 구하는 것입니다.

보통은 다음과 같이 간단한 루프로 처리합니다:

```c++
float s = 0;
for (int i = 0; i < n; i++) {
     s += a[i]; 
}
```

이의 계산 그래프는 다음과 같습니다:

<img width='400px' src='https://www.elemarjr.com/wp-content/uploads/2018/03/sequential_sum.png'>

이것은 작업 복잡도 측면에서는 최적이지만, 단계 복잡도 측면에서는 최적이 아닙니다. $O(n)$이기 때문입니다. 작업 복잡도는 약간 나쁘더라도 병렬화할 수 있는 방식을 원할 수 있습니다.

분할 정복(divide-and-conquer) 접근 방식을 시도해 봅시다:

<img width='400px' src='https://www.elemarjr.com/wp-content/uploads/2018/03/parallel_sum.png'>

여전히 $O(n)$ 작업 복잡도를 가지지만(실제로 동일한 횟수의 덧셈이 필요함), 단계 복잡도는 $O(\log n)$입니다.

재귀를 위에서 아래로 풀어서 보면, 필요한 각 값을 얻기 위해...

<img width='400px' src='http://i.stack.imgur.com/Uehc3.png'>

## 작은 배열 리덕션하기 (Reducing small arrays) 


```python
a = numpy.random.randn(2048).astype('float32')

mod = SourceModule("""
    __global__ void sum(float *dest, float *a, float *b) {
        const int i = threadIdx.x;
        // 0부터 logn까지 l에 대해:
        //   __sync_threads()
        //   스레드가 활성 상태이면
        //     두 원소를 제자리에 더함
        // a[0]에 최종 합계가 포함되어야 함
    }
""")

sum_kernel = mod.get_function("sum")

add_kernel(
    drv.InOut(a),
    block=(1024,1,1)
)

assert np.allclose(dest, a + b), 'WA'  # 두 값이 같은지 확인합니다.
print('OK')
```

## 워프와 스레드 블록 (Warps and thread blocks)

스레드는 32개씩 그룹으로 묶입니다. 한 그룹 내의 모든 스레드는 대기 중이거나 동일한 연산을 수행해야 합니다. 이는 아키텍처상의 제약 때문입니다.

<img width='300px' src='https://upload.wikimedia.org/wikipedia/commons/thumb/5/5b/Block-thread.svg/1920px-Block-thread.svg.png'>

실제로 2D 및 3D 인덱싱으로도 동일한 작업을 수행할 수 있습니다. 신기하죠?

## 원자적 연산 (Atomics)

## 큰 배열 리덕션하기

## 매우 큰 배열 리덕션하기

이제 상황이 더 어려워집니다. GPU 병렬 처리가 정확히 어떻게 작동하는지 설명할 때입니다.




```python

```

## 밀집 행렬 곱셈 (Dense Matrix multiplication)

GPU를 사용하는 것이 실제로 의미 있는 첫 번째 예제인 행렬 곱셈을 살펴보겠습니다.

## 정렬 (Sorting)

마지막(그리고 가장 어려운) 과제는 정렬을 구현하는 것입니다.

우리가 대부분의 경우 분할 정복 접근 방식을 옹호했다는 것을 눈치채셨을 것입니다.

사실입니다. 효과가 있거든요. 하지만 바로 작동하는 알고리즘을 얻을 수는 없습니다.


```python
# 벤치마킹을 위해 딥러닝 라이브러리를 사용하겠습니다. 다른 것에 익숙하지 않거든요.
import torch

a = torch.randn(10**8)
b = a.cuda()
```


```python
# 이 코드는 약 15초 동안 실행됩니다.
%time c = torch.sort(a)
%time c = torch.sort(b)
```

    CPU times: user 15.2 s, sys: 177 µs, total: 15.2 s
    Wall time: 15.2 s
    CPU times: user 274 ms, sys: 237 ms, total: 511 ms
    Wall time: 511 ms


30배의 속도 향상이 있습니다. 이제 우리가 경쟁해야 할 목표를 알았습니다.


```python
b.sort()
```




    (tensor([-5.4567, -5.3551, -5.3288,  ...,  5.3529,  5.4484,  5.4486],
            device='cuda:0'),
     tensor([55083205,  8383169, 73705953,  ..., 79814161, 50474932, 27805828],
            device='cuda:0'))



정렬 알고리즘에는 두 가지 유형이 있습니다: 데이터 주도형(data-driven)과...

두 번째 유형은 정렬 네트워크(sorting networks)로 표현하고 분석할 수 있습니다. 여기서 우리가 사용할 것은 바이토닉 정렬(bitonic sort)입니다.

<img src='https://upload.wikimedia.org/wikipedia/commons/thumb/c/c6/BitonicSort.svg/1686px-BitonicSort.svg.png'>

이는 $O(\log n)$개의 단계를 가지며, 총 $1 + 2 + 3 + \ldots + \log n = O(\log^2 n)$개의 비교 블록이 있습니다. 이들은 병렬화할 수 없으며 배열의 모든 원소를 포함합니다. 따라서 총 작업 복잡도는 $O(n \log^2 n)$이지만 단계 복잡도는 $O(\log^2 n)$으로 꽤 괜찮습니다.

구현하는 것이 그렇게 어렵지는 않습니다. 이해를 돕기 위해 여기 느린 재귀 파이썬 구현이 있습니다:


```python
def bitonic_sort(a, up=False):
    if len(a) <= 1:
        return a
    else: 
        l = bitonic_sort(x[:len(a) // 2], True)
        r = bitonic_sort(x[len(a) // 2:], False)
        return bitonic_merge(first + second, up)

def bitonic_merge(a, up): 
    # 입력 a가 바이토닉이라고 가정하고 정렬된 리스트를 반환함
    if len(a) == 1:
        return a
    else:
        bitonic_compare(a, up)
        l = bitonic_merge(a[:len(a) // 2], up)
        r = bitonic_merge(a[len(a) // 2:], up)
        return l + r

def bitonic_compare(a, up):
    dist = len(a) // 2
    for i in range(dist):  
        if (a[i] > a[i + dist]) == up:
            a[i], a[i + dist] = a[i + dist], x[i]  # 파이썬에서 스왑이 이루어지는 방식
```


```python
bitonic_sort([57, 179, 42, 17, 300, 111])
```




    [300, 179, 111, 57, 42, 17]




```python
a = np.random.randn(10**8).astype('float32')
```


    ---------------------------------------------------------------------------

    NameError                                 Traceback (most recent call last)

    <ipython-input-27-58a927c14aae> in <module>()
    ----> 1 a = np.random.randn(10**8).astype('float32')
    

    NameError: name 'np' is not defined


## 왜 CUDA인가

대부분은 여전히 적용 가능합니다.

다시 말하지만, GPU 프로그래밍은 매우 특수합니다.

SSE와 텐서 코어(tensor cores).

## 커널 (Kernels)

몇 가지 커스텀 내장 함수와 지정자를 사용한다는 점을 제외하면 C 또는 C++와 거의 같습니다.

CUDA는 일반적인 C와 거의 비슷하지만, 특정 함수가 실행되도록 지정할 수 있습니다. 구현에 따라 워크플로우는 다음과 같이 진행됩니다:

컴퓨터를 이기종 머신으로 생각해야 합니다. 호스트 데이터와 디바이스 데이터가 있습니다.

* 입력 데이터를 디바이스 메모리로 이동합니다.
* 디바이스에서 계산을 수행합니다.
* 이 데이터를 다시 가져옵니다.

사실 커널 실행은 병렬로 이루어집니다. 커널 실행이 완료될 때까지 프로그램이 차단(block)되지 않습니다. 최신 디바이스는 이런 식으로 여러 커널을 동시에 실행하고 그 결과를 기다릴 수도 있습니다.

GPU에 대해 이해해야 할 점은 GPU가 해당 애플리케이션에 매우 특화되어 있다는 것입니다.

이를 위한 인트린직(Intrinsics).

이제 많은 가치가 암호화폐와 딥러닝에서 나옵니다. 후자는 두 가지 특정 연산에 의존합니다: 선형 레이어를 위한 행렬 곱셈과 컴퓨터 비전에 사용되는 컨볼루션 레이어를 위한 컨볼루션입니다.

첫째, 그들은 1 GPU 클록 사이클당 "곱셈-누적(multiply-accumulate)" 연산(예: `x += y * z`)을 도입했습니다.

구글은 텐서 처리 장치(Tensor Processing Units, TPU)를 사용합니다. 아무도 그것들이 어떻게 작동하는지 모릅니다 (판매하지 않고 대여만 하는 독점 하드웨어입니다).

각 텐서 코어는 4x4 크기의 작은 행렬에 대해 연산을 수행합니다. 각 텐서 코어는 1 GPU 클록당 1번의 행렬 곱셈-누적 연산을 수행할 수 있습니다. 두 개의 fp16 4x4 행렬을 곱하고 곱셈 결과인 fp32 행렬(크기: 4x4)을 누적기(마찬가지로 fp32 4x4 행렬)에 더합니다.

이는 클록당 엄청난 양의 작업입니다.

글쎄요, 딥러닝을 위해서라면 이보다 더 정밀한 것은 사실 필요 없습니다.


입력 행렬은 fp16이지만 곱셈 결과와 누적기는 fp32 행렬이기 때문에 이를 혼합 정밀도(mixed precision)라고 부릅니다.

아마도 적절한 이름은 "4x4 행렬 코어"였겠지만, NVIDIA 마케팅 팀은 "텐서 코어"라는 이름을 사용하기로 결정했습니다.

그러니 보시다시피 이것은 정확히 공정한 비교는 아닙니다.

<img width='500px' src='https://static.seekingalpha.com/uploads/2018/8/11/275308-15340093003448672_origin.png'>

*<center>이 그래프를 조금만 더 확장해 보세요: 지난 11월, 비트코인 폭락 이후 NVIDIA의 주가는 30% 하락했으므로 너무 희망적일 필요는 없습니다.</center>*

int4까지 (16개 값, 제대로 들으셨습니다)

효율적인 코드를 작성하려면 이러한 전문 지식을 많이 알아야 합니다. 따라서 라이브러리를 처음부터 작성하는 것은 좋지 않은 생각입니다.

어쨌든 교육적이고 유희적인 이유로, 오늘은 바퀴를 다시 발명하여 행렬 곱셈을 해보겠습니다.

## 배열 리덕션하기

간단해 보입니다: 그저 ...하면 됩니다.

`s += x`를 할 때 실제로 어떤 일이 일어날까요? 이것은 단일 연산이 아닙니다. 실제로는 네 가지 일이 일어납니다:

1. $x$를 레지스터로 읽어옴
2. $s$를 레지스터로 읽어옴
3. $s + x$를 계산함
4. 원래 $s$가 있던 곳에 다시 씀

두 스레드가 이를 교차해서 실행할 수 있습니다. 예를 들어 스레드 A가 $s$를 가져왔는데, 1나노초 후에 스레드 B가 여기에 쓰고 있다면, 스레드 A는 이를 알지 못하고 변경되지 않은 값을 덮어쓰게 됩니다.



참고: 이를 수행하기 위한 원자적 연산(Atomics)

작은 데이터 타입의 경우 하드웨어 수준에서 구현되어 훨씬 빠릅니다.

멀티스레드 컨텍스트에서 원자적 연산을 처리하기 위해 `std::atomic`이 도입되었습니다. 멀티스레드 환경에서 두 스레드가 동일한 변수에 대해 작업할 때 경주 조건(race condition)을 피하기 위해 각별히 주의해야 합니다. 

## 메모리 유형 (Memory types)

다양한 유형의 디바이스 메모리가 경주를 한다면 다음과 같은 결과가 나올 것입니다:

레지스터 크기(= 머신 워드 너비)는 32비트이지만, 64비트 기능도 포함하고 있습니다(그렇지 않으면 4GB 이상의 메모리를 가질 수 없습니다).

* 1위: **레지스터 메모리 (Register memory)**
  <br> 이 데이터는 그것을 쓴 스레드에게만 보입니다. 해당 스레드의 수명 동안만 유지됩니다.
* 2위: **공유 메모리 (Shared Memory)**
  <br> 스레드 블록 내의 모든 스레드에 공유됩니다. 해당 블록의 수명 동안만 유지됩니다. 이 유형의 메모리는 스레드 간의 통신(데이터 공유)을 가능하게 합니다. 이것이 여러분이 ...해야 하는 이유입니다.
* 3위: **상수 메모리 (Constant Memory)**
  <br> 
* 4위: 텍스처 메모리 (Texture Memory)
* 공동 꼴찌: 로컬 메모리(Local Memory) 및 글로벌 메모리(Global Memory)

지금 신경 써야 할 것은 레지스터...

지금은 차이점에 대해 신경 써야 합니다.

글로벌 메모리에 액세스하는 데는 수백...이 걸립니다.

## 문제: 밀집 행렬 곱셈

이들 중 많은 수가 실제로는 희소(sparse)합니다. 소셜 네트워크 그래프나 웹 그래프로 작업을 수행할 수 있습니다.

좋습니다. 하지만 잠시 우리를 실망시켜 봅시다:
