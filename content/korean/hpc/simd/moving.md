---
title: 데이터 이동
aliases: [/hpc/simd/vectorization]
weight: 2
---

[레퍼런스](https://software.intel.com/sites/landingpage/IntrinsicsGuide)를 공부하는 데 시간을 좀 투자했다면, 벡터 연산에는 크게 두 가지 그룹이 있다는 것을 눈치챘을 것입니다.

1. 요소별(elementwise) 연산(`+`, `*`, `<`, `acos` 등)을 수행하는 명령들.
2. 데이터를 로드(load), 스토어(store), 마스크(mask), 셔플(shuffle)하고 전반적으로 데이터를 이동시키는 명령들.

요소별 연산 명령을 사용하는 것은 쉽지만, SIMD에서 가장 큰 도전 과제는 애초에 데이터를 벡터 레지스터에 넣는 것입니다. 이때 오버헤드가 충분히 낮아야 전체 작업이 가치 있게 됩니다.

### 정렬된 로드와 스토어 (Aligned Loads and Stores)

SIMD 레지스터의 내용을 메모리에 읽고 쓰는 연산에는 각각 두 가지 버전이 있습니다: `load` / `loadu` 및 `store` / `storeu`. 여기서 "u"는 "unaligned(정렬되지 않은)"를 의미합니다. 차이점은 전자는 읽기/쓰기 블록이 단일 [캐시 라인](/hpc/cpu-cache/cache-lines) 내에 들어갈 때만 올바르게 작동하고(그렇지 않으면 크래시가 발생함), 후자는 어느 쪽이든 작동하지만 블록이 캐시 라인을 가로지르는 경우 약간의 성능 저하가 발생한다는 것입니다.

때때로, 특히 "내부" 연산이 매우 가벼울 때 성능 차이가 상당해질 수 있습니다(최소한 하나의 캐시 라인 대신 두 개를 가져와야 하기 때문입니다). 극단적인 예로, 두 배열을 함께 더하는 이 방식은:

```c++
for (int i = 3; i + 7 < n; i += 8) {
    __m256i x = _mm256_loadu_si256((__m256i*) &a[i]);
    __m256i y = _mm256_loadu_si256((__m256i*) &b[i]);
    __m256i z = _mm256_add_epi32(x, y);
    _mm256_storeu_si256((__m256i*) &c[i], z);
}
```

…정렬된 버전보다 약 30% 더 느립니다:

```c++
for (int i = 0; i < n; i += 8) {
    __m256i x = _mm256_load_si256((__m256i*) &a[i]);
    __m256i y = _mm256_load_si256((__m256i*) &b[i]);
    __m256i z = _mm256_add_epi32(x, y);
    _mm256_store_si256((__m256i*) &c[i], z);
}
```

첫 번째 버전에서 배열 `a`, `b`, `c`가 모두 64바이트 *정렬(aligned)*되어 있다고 가정하면(첫 번째 요소의 주소가 64로 나누어 떨어지므로 캐시 라인의 시작 부분에서 시작함), 읽기 및 쓰기의 대략 절반은 캐시 라인 경계를 가로지르기 때문에 "나쁜" 상태가 됩니다.

성능 차이는 명령 자체가 아니라 캐시 시스템에 의해 발생한다는 점에 유의하십시오. 대부분의 현대 아키텍처에서 `loadu` / `storeu` 인트린직은 블록이 하나의 캐시 라인만 차지하는 경우 `load` / `store`만큼 빨라야 합니다. 후자의 장점은 모든 읽기 및 쓰기가 정렬되어 있다는 런타임 어설션(assertion) 역할을 무료로 수행할 수 있다는 것입니다.

이로 인해 할당 시 배열 및 기타 데이터를 적절하게 [정렬](/hpc/cpu-cache/alignment)하는 것이 중요하며, 이는 컴파일러가 항상 효율적으로 [자동 벡터화](../auto-vectorization)를 수행할 수 없는 이유 중 하나이기도 합니다. 대부분의 목적을 위해, 32바이트 SIMD 블록이 캐시 라인 경계를 가로지르지 않도록 보장하기만 하면 되며, `alignas` 지정자를 사용하여 이 정렬을 지정할 수 있습니다.

<!--

By default, when you allocate an array, the only guarantee about its alignment you get is that none of its elements are split by a cache line. For an array of `int`, this means that it gets the alignment of 4 bytes (`sizeof int`), which lets you load exactly one cache line when reading any element. For our purposes, we want to guarantee that any (256-bit = 32-byte) SIMD block will not be split, so we need to specify the alignment of 32 bytes. For static arrays, we can do so with the `alignas` specifier:

-->

```c++
alignas(32) float a[n];

for (int i = 0; i < n; i += 8) {
    __m256 x = _mm256_load_ps(&a[i]);
    // ...
}
```

[내장 벡터 타입](../intrinsics)에는 이미 해당 정렬 요구 사항이 있으며 정렬된 메모리 읽기 및 쓰기를 가정합니다. 따라서 `v8si` 배열을 할당할 때는 항상 안전하지만, `int*`에서 변환할 때는 정렬되어 있는지 확인해야 합니다.

스칼라의 경우와 유사하게, 많은 산술 명령은 메모리 주소를 피연산자로 사용합니다([벡터 덧셈](../intrinsics)이 그 예입니다). 비록 이를 명시적으로 인트린직으로 사용할 수는 없으며 컴파일러에 의존해야 하지만 말입니다. 메모리에서 SIMD 블록을 읽기 위한 몇 가지 다른 명령도 있는데, 특히 캐시 계층 구조에서 액세스된 데이터를 올리지 않는 [비 temporal(non-temporal)](/hpc/cpu-cache/bandwidth#bypassing-the-cache) 로드 및 스토어 연산이 주목할 만합니다.

### 레지스터 에일리어싱 (Register Aliasing)

첫 번째 SIMD 확장인 MMX는 상당히 작게 시작되었습니다. 단지 64비트 벡터만 사용했는데, 이는 [80비트 부동 소수점](/hpc/arithmetic/ieee-754)의 가수(mantissa) 부분에 편리하게 에일리어싱되어 별도의 레지스터 세트를 도입할 필요가 없었습니다. 이후 확장에서 벡터 크기가 커짐에 따라, 하위 호환성을 유지하기 위해 범용 레지스터에서 사용된 것과 동일한 [레지스터 에일리어싱](/hpc/architecture/assembly#instructions-and-registers) 메커니즘이 벡터 레지스터에도 채택되었습니다. `xmm0`은 `ymm0`의 첫 번째 절반(128비트)이고, `xmm1`은 `ymm1`의 첫 번째 절반인 식입니다.

벡터 레지스터가 FPU에 위치한다는 사실과 결합된 이 기능은 벡터 레지스터와 범용 레지스터 간에 데이터를 이동하는 것을 약간 복잡하게 만듭니다.

### 추출과 삽입 (Extract and Insert)

벡터에서 특정 값을 *추출(extract)*하려면 `_mm256_extract_epi32` 및 유사한 인트린직을 사용할 수 있습니다. 추출할 정수의 인덱스를 두 번째 매개변수로 취하며 그 값에 따라 다른 명령 시퀀스를 생성합니다.

첫 번째 요소를 추출해야 하는 경우, `vmovd` 명령을 생성합니다(벡터의 첫 번째 절반인 `xmm0`의 경우):

```nasm
vmovd eax, xmm0
```

SSE 벡터의 다른 요소의 경우, 약간 더 느릴 수 있는 `vpextrd`를 생성합니다:

```nasm
vpextrd eax, xmm0, 1
```

AVX 벡터의 두 번째 절반에서 무언가를 추출하려면, 먼저 그 두 번째 절반을 추출한 다음 스칼라 자체를 추출해야 합니다. 예를 들어, 마지막(여덟 번째) 요소를 추출하는 방법은 다음과 같습니다:

```nasm
vextracti128 xmm0, ymm0, 0x1
vpextrd      eax, xmm0, 3
```

특정 요소를 덮어쓰기 위한 유사한 `_mm256_insert_epi32` 인트린직이 있습니다:

```nasm
mov          eax, 42

; v = _mm256_insert_epi32(v, 42, 0);
vpinsrd xmm2, xmm0, eax, 0
vinserti128     ymm0, ymm0, xmm2, 0x0

; v = _mm256_insert_epi32(v, 42, 7);
vextracti128 xmm1, ymm0, 0x1
vpinsrd      xmm2, xmm1, eax, 3
vinserti128  ymm0, ymm0, xmm2, 0x1
```

핵심 요점: 스칼라 데이터를 벡터 레지스터로 또는 그 반대로 이동하는 것은 느리며, 특히 첫 번째 요소가 아닐 때 더욱 그렇습니다.

### 상수 만들기 (Making Constants)

하나의 요소뿐만 아니라 전체 벡터를 채워야 하는 경우 `_mm256_setr_epi32` 인트린직을 사용할 수 있습니다:

```c++
__m256 iota = _mm256_setr_epi32(0, 1, 2, 3, 4, 5, 6, 7);
```

여기서 "r"은 인간 기준이 아니라 [CPU 관점](/hpc/arithmetic/integer#integer-types)에서 "reversed(반전된)"를 의미합니다. 반대 방향에서 값을 채우는 `_mm256_set_epi32`("r" 없음)도 있습니다. 둘 다 주로 컴파일 타임 상수를 만드는 데 사용되며, 이 상수는 블록 로드를 통해 레지스터로 페치(fetch)됩니다. 벡터를 0으로 채우는 것이 목적이라면 대신 `_mm256_setzero_si256`를 사용하십시오. 이는 레지스터를 자기 자신과 `xor` 연산합니다.

내장 벡터 타입에서는 그냥 일반적인 중괄호 초기화(braced initialization)를 사용할 수 있습니다:

```c++
vec zero = {};
vec iota = {0, 1, 2, 3, 4, 5, 6, 7};
```

### 브로드캐스트 (Broadcast)

단 하나의 요소만 수정하는 대신, 단일 값을 모든 위치로 *브로드캐스트(broadcast)*할 수도 있습니다:

```nasm
; __m256i v = _mm256_set1_epi32(42);
mov          eax, 42
vmovd        xmm0, eax
vpbroadcastd ymm0, xmm0
```

이것은 자주 사용되는 연산이므로 메모리 위치를 사용할 수도 있습니다:

```nasm
; __m256 v = _mm256_broadcast_ss(&a[i]);
vbroadcastss ymm0, DWORD PTR [rdi]
```

내장 벡터 타입을 사용할 때는 제로 벡터를 만들고 여기에 스칼라를 더할 수 있습니다:

```c++
vec v = 42 + vec{};
```

### 배열로 매핑하기 (Mapping to Arrays)

이 모든 복잡함을 피하고 싶다면 벡터를 메모리에 덤프하고 그 값을 스칼라로 다시 읽을 수 있습니다:

```c++
void print(__m256i v) {
    auto t = (unsigned*) &v;
    for (int i = 0; i < 8; i++)
        std::cout << std::bitset<32>(t[i]) << " ";
    std::cout << std::endl;
}
```

이것은 빠르지 않거나 기술적으로 올바르지 않을 수 있지만(C++ 표준은 이와 같이 데이터를 캐스팅할 때 어떤 일이 일어나는지 명시하지 않음), 간단하며 필자는 디버깅 중에 벡터의 내용을 출력하기 위해 이 코드를 자주 사용합니다.

<!-- vector types syntax -->

### 비연속적 로드 (Non-Contiguous Load)

이후의 SIMD 확장은 임의의 배열 인덱스를 사용하여 데이터를 비순차적으로 읽고 쓰는 특별한 "게더(gather)" 및 "스캐터(scatter)" 명령을 추가했습니다. 하지만 이것들이 8배 더 빨리 작동하지는 않으며 대개 CPU보다는 메모리에 의해 제한되지만, 희소 선형 대수(sparse linear algebra)와 같은 특정 응용 프로그램에는 여전히 유용합니다.

게더는 AVX2부터 사용할 수 있으며, 다양한 스캐터 명령은 AVX512부터 사용할 수 있습니다.

![](../img/gather-scatter.png)

이것들이 스칼라 읽기보다 빠르게 작동하는지 확인해 봅시다. 먼저 크기가 $N$인 배열과 $Q$개의 무작위 읽기 쿼리를 생성합니다:

```c++
int a[N], q[Q];

for (int i = 0; i < N; i++)
    a[i] = rand();

for (int i = 0; i < Q; i++)
    q[i] = rand() % N;
```

스칼라 코드에서는 쿼리에 의해 지정된 요소를 체크섬(checksum)에 하나씩 더합니다:

```c++
int s = 0;

for (int i = 0; i < Q; i++)
    s += a[q[i]];
```

그리고 SIMD 코드에서는 `gather` 명령을 사용하여 8개의 서로 다른 인덱스에 대해 병렬로 수행합니다:

```c++
reg s = _mm256_setzero_si256();

for (int i = 0; i < Q; i += 8) {
    reg idx = _mm256_load_si256( (reg*) &q[i] );
    reg x = _mm256_i32gather_epi32(a, idx, 4);
    s = _mm256_add_epi32(s, x);
}
```

배열이 L1 캐시에 들어가는 경우를 제외하고는 거의 동일하게 수행됩니다.

![](../img/gather.svg)

`gather`와 `scatter`의 목적은 메모리 연산을 더 빠르게 수행하는 것이 아니라, 데이터를 레지스터로 가져와서 무거운 계산을 수행하기 위한 것입니다. 단순한 덧셈 하나보다 더 비싼 작업에 대해서는 매우 유리합니다.

(빠른) 게더 및 스캐터 명령의 부재로 인해 CPU에서의 SIMD 프로그래밍은 독립적인 메모리 액세스를 지원하는 적절한 병렬 컴퓨팅 환경과는 매우 다릅니다. 항상 이를 우회하여 설계해야 하며, 데이터를 레지스터로 로드할 수 있도록 순차적으로 구성하는 다양한 방법을 채택해야 합니다.
