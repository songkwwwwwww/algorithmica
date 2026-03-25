---
title: 리덕션 (Reductions)
weight: 3
---

*리덕션(Reduction)* (함수형 프로그래밍에서는 *폴딩(folding)*이라고도 함)은 임의의 요소 범위에 대해 결합 법칙과 교환 법칙이 성립하는 연산(즉, $(a \circ b) \circ c = a \circ (b \circ c)$ 이고 $a \circ b = b \circ a$)을 적용하여 하나의 값을 계산하는 작업입니다.

리덕션의 가장 간단한 예는 배열의 합계를 구하는 것입니다.

```c++
int sum(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s += a[i];
    return s;
}
```

나이브(naive)한 접근 방식은 벡터화하기가 쉽지 않은데, 루프의 상태(현재 프리픽스까지의 합 $s$)가 이전 반복에 의존하기 때문입니다. 이를 해결하는 방법은 하나의 스칼라 누산기 $s$를 8개의 별도 누산기로 나누는 것입니다. 그러면 $s_i$는 원래 배열의 매 8번째 요소들을 $i$만큼 오프셋을 두어 합산한 값을 갖게 됩니다.

$$
s_i = \sum_{j=0}^{n / 8} a_{8 \cdot j + i }
$$

이 8개의 누산기를 하나의 256비트 벡터에 저장하면, 배열의 연속된 8개 요소 세그먼트를 더함으로써 한 번에 모두 업데이트할 수 있습니다. [벡터 확장(vector extensions)](../x86-simd)을 사용하면 이를 직관적으로 구현할 수 있습니다.

```c++
int sum_simd(v8si *a, int n) {
    // ^ 다른 포인터 타입과 마찬가지로 일반적인 포인터 캐스팅이 가능합니다.
    v8si s = {0};

    for (int i = 0; i < n / 8; i++)
        s += a[i];
    
    int res = 0;
    
    // 8개의 누산기를 하나로 합칩니다.
    for (int i = 0; i < 8; i++)
        res += s[i];

    // 배열 a의 남은 부분을 더합니다.
    for (int i = n / 8 * 8; i < n; i++)
        res += a[i];
        
    return res;
}
```

이 접근 방식은 배열의 최솟값을 찾거나 XOR 합계를 구하는 등 다른 리덕션 작업에도 사용할 수 있습니다.

### 명령어 수준 병렬성 (Instruction-Level Parallelism)

우리의 구현은 컴파일러가 자동으로 생성하는 것과 일치하지만, 실제로는 최적의 상태가 아닙니다. 하나의 누산기만 사용하면 벡터 덧셈이 완료될 때까지 루프 반복 사이에 [한 사이클을 기다려야 하지만](/hpc/pipelining/throughput), 이 마이크로아키텍처에서 해당 명령어의 [처리량(throughput)](/hpc/pipelining/tables/)은 2이기 때문입니다.

배열을 다시 $B \geq 2$ 부분으로 나누고 각각에 대해 *별도의* 누산기를 사용하면, 벡터 덧셈의 처리량을 최대한 활용하여 성능을 두 배로 높일 수 있습니다.

```c++
const int B = 2; // 사용할 벡터 누산기 개수

int sum_simd(v8si *a, int n) {
    v8si b[B] = {0};

    for (int i = 0; i < n / 8; i += B)
        for (int j = 0; j < B; j++)
            b[j] += a[i + j];

    // 모든 벡터 누산기를 하나로 합칩니다.
    for (int i = 1; i < B; i++)
        b[0] += b[i];
    
    int s = 0;

    // 8개의 스칼라 누산기를 하나로 합칩니다.
    for (int i = 0; i < 8; i++)
        s += b[0][i];

    // 배열 a의 남은 부분을 더합니다.
    for (int i = n / (8 * B) * (8 * B); i < n; i++)
        s += a[i];

    return s;
}
```

관련 실행 포트가 2개보다 많다면 `B` 상수를 그에 맞게 늘릴 수 있습니다. 다만, $n$배의 성능 향상은 L1 캐시에 들어가는 크기의 배열에만 적용되며, 그보다 큰 배열은 [메모리 대역폭(memory bandwidth)](/hpc/cpu-cache/bandwidth)이 병목 현상이 될 것입니다.

### 수평적 합계 (Horizontal Summation)

벡터 레지스터에 저장된 8개의 누산기를 하나의 스칼라로 합산하여 최종 합계를 구하는 부분을 "수평적 합계"라고 합니다.

각 스칼라를 하나씩 추출하여 더하는 것도 일정한 사이클만 소요되지만, 레지스터 내의 인접한 요소 쌍을 더하는 [특수 명령어](https://software.intel.com/sites/landingpage/IntrinsicsGuide/#techs=AVX,AVX2&text=_mm256_hadd_epi32&expand=2941)를 사용하면 조금 더 빠르게 계산할 수 있습니다.

![SSE/AVX의 수평적 합계. 출력 값이 어떻게 저장되는지 주의 깊게 보세요. (a b a b) 형태의 인터리빙은 리덕션 연산에서 흔히 나타납니다.](../img/hsum.png)

이는 매우 특정한 연산이므로 SIMD 인트린직으로만 수행할 수 있습니다. 다만 컴파일러도 아마 스칼라 코드에 대해 거의 동일한 절차를 생성할 것입니다.

```c++
int hsum(__m256i x) {
    __m128i l = _mm256_extracti128_si256(x, 0);
    __m128i h = _mm256_extracti128_si256(x, 1);
    l = _mm_add_epi32(l, h);
    l = _mm_hadd_epi32(l, l);
    return _mm_extract_epi32(l, 0) + _mm_extract_epi32(l, 1);
}
```

정수 곱셈이나 인접 요소 간의 절대 차이 계산(이미지 처리에서 사용됨)을 위한 [다른 유사한 명령어](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#techs=AVX,AVX2&ig_expand=3037,3009,5135,4870,4870,4872,4875,833,879,874,849,848,6715,4845&text=horizontal)들도 있습니다.

또한 `_mm_minpos_epu16`이라는 특정 명령어는 8개의 16비트 정수 중 수평적 최솟값과 그 인덱스를 계산합니다. 이는 한 번에 작동하는 유일한 수평적 리덕션이며, 다른 모든 것들은 여러 단계에 걸쳐 계산됩니다.
