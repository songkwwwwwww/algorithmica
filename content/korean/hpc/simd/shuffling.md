---
title: 레지스터 내 셔플 (In-Register Shuffles)
weight: 6
---

[마스킹](../masking)을 사용하면 벡터 요소의 일부에만 연산을 적용할 수 있습니다. 이는 매우 효과적이고 자주 사용되는 데이터 조작 기술이지만, 많은 경우 단순히 다른 벡터와 혼합하는 것을 넘어 벡터 레지스터 내부의 값을 치환(permute)하는 더 고급 연산이 필요합니다.

문제는 하드웨어에서 가능한 모든 사용 사례에 대해 별도의 요소 셔플링 명령어를 추가하는 것이 불가능하다는 점입니다. 대신 우리가 할 수 있는 일은 치환 인덱스를 받아서 미리 계산된 룩업 테이블(lookup table)을 사용해 결과를 생성하는 하나의 일반적인 치환 명령어를 추가하는 것입니다.

이 일반적인 개념은 너무 추상적일 수 있으니, 바로 예제로 넘어가 보겠습니다.

### 셔플과 팝카운트 (Popcount)

*팝카운트(Population count)* 또는 *해밍 중량(Hamming weight)*은 이진 문자열에서 `1` 비트의 개수를 세는 연산입니다.

이는 자주 사용되는 연산이어서 x86에는 워드의 팝카운트를 계산하는 별도의 명령어가 있습니다.

```c++
const int N = (1<<12);
int a[N];

int popcnt() {
    int res = 0;
    for (int i = 0; i < N; i++)
        res += __builtin_popcount(a[i]);
    return res;
}
```

또한 64비트 정수를 지원하여 전체 처리량을 두 배로 높일 수 있습니다.

```c++
int popcnt_ll() {
    long long *b = (long long*) a;
    int res = 0;
    for (int i = 0; i < N / 2; i++)
        res += __builtin_popcountl(b[i]);
    return res;
}
```

필요한 명령어는 로드-융합(load-fused) 팝카운트와 덧셈뿐입니다. 두 명령어 모두 처리량이 높기 때문에, 이 CPU의 디코드 너비 4에 의해 제한되어 사이클당 약 $8+8=16$바이트를 처리합니다.

이 명령어들은 2008년경 SSE4와 함께 x86 CPU에 추가되었습니다. 잠시 벡터화가 대중화되기 전으로 시간을 되돌려 다른 방법으로 팝카운트를 구현해 봅시다.

가장 나이브한 방법은 이진 문자열을 비트 단위로 훑는 것입니다.

```c++
__attribute__ (( optimize("no-tree-vectorize") ))
int popcnt() {
    int res = 0;
    for (int i = 0; i < N; i++)
        for (int l = 0; l < 32; l++)
            res += (a[i] >> l & 1);
    return res;
}
```

예상대로 사이클당 1/8바이트보다 약간 빠른 약 0.2바이트 정도의 속도로 작동합니다.

개별 비트 대신 바이트 단위로 처리하기 위해, 각 바이트의 팝카운트 값을 담고 있는 256개 요소의 작은 *룩업 테이블*을 [미리 계산](../hpc/compilation/precalc)해 두고 배열의 원시 바이트를 반복하면서 이를 조회할 수 있습니다.

```c++
struct Precalc {
    alignas(64) char counts[256];

    constexpr Precalc() : counts{} {
        for (int m = 0; m < 256; m++)
            for (int i = 0; i < 8; i++)
                counts[m] += (m >> i & 1);
    }
};

constexpr Precalc P;

int popcnt() {
    auto b = (unsigned char*) a; // 주의: 일반 "char"는 부호가 있음
    int res = 0;
    for (int i = 0; i < 4 * N; i++)
        res += P.counts[b[i]];
    return res;
}
```

이제 사이클당 약 2바이트를 처리하며, 16비트 워드(`unsigned short`)로 전환하면 약 2.7바이트까지 올라갑니다.

이 솔루션은 `popcnt` 명령어에 비하면 여전히 매우 느리지만, 이제 벡터화가 가능해졌습니다. [개더(gather)](../moving#non-contiguous-load) 명령어를 통해 속도를 높이는 대신 다른 접근 방식을 취하겠습니다. 룩업 테이블을 레지스터 안에 들어갈 만큼 작게 만들고, 특수한 [pshufb](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#text=pshuf&techs=AVX,AVX2&expand=6331) 명령어를 사용하여 병렬로 값을 조회하는 것입니다.

128비트 SSE3에서 도입된 원래의 `pshufb`는 두 개의 레지스터를 받습니다. 하나는 16개의 바이트 값을 포함하는 룩업 테이블이고, 다른 하나는 각 위치에서 가져올 바이트를 지정하는 16개의 4비트 인덱스(0~15) 벡터입니다. 256비트 AVX2에서는 어색한 5비트 인덱스를 사용하는 32바이트 룩업 테이블 대신, 두 개의 128비트 레인에서 독립적으로 동일한 셔플링 연산을 수행하는 명령어를 사용합니다.

따라서 우리의 사례에서는 각 니블(nibble, 반 바이트)에 대한 팝카운트 값을 포함하는 16바이트 룩업 테이블을 두 번 반복하여 생성합니다.

```c++
const reg lookup = _mm256_setr_epi8(
    /* 0 */ 0, /* 1 */ 1, /* 2 */ 1, /* 3 */ 2,
    /* 4 */ 1, /* 5 */ 2, /* 6 */ 2, /* 7 */ 3,
    /* 8 */ 1, /* 9 */ 2, /* a */ 2, /* b */ 3,
    /* c */ 2, /* d */ 3, /* e */ 3, /* f */ 4,

    /* 0 */ 0, /* 1 */ 1, /* 2 */ 1, /* 3 */ 2,
    /* 4 */ 1, /* 5 */ 2, /* 6 */ 2, /* 7 */ 3,
    /* 8 */ 1, /* 9 */ 2, /* a */ 2, /* b */ 3,
    /* c */ 2, /* d */ 3, /* e */ 3, /* f */ 4
);
```

이제 벡터의 팝카운트를 계산하기 위해, 각 바이트를 하위 니블과 상위 니블로 나누고 이 룩업 테이블을 사용하여 각각의 개수를 가져옵니다. 남은 일은 이를 조심스럽게 합산하는 것뿐입니다.

```c++
const reg low_mask = _mm256_set1_epi8(0x0f);

int popcnt() {
    int k = 0;

    reg t = _mm256_setzero_si256();

    for (; k + 15 < N; k += 15) {
        reg s = _mm256_setzero_si256();
        
        for (int i = 0; i < 15; i += 8) {
            reg x = _mm256_load_si256( (reg*) &a[k + i] );
            
            reg l = _mm256_and_si256(x, low_mask);
            reg h = _mm256_and_si256(_mm256_srli_epi16(x, 4), low_mask);

            reg pl = _mm256_shuffle_epi8(lookup, l);
            reg ph = _mm256_shuffle_epi8(lookup, h);

            s = _mm256_add_epi8(s, pl);
            s = _mm256_add_epi8(s, ph);
        }

        t = _mm256_add_epi64(t, _mm256_sad_epu8(s, _mm256_setzero_si256()));
    }

    int res = hsum(t);

    while (k < N)
        res += __builtin_popcount(a[k++]);

    return res;
}
```

이 코드는 사이클당 약 30바이트를 처리합니다. 이론적으로 내부 루프는 32바이트를 처리할 수 있지만, 8비트 카운터가 오버플로될 수 있기 때문에 15번의 반복마다 멈춰야 합니다.

`pshufb` 명령어는 일부 SIMD 알고리즘에서 매우 중요한 역할을 하기 때문에, 이 알고리즘을 고안한 [Wojciech Muła](http://0x80.pl/)는 자신의 [트위터 핸들](https://twitter.com/pshufb)로 이를 사용하기도 했습니다. 팝카운트를 더 빠르게 계산할 수도 있습니다. 다양한 벡터화된 팝카운트 구현이 담긴 그의 [GitHub 저장소](https://github.com/WojciechMula/sse-popcount)와 최신 기술에 대한 자세한 설명이 담긴 [최근 논문](https://arxiv.org/pdf/1611.07612.pdf)을 확인해 보세요.

### 치환과 룩업 테이블 (Permutations and Lookup Tables)

이 장의 마지막 주요 예제는 `filter`입니다. 이는 배열을 입력으로 받아 주어진 조건(predicate)을 만족하는 요소들만 원래 순서대로 출력하는 매우 중요한 데이터 처리 기본 연산입니다.

단일 스레드 스칼라의 경우, 쓰기가 발생할 때마다 증가하는 카운터를 유지함으로써 아주 쉽게 구현할 수 있습니다.

```c++
int a[N], b[N];

int filter() {
    int k = 0;

    for (int i = 0; i < N; i++)
        if (a[i] < P)
            b[k++] = a[i];

    return k;
}
```

이를 벡터화하기 위해 `_mm256_permutevar8x32_epi32` 인트린직을 사용할 것입니다. 이 인트린직은 값 벡터를 받아 인덱스 벡터에 따라 개별적으로 선택합니다. 이름과는 달리 값을 실제로 *치환*하는 것이 아니라 새로운 벡터를 형성하기 위해 값을 *복사*하는 것이므로 결과에 중복된 값이 포함될 수 있습니다.

우리 알고리즘의 일반적인 아이디어는 다음과 같습니다.

- 데이터 벡터에 대해 조건을 계산합니다. 이 경우 비교를 수행하여 마스크를 얻는 것을 의미합니다.
- `movemask` 명령어를 사용하여 스칼라 8비트 마스크를 얻습니다.
- 이 마스크를 사용하여 조건을 만족하는 요소들을 벡터의 앞으로 (원래 순서대로) 옮기는 치환 정보를 반환하는 룩업 테이블을 참조합니다.
- `_mm256_permutevar8x32_epi32` 인트린직을 사용하여 값을 치환합니다.
- 치환된 전체 벡터를 버퍼에 씁니다. 뒤쪽에 쓰레기 데이터가 포함될 수 있지만 앞부분은 정확합니다.
- 스칼라 마스크의 팝카운트를 계산하고 그 수만큼 버퍼 포인터를 이동시킵니다.

먼저 치환 정보를 미리 계산해야 합니다.

```c++
struct Precalc {
    alignas(64) int permutation[256][8];

    constexpr Precalc() : permutation{} {
        for (int m = 0; m < 256; m++) {
            int k = 0;
            for (int i = 0; i < 8; i++)
                if (m >> i & 1)
                    permutation[m][k++] = i;
        }
    }
};

constexpr Precalc T;
```

그런 다음 알고리즘을 구현할 수 있습니다.

```c++
const reg p = _mm256_set1_epi32(P);

int filter() {
    int k = 0;

    for (int i = 0; i < N; i += 8) {
        reg x = _mm256_load_si256( (reg*) &a[i] );
        
        reg m = _mm256_cmpgt_epi32(p, x);
        int mask = _mm256_movemask_ps((__m256) m);
        reg permutation = _mm256_load_si256( (reg*) &T.permutation[mask] );
        
        x = _mm256_permutevar8x32_epi32(x, permutation);
        _mm256_storeu_si256((reg*) &b[k], x);
        
        k += __builtin_popcount(mask);
    }

    return k;
}
```

벡터화된 버전은 구현하는 데 약간의 노력이 필요하지만, 스칼라 버전보다 6~7배 더 빠릅니다. (단, `P` 값이 매우 낮거나 매우 높아서 [분기가 예측 가능해지면](/hpc/pipelining/branching) 성능 향상 폭이 약간 줄어듭니다.)

![](../img/filter.svg)

루프 성능은 여전히 반복당 4 CPU 사이클 정도로 상대적으로 낮은 편입니다. 이는 특정 CPU(Zen 2)에서 `movemask`, `permute`, `store`의 처리량이 낮고 모두 동일한 실행 포트(P2)를 거쳐야 하기 때문입니다. 대부분의 다른 x86 CPU에서는 약 2배 정도 더 빠를 것으로 기대할 수 있습니다.

필터링은 AVX-512에서 훨씬 더 빠르게 구현할 수 있습니다. AVX-512에는 데이터 벡터와 마스크를 받아 마스킹되지 않은 요소들만 연속적으로 써주는 특별한 "[compress](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=7395,7392,7269,4868,7269,7269,1820,1835,6385,5051,4909,4918,5051,7269,6423,7410,150,2138,1829,1944,3009,1029,7077,519,5183,4462,4490,1944,1395&text=_mm512_mask_compress_epi32)" 명령어가 있기 때문입니다. 이는 퀵소트(quicksort)와 같이 다양한 필터링 서브루틴에 의존하는 알고리즘에서 엄청난 차이를 만듭니다.
