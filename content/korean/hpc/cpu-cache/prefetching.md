---
title: 프리페칭 (Prefetching)
weight: 6
---

메모리 하드웨어에서 사용 가능한 [자유로운 동시성](../mlp)을 활용하여, 다음에 액세스할 데이터의 위치를 예측할 수 있다면 해당 데이터를 *프리페치(prefetch)*하는 것이 유익할 수 있습니다. 파이프라인에 [데이터나 제어 해저드(hazards)](/hpc/pipelining/hazards)가 없을 때는 CPU가 명령어 스트림을 앞서 나가 비순차적으로 메모리 연산을 실행할 수 있어 이 작업이 쉽습니다.

하지만 때로는 메모리 위치가 명령어 스트림에 직접 나타나지 않음에도 불구하고 높은 확률로 예측 가능할 때가 있습니다. 이런 경우 다른 수단을 통해 프리페치할 수 있습니다:

- 명시적으로, 다음 데이터 워드나 동일한 캐시 라인의 바이트를 별도로 읽어 캐시 계층으로 끌어올립니다.
- 암시적으로, 선형 반복과 같이 하드웨어가 감지할 수 있는 단순한 액세스 패턴을 사용하여 자동으로 프리페칭이 시작되도록 합니다.

메모리 지연 시간을 숨기는 것은 성능 달성에 결정적이므로, 이번 섹션에서는 프리페칭 기법들을 살펴보겠습니다.

### 하드웨어 프리페칭 (Hardware Prefetching)

하드웨어 프리페칭의 효과를 보여주기 위해 [포인터 추적](../latency) 벤치마크를 수정해 보겠습니다. 이제 순열을 생성할 때, 순열을 따라 반복할 때 CPU가 연속적인 캐시 라인을 요청하도록 하되, 캐시 라인 내부의 요소들은 여전히 무작위 순서로 액세스하도록 만듭니다:

```cpp
int p[15], q[N];

iota(p, p + 15, 1);

for (int i = 0; i + 16 < N; i += 16) {
    random_shuffle(p, p + 15);
    int k = i;
    for (int j = 0; j < 15; j++)
        k = q[k] = i + p[j];
    q[k] = i + 16;
}
```

그래프를 그릴 필요가 없습니다. 결과가 평평하기 때문입니다: 배열 크기와 관계없이 지연 시간은 3ns입니다. 비록 명령어 스케줄러가 다음에 무엇을 가져올지 여전히 알 수 없지만, 메모리 프리페처는 메모리 액세스 양상을 보고 패턴을 감지하여 다음 캐시 라인을 미리 로드하기 시작함으로써 지연 시간을 완화합니다.

하드웨어 프리페칭은 대부분의 사례에서 충분히 똑똑하지만, 단순한 패턴만 감지할 수 있습니다. 여러 배열을 정방향 또는 역방향으로 병렬 반복하는 정도(아마도 작거나 중간 정도의 스트라이드 포함)가 한계입니다. 그보다 복잡한 경우 프리페처는 무슨 일이 일어나고 있는지 파악하지 못하며, 우리가 직접 도와주어야 합니다.

### 소프트웨어 프리페칭 (Software Prefetching)

소프트웨어 프리페칭을 수행하는 가장 간단한 방법은 `mov`나 다른 메모리 명령어로 캐시 라인의 아무 바이트나 로드하는 것이지만, CPU에는 데이터를 실제로 사용하지 않고 캐시 라인만 끌어올리는 별도의 `prefetch` 명령어가 있습니다. 이 명령어는 C나 C++ 표준의 일부는 아니지만, 대부분의 컴파일러에서 `__builtin_prefetch` 인트린직으로 사용할 수 있습니다:

```c++
__builtin_prefetch(&a[k]);
```

이것이 유용한 *단순한* 예제를 찾기는 꽤 어렵습니다. 포인터 추적 벤치마크가 소프트웨어 프리페칭의 이득을 보게 하려면, 전체 배열을 순환하면서도 하드웨어 프리페처가 예측할 수 없고 다음 주소를 쉽게 계산할 수 있는 순열을 구성해야 합니다.

다행히 [선형 합동 생성기(linear congruential generator, LCG)](https://en.wikipedia.org/wiki/Linear_congruential_generator)는 모듈러스 $n$이 소수이면 생성기의 주기가 정확히 $n$이 되는 성질이 있습니다. 따라서 현재 인덱스를 상태로 사용하는 LCG로 생성된 순열을 사용하면 필요한 모든 조건을 충족할 수 있습니다:

```cpp
const int n = find_prime(N); // N을 초과하지 않는 가장 큰 소수

for (int i = 0; i < n; i++)
    q[i] = (2 * i + 1) % n;
```

이를 실행하면 일반적인 무작위 순열과 성능이 일치합니다. 하지만 이제 우리는 앞을 내다볼 수 있는 능력을 갖게 되었습니다:

```cpp
int k = 0;

for (int t = 0; t < K; t++) {
    for (int i = 0; i < n; i++) {
        __builtin_prefetch(&q[(2 * k + 1) % n]);
        k = q[k];
    }
}
```

다음 주소를 계산하는 데 약간의 오버헤드가 발생하지만, 배열이 충분히 크면 거의 두 배 더 빠릅니다:

![](../img/sw-prefetch.svg)

흥미롭게도, LCG 함수의 패턴을 이용하여 단 하나가 아니라 더 앞선 요소를 프리페치할 수도 있습니다:

$$
\begin{aligned}
   f(x)   &= 2 \cdot x + 1
\\ f^2(x) &= 4 \cdot x + 2 + 1
\\ f^3(x) &= 8 \cdot x + 4 + 2 + 1
\\ &\ldots
\\ f^k(x) &= 2^k \cdot x + (2^k - 1)
\end{aligned}
$$

따라서 `D`번째 앞의 요소를 로드하려면 다음과 같이 할 수 있습니다:

```cpp
__builtin_prefetch(&q[((1 << D) * k + (1 << D) - 1) % n]);
```

매 반복마다 이 요청을 실행하면 평균적으로 `D`개 앞의 요소를 동시에 프리페치하게 되어 처리량이 `D`배 증가합니다. `D`가 너무 큰 때의 정수 오버플로와 같은 문제들을 무시한다면, 평균 지연 시간을 다음 인덱스를 계산하는 비용(이 경우 [나머지 연산](/hpc/arithmetic/division)이 지배적임)에 임의로 가깝게 줄일 수 있습니다.

![](../img/sw-prefetch-others.svg)

이것은 인위적인 예제이며, 실제 프로그램에 소프트웨어 프리페칭을 삽입하려고 할 때 성공보다는 실패할 확률이 더 높다는 점에 유의하십시오. 주로 다른 명령어들과 리소스를 경쟁할 수 있는 별도의 메모리 명령어를 발행해야 하기 때문입니다. 반면 하드웨어 프리페칭은 메모리와 캐시 버스가 한가할 때만 활성화되므로 100% 무해합니다.

소프트웨어 프리페칭을 할 때 데이터가 도달해야 할 특정 캐시 계층을 지정할 수도 있습니다 — 해당 데이터를 사용할지 확실하지 않거나 이미 L1 캐시에 있는 것을 쫓아내고 싶지 않을 때 유용합니다. 두 번째 파라미터로 캐시 계층을 지정하는 정수 값을 받는 `_mm_prefetch` 인트린직을 사용하면 됩니다. 이는 [비임시 로드 및 스토어](../bandwidth#bypassing-the-cache)와 결합하여 유용하게 쓰일 수 있습니다.

<!--

In the bandwidth benchmark, we iterated over array and fetched its elements. Although separately each memory read in that case is not different from the fetch in pointer chasing, they run much faster because they can are overlapped: and in fact, CPU issues read requests in advance without waiting for the old ones to complete, so that the results come about the same time as the CPU needs them.

Apart from having a very large pipeline and using the fact that scheduler can look ahead in it, modern memory controllers can detect simple patterns such as iterating backwards, forwards, including using constant small-ish strides.

### Speculative Execution

In fact, this sometimes works even when we are not sure which instruction is going to be executed next due to [speculative execution]. Consider the following example:

```cpp
bool cond = some_long_memory_operation();

if (cond)
    do_this_fast_operation();
else
    do_that_fast_operation();
```

What most modern CPUs do is they start evaluating one (most likely) branch without waiting for the condition to be computed. If they are right, then you will progress faster, and if they are wrong, the worst thing will happen is they discard some useless computation. This includes memory operations too, including cache system — because, well, we wait for a hundred cycles anyway, why not evaluate at least one of the branches ahead of time. By the way, this is what Meltdown was all about.

-->
