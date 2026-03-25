---
title: 마스킹과 블렌딩 (Masking and Blending)
weight: 4
---

SIMD 프로그래밍의 큰 과제 중 하나는 제어 흐름(control flow) 옵션이 매우 제한적이라는 것입니다. 벡터에 적용하는 연산이 모든 요소에 대해 동일하기 때문입니다.

이로 인해 보통 `if`나 다른 유형의 분기(branching)를 통해 사소하게 해결되는 문제들이 훨씬 어려워집니다. SIMD에서는 이를 [분기 없는 프로그래밍(branchless programming)](/hpc/pipelining/branchless) 기술을 사용하여 처리해야 하며, 이러한 기술을 적용하는 것이 항상 간단하지는 않습니다.

### 마스킹 (Masking)

계산을 분기 없이 만드는 주요 방법은 **프리디케이션(predication)**을 통하는 것입니다. 즉, 두 분기의 결과를 모두 계산한 다음 산술적인 트릭이나 특수한 "조건부 이동(conditional move)" 명령을 사용하는 것입니다.

```c++
for (int i = 0; i < N; i++)
    a[i] = rand() % 100;

int s = 0;

// 분기 사용:
for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];

// 분기 없음:
for (int i = 0; i < N; i++)
    s += (a[i] < 50) * a[i];

// 이것도 분기 없음:
for (int i = 0; i < N; i++)
    s += (a[i] < 50 ? a[i] : 0);
```

이 루프를 벡터화하려면 두 가지 새로운 명령이 필요합니다.

- `_mm256_cmpgt_epi32`: 두 벡터의 정수를 비교하여 첫 번째 요소가 두 번째 요소보다 크면 모든 비트가 1인 마스크를 생성하고, 그렇지 않으면 모든 비트가 0인 마스크를 생성합니다.
- `_mm256_blendv_epi8`: 제공된 마스크를 기반으로 두 벡터의 값을 혼합(blend)합니다.

벡터의 요소를 마스킹하고 블렌딩하여 계산에 의해 선택된 하위 집합만 영향을 받게 함으로써, 조건부 이동과 유사한 방식으로 프리디케이션을 수행할 수 있습니다.

```c++
const reg c = _mm256_set1_epi32(49);
const reg z = _mm256_setzero_si256();
reg s = _mm256_setzero_si256();

for (int i = 0; i < N; i += 8) {
    reg x = _mm256_load_si256( (reg*) &a[i] );
    reg mask = _mm256_cmpgt_epi32(x, c);
    x = _mm256_blendv_epi8(x, z, mask);
    s = _mm256_add_epi32(s, x);
}
```

([수평 합(horizontal summation)과 배열의 나머지 처리](../reduction)와 같은 사소한 세부 사항은 간결함을 위해 생략되었습니다.)

이것이 SIMD에서 프리디케이션을 수행하는 일반적인 방법이지만, 항상 가장 최적의 접근 방식은 아닙니다. 블렌딩된 값 중 하나가 0이라는 사실을 이용하여 블렌딩 대신 마스크와 비트 단위 `and`를 사용할 수 있습니다.

```c++
const reg c = _mm256_set1_epi32(50);
reg s = _mm256_setzero_si256();

for (int i = 0; i < N; i += 8) {
    reg x = _mm256_load_si256( (reg*) &a[i] );
    reg mask = _mm256_cmpgt_epi32(c, x);
    x = _mm256_and_si256(x, mask);
    s = _mm256_add_epi32(s, x);
}
```

이 루프는 이 특정 CPU에서 벡터 `and`가 `blend`보다 한 사이클 덜 걸리기 때문에 약간 더 빠르게 실행됩니다.

마스크를 입력으로 지원하는 몇 가지 다른 명령들이 있으며, 가장 주목할 만한 것은 다음과 같습니다.

- `_mm256_blend_epi32` 인트린직은 벡터 대신 8비트 정수 마스크를 받는 `blend`입니다 (이것이 끝에 `v`가 없는 이유입니다).
- `_mm256_maskload_epi32` 및 `_mm256_maskstore_epi32` 인트린직은 메모리에서 SIMD 블록을 로드/스토어하고 한 번에 마스크와 `and` 연산을 수행합니다.

내장 벡터 타입을 사용해서도 프리디케이션을 사용할 수 있습니다.

```c++
vec *v = (vec*) a;
vec s = {};

for (int i = 0; i < N / 8; i++)
    s += (v[i] < 50 ? v[i] : 0);
```

이 모든 버전은 약 13 GFLOPS로 작동하며, 이 예제는 매우 간단해서 컴파일러가 루프를 스스로 벡터화할 수 있습니다. 이제 자동 벡터화가 불가능한 더 복잡한 예제로 넘어가 보겠습니다.

### 검색 (Searching)

다음 예제에서는 배열에서 특정 값을 찾아 그 위치를 반환해야 합니다 (`std::find`와 유사).

```c++
const int N = (1<<12);
int a[N];

int find(int x) {
    for (int i = 0; i < N; i++)
        if (a[i] == x)
            return i;
    return -1;
}
```

`find` 함수를 벤치마킹하기 위해 배열을 $0$에서 $(N - 1)$까지의 숫자로 채우고 무작위 요소를 반복적으로 검색합니다.

```c++
for (int i = 0; i < N; i++)
    a[i] = i;

for (int t = 0; t < K; t++)
    checksum ^= find(rand() % N);
```

스칼라 버전은 약 4 GFLOPS의 성능을 제공합니다. 이 수치에는 처리할 필요가 없었던 요소들도 포함되어 있으므로, 머릿속으로 이 숫자를 2로 나누십시오 (우리가 확인해야 하는 요소의 예상 비율).

이를 벡터화하려면, 요소 벡터를 검색하려는 값과 비교하여 일치 여부를 확인하고 마스크를 생성한 다음, 이 마스크가 0인지 확인해야 합니다. 만약 0이 아니라면, 필요한 요소가 이 8개 블록 안에 어딘가에 있다는 뜻입니다.

마스크가 0인지 확인하기 위해 `_mm256_movemask_ps` 인트린직을 사용할 수 있습니다. 이 인트린직은 벡터의 각 32비트 요소의 첫 번째 비트를 가져와 8비트 정수 마스크를 생성합니다. 그런 다음 이 마스크가 0이 아닌지 확인하고, 만약 0이 아니라면 `ctz` 명령을 통해 즉시 인덱스를 얻을 수 있습니다.

```c++
int find(int needle) {
    reg x = _mm256_set1_epi32(needle);

    for (int i = 0; i < N; i += 8) {
        reg y = _mm256_load_si256( (reg*) &a[i] );
        reg m = _mm256_cmpeq_epi32(x, y);
        int mask = _mm256_movemask_ps((__m256) m);
        if (mask != 0)
            return i + __builtin_ctz(mask);
    }

    return -1;
}
```

이 버전은 약 20 GFLOPS의 성능을 보여주며 스칼라 버전보다 약 5배 빠릅니다. 핵심 루프(hot loop)에서는 단 3개의 명령만 사용합니다.

```nasm
vpcmpeqd  ymm0, ymm1, YMMWORD PTR a[0+rdx*4]
vmovmskps eax, ymm0
test      eax, eax
je        loop
```

벡터가 0인지 확인하는 것은 흔한 연산이며, SIMD에는 `test`와 유사하게 사용할 수 있는 연산이 있습니다.

```c++
int find(int needle) {
    reg x = _mm256_set1_epi32(needle);

    for (int i = 0; i < N; i += 8) {
        reg y = _mm256_load_si256( (reg*) &a[i] );
        reg m = _mm256_cmpeq_epi32(x, y);
        if (!_mm256_testz_si256(m, m)) {
            int mask = _mm256_movemask_ps((__m256) m);
            return i + __builtin_ctz(mask);
        }
    }

    return -1;
}
```

나중에 `ctz`를 수행하기 위해 여전히 `movemask`를 사용하고 있지만, 핵심 루프는 이제 명령 하나가 더 짧아졌습니다.

```nasm
vpcmpeqd ymm0, ymm1, YMMWORD PTR a[0+rdx*4]
vptest   ymm0, ymm0
je       loop
```

이것은 성능을 크게 향상시키지는 않는데, 그 이유는 `vptest`와 `vmovmskps` 모두 처리량(throughput)이 1이며, 루프에서 다른 작업을 무엇을 하든 이들이 병목 현상이 되기 때문입니다.

이 제한을 우회하기 위해 16개 요소 블록씩 반복하고 비트 단위 `or`를 사용하여 두 개의 256비트 AVX2 레지스터의 독립적인 비교 결과를 결합할 수 있습니다.

```c++
int find(int needle) {
    reg x = _mm256_set1_epi32(needle);

    for (int i = 0; i < N; i += 16) {
        reg y1 = _mm256_load_si256( (reg*) &a[i] );
        reg y2 = _mm256_load_si256( (reg*) &a[i + 8] );
        reg m1 = _mm256_cmpeq_epi32(x, y1);
        reg m2 = _mm256_cmpeq_epi32(x, y2);
        reg m = _mm256_or_si256(m1, m2);
        if (!_mm256_testz_si256(m, m)) {
            int mask = (_mm256_movemask_ps((__m256) m2) << 8)
                     +  _mm256_movemask_ps((__m256) m1);
            return i + __builtin_ctz(mask);
        }
    }

    return -1;
}
```

이 장애물이 제거되면 성능은 약 34 GFLOPS까지 올라갑니다. 하지만 왜 40이 아닐까요? 두 배 더 빨라야 하지 않을까요?

다음은 어셈블리로 본 루프의 한 반복(iteration)입니다.

```nasm
vpcmpeqd ymm2, ymm1, YMMWORD PTR a[0+rdx*4]
vpcmpeqd ymm3, ymm1, YMMWORD PTR a[32+rdx*4]
vpor     ymm0, ymm3, ymm2
vptest   ymm0, ymm0
je       loop
```

매 반복마다 5개의 명령을 실행해야 합니다. 관련 실행 포트들의 처리량은 평균적으로 한 사이클에 이를 수행할 수 있게 해주지만, 이 특정 CPU(Zen 2)의 디코드 폭(decode width)이 4이기 때문에 그렇게 할 수 없습니다. 따라서 성능은 잠재력의 4/5로 제한됩니다.

<!--

CPU(Zen 2)는 4개만 처리할 수 있습니다. 다음은 [llvm-mca 보고서](/hpc/profiling/mca)의 관련 부분입니다.

vpcmpeqd 013
vpcmpeqd 013
vpor 0123
vptest 2

[7]    [8]    [9]    [10]   Instructions:
0.46   0.09    -     0.45   vpcmpeqd	ymm2, ymm1, ymmword ptr [4*rdx + a]
0.40   0.09   0.22   0.29   vpcmpeqd	ymm3, ymm1, ymmword ptr [4*rdx + a+32]
0.34   0.11   0.08   0.47   vpor	ymm0, ymm3, ymm2
 -     1.00   1.00    -     vptest	ymm0, ymm0

-->

이를 완화하기 위해 각 반복에서 처리하는 SIMD 블록의 수를 다시 한번 두 배로 늘릴 수 있습니다.

```c++
unsigned get_mask(reg m) {
    return _mm256_movemask_ps((__m256) m);
}

reg cmp(reg x, int *p) {
    reg y = _mm256_load_si256( (reg*) p );
    return _mm256_cmpeq_epi32(x, y);
}

int find(int needle) {
    reg x = _mm256_set1_epi32(needle);

    for (int i = 0; i < N; i += 32) {
        reg m1 = cmp(x, &a[i]);
        reg m2 = cmp(x, &a[i + 8]);
        reg m3 = cmp(x, &a[i + 16]);
        reg m4 = cmp(x, &a[i + 24]);
        reg m12 = _mm256_or_si256(m1, m2);
        reg m34 = _mm256_or_si256(m3, m4);
        reg m = _mm256_or_si256(m12, m34);
        if (!_mm256_testz_si256(m, m)) {
            unsigned mask = (get_mask(m4) << 24)
                          + (get_mask(m3) << 16)
                          + (get_mask(m2) << 8)
                          +  get_mask(m1);
            return i + __builtin_ctz(mask);
        }
    }

    return -1;
}
```

이제 43 GFLOPS의 처리량을 보여주며, 이는 원래 스칼라 구현보다 약 10배 빠릅니다.

사이클당 64개 값으로 확장하는 것은 도움이 되지 않습니다. 작은 배열은 조건을 만족했을 때 추가적인 `movemask`들의 오버헤드로 인해 손해를 보고, 큰 배열은 어차피 [메모리 대역폭(memory bandwidth)](/hpc/cpu-cache/bandwidth)에 의해 병목이 발생하기 때문입니다.

### 값 계산하기 (Counting Values)

마지막 연습으로, 첫 번째 발생 지점만 찾는 대신 배열에서 특정 값의 개수를 세어 보겠습니다.

```c++
int count(int x) {
    int cnt = 0;
    for (int i = 0; i < N; i++)
        cnt += (a[i] == x);
    return cnt;
}
```

이를 벡터화하려면, 비교 마스크를 각 요소당 1 또는 0으로 변환하고 합계를 계산하기만 하면 됩니다.

```c++
const reg ones = _mm256_set1_epi32(1);

int count(int needle) {
    reg x = _mm256_set1_epi32(needle);
    reg s = _mm256_setzero_si256();

    for (int i = 0; i < N; i += 8) {
        reg y = _mm256_load_si256( (reg*) &a[i] );
        reg m = _mm256_cmpeq_epi32(x, y);
        m = _mm256_and_si256(m, ones);
        s = _mm256_add_epi32(s, m);
    }

    return hsum(s);
}
```

두 구현 모두 약 15 GFLOPS를 제공하며, 컴파일러는 첫 번째 구현을 스스로 벡터화할 수 있습니다.

하지만 컴파일러가 찾지 못하는 트릭은 모든 비트가 1인 마스크가 정수로 재해석될 때 [-1](/hpc/arithmetic/integer)이라는 것을 알아채는 것입니다. 따라서 하위 비트만 취하는 `and` 부분을 건너뛰고 마스크 자체를 사용한 다음, 최종 결과에 음수 부호를 붙일 수 있습니다.

```c++
int count(int needle) {
    reg x = _mm256_set1_epi32(needle);
    reg s = _mm256_setzero_si256();

    for (int i = 0; i < N; i += 8) {
        reg y = _mm256_load_si256( (reg*) &a[i] );
        reg m = _mm256_cmpeq_epi32(x, y);
        s = _mm256_add_epi32(s, m);
    }

    return -hsum(s);
}
```

이 특정 아키텍처에서는 이것이 성능을 향상시키지 않는데, 그 이유는 처리량이 실제로는 `s`를 업데이트하는 데 병목이 걸려 있기 때문입니다. 이전 반복에 대한 의존성(dependency)이 있어서 루프가 CPU 사이클당 한 번의 반복보다 빠르게 진행될 수 없습니다. 누산기(accumulator)를 둘로 나누면 [명령어 수준 병렬성(instruction-level parallelism)](../reduction#instruction-level-parallelism)을 활용할 수 있습니다.

```c++
int count(int needle) {
    reg x = _mm256_set1_epi32(needle);
    reg s1 = _mm256_setzero_si256();
    reg s2 = _mm256_setzero_si256();

    for (int i = 0; i < N; i += 16) {
        reg y1 = _mm256_load_si256( (reg*) &a[i] );
        reg y2 = _mm256_load_si256( (reg*) &a[i + 8] );
        reg m1 = _mm256_cmpeq_epi32(x, y1);
        reg m2 = _mm256_cmpeq_epi32(x, y2);
        s1 = _mm256_add_epi32(s1, m1);
        s2 = _mm256_add_epi32(s2, m2);
    }

    s1 = _mm256_add_epi32(s1, s2);

    return -hsum(s1);
}
```

이제 약 22 GFLOPS의 성능을 보여주며, 이는 얻을 수 있는 최대치입니다.

이 코드를 더 짧은 데이터 타입에 맞게 조정할 때는 누산기가 오버플로될 수 있음을 명심하십시오. 이를 해결하려면 더 큰 크기의 또 다른 누산기를 추가하고, 정기적으로 루프를 멈춰 로컬 누산기의 값을 큰 누산기에 더한 다음 로컬 누산기를 초기화하십시오. 예를 들어, 8비트 정수의 경우 $\lfloor \frac{256-1}{8} \rfloor = 15$번의 반복을 수행하는 또 다른 내부 루프를 만드는 것을 의미합니다.
 
<!-- TODO: 8-bit example -->
<!-- TODO: ILP first, -1 second -->
