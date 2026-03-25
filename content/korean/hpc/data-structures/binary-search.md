---
title: 이진 탐색 (Binary Search)
weight: 1
---

<!-- mention interpolation search and radix trees? -->

사용자용 애플리케이션의 속도를 높이는 것이 성능 공학의 최종 목표이긴 하지만, 어떤 데이터베이스에서 5~10% 정도 성능이 향상되었다고 해서 사람들이 크게 열광하지는 않습니다. 물론 이것이 소프트웨어 엔지니어가 보수를 받는 이유이긴 하지만, 이런 종류의 최적화는 너무 복잡하고 시스템에 특화된 경우가 많아 다른 소프트웨어에 일반화하여 적용하기 어렵기 때문입니다.

그 대신, 성능 공학의 가장 매력적인 사례는 교과서에 나오는 알고리즘을 수배씩 최적화하는 것입니다. 누구나 알고 있고 너무 간단해서 애초에 최적화할 생각조차 못 했던 그런 알고리즘들 말이죠. 이러한 최적화는 간단하고 교육적이며 다른 곳에서도 충분히 채택될 수 있습니다. 그리고 놀랍게도 생각보다 그리 드물지 않습니다.

<!-- Yet, with remarkable periodicity, these can be optimized to ridiculous levels of performance. -->

이 섹션에서는 이러한 근본적인 알고리즘 중 하나인 *이진 탐색(binary search)*에 초점을 맞춥니다. 문제 크기에 따라 `std::lower_bound`보다 최대 4배 빠르면서도 15줄 이내의 코드로 구현할 수 있는 두 가지 변형을 살펴보겠습니다.

첫 번째 알고리즘은 [분기(branches)](/hpc/pipelining/branching)를 제거하여 성능을 높이고, 두 번째 알고리즘은 더 나은 [캐시 시스템](/hpc/cpu-cache) 성능을 위해 메모리 레이아웃을 최적화합니다. 두 번째 방식은 쿼리에 응답하기 전에 배열의 요소를 재배치해야 하므로 엄밀히 말하면 `std::lower_bound`를 그대로 대체할 수는 없습니다. 하지만 정렬된 배열을 얻으면서 전처리에 선형 시간(linear time)을 투자할 여유가 없는 상황은 그리 많지 않을 것입니다.

<!--

- *Branchless binary search* that is up to 3x faster on *small* arrays and can act as a drop-in replacement to `std::lower_bound`.
- *Eytzinger binary search* that rearranges the elements of a sorted array in a cache-friendly way of is also 3x faster on small arrays and 2x faster on large arrays.

-->

일반적인 고지 사항: CPU는 [Zen 2](https://www.7-cpu.com/cpu/Zen2.html), RAM은 [DDR4-2666](/hpc/cpu-cache/)이며, 기본적으로 Clang 10 컴파일러를 사용합니다. 여러분의 머신에서의 성능은 다를 수 있으므로 직접 [테스트](https://godbolt.org/z/14rd5Pnve)해 보시기를 강력히 권장합니다.

<!--

It performs slightly worse on array sizes that fit lower layers of cache, but in low-bandwidth environments it can be up to 3x faster (or 7x faster than `std::lower_bound`). GCC sucked on all benchmarks, so we will mostly be using Clang (10.0). The CPU is a Zen 2, although the results should be transferrable to other platforms, including most Arm-based chips.

The CPU is a Zen 2, and as always, the results are a bit architecture dependant, although the results should be transferrable to other platforms, including most Arm-based chips.

This is a large article, which will turn into a multi-hour read. If you feel comfortable reading [intrinsic](/hpc/simd/intrinsics)-heavy code without any context whatsoever, you can skim through the first four implementation and jump straight to the last section.

Build up understanding gradually, but you can skip them.

-->

## 이진 탐색 (Binary Search)

<!--

For our benchmark, we create an array of random integers of size `n` and sort it. Then, each implementations can do some preprocessing:

```c++
void prepare(int *a, int n);
int lower_bound(int x);
```

Already sorted array `t` of size `n`.

We are going ot create an array named `a` into array named `t`.

-->

정렬된 $n$개의 정수 배열 `t`에서 `x`보다 작지 않은 첫 번째 요소를 찾는 일반적인 방법은 컴퓨터 과학 입문서 어디에서나 볼 수 있는 방식입니다.

```c++
int lower_bound(int x) {
    int l = 0, r = n - 1;
    while (l < r) {
        int m = (l + r) / 2;
        if (t[m] >= x)
            r = m;
        else
            l = m + 1;
    }
    return t[l];
}
```

<!-- We maintain the indices of first and the last element that may be the answer, compare the element the middle to the key `x`, and then shrink the search interval by half depending how the comparison went. Beautiful in its simplicity. -->

탐색 범위의 중간 요소를 찾아 `x`와 비교하고 범위를 절반으로 줄입니다. 단순함의 미학이죠.

`std::lower_bound`도 비슷한 접근 방식을 사용하지만, 임의 접근 반복자(random-access iterator)가 없는 컨테이너도 지원해야 하므로 더 범용적입니다. 그래서 두 끝점 대신 첫 번째 요소와 탐색 구간의 크기를 사용합니다. 이를 위해 [Clang](https://github.com/llvm-mirror/libcxx/blob/78d6a7767ed57b50122a161b91f59f19c9bd0d19/include/algorithm#L4169)과 [GCC](https://github.com/gcc-mirror/gcc/blob/d9375e490072d1aae73a93949aa158fcd2a27018/libstdc%2B%2B-v3/include/bits/stl_algobase.h#L1023)의 구현체는 다음과 같은 메타프로그래밍의 괴물을 사용합니다.

```c++
template <class _Compare, class _ForwardIterator, class _Tp>
_LIBCPP_CONSTEXPR_AFTER_CXX17 _ForwardIterator
__lower_bound(_ForwardIterator __first, _ForwardIterator __last, const _Tp& __value_, _Compare __comp)
{
    typedef typename iterator_traits<_ForwardIterator>::difference_type difference_type;
    difference_type __len = _VSTD::distance(__first, __last);
    while (__len != 0)
    {
        difference_type __l2 = _VSTD::__half_positive(__len);
        _ForwardIterator __m = __first;
        _VSTD::advance(__m, __l2);
        if (__comp(*__m, __value_))
        {
            __first = ++__m;
            __len -= __l2 + 1;
        }
        else
            __len = __l2;
    }
    return __first;
}
```

컴파일러가 추상화 제거에 성공한다면 대략 비슷한 머신 코드로 컴파일되며, 배열 크기에 따라 [예상대로](/hpc/cpu-cache/latency) 증가하는 거의 동일한 평균 지연 시간을 보여줍니다.

![](../img/search-std.svg)

대부분의 사람들은 이진 탐색을 직접 구현하지 않으므로 Clang의 `std::lower_bound`를 기준(baseline)으로 사용하겠습니다.

### 병목 현상 (The Bottleneck)

최적화된 구현으로 넘어가기 전에, 왜 이진 탐색이 느린지 간단히 논의해 보겠습니다.

[perf](/hpc/profiling/events)로 `std::lower_bound`를 실행해 보면, 대부분의 시간을 [조건부 점프(conditional jump)](/hpc/architecture/loops) 인스트럭션에서 보내는 것을 볼 수 있습니다.

```nasm
       │35:   mov    %rax,%rdx
  0.52 │      sar    %rdx
  0.33 │      lea    (%rsi,%rdx,4),%rcx
  4.30 │      cmp    (%rcx),%edi
 65.39 │    ↓ jle    b0
  0.07 │      sub    %rdx,%rax
  9.32 │      lea    0x4(%rcx),%rsi
  0.06 │      dec    %rax
  1.37 │      test   %rax,%rax
  1.11 │    ↑ jg     35
```

이 [파이프라인 스톨(pipeline stall)](/hpc/)은 탐색 진행을 멈추게 하며, 주로 두 가지 [요인](/hpc/pipelining/hazards)에 의해 발생합니다.

- 예측이 불가능한 [분기(branch)](/hpc/pipelining/branching)로 인해 *제어 해저드(control hazard)*가 발생합니다 (쿼리와 키가 무작위로 독립적으로 선택되기 때문입니다). 프로세서는 분기 예측 실패 시마다 파이프라인을 비우고 다시 채우기 위해 10~15 사이클 동안 멈춰야 합니다.
- 이전 비교가 완료될 때까지 기다려야 하므로 *데이터 해저드(data hazard)*가 발생합니다. 비교 연산은 피연산자 중 하나를 메모리에서 가져올 때까지 기다려야 하며, 데이터 위치에 따라 [0에서 300 사이클](/hpc/cpu-cache/latency)이 걸릴 수 있습니다.

이제 이러한 장애물들을 하나씩 제거해 보겠습니다.

## 분기 제거 (Removing Branches)

분기를 [조건부 실행(predication)](/hpc/pipelining/branchless)으로 대체할 수 있습니다. 작업을 더 쉽게 만들기 위해 STL 방식을 채택하여 두 끝점 대신 첫 번째 요소와 탐색 구간의 크기를 사용하여 루프를 다시 작성할 수 있습니다.

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        if (base[half - 1] < x) {
            base += half;
            len = len - half;
        } else {
            len = half;
        }
    }
    return *base;
}
```

매 반복마다 `len`은 기본적으로 절반으로 줄어들며, 비교 결과에 따라 내림(floor) 또는 올림(ceil)됩니다. 이 조건부 업데이트는 불필요해 보입니다. 이를 피하기 위해 항상 올림된다고 가정할 수 있습니다.

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        if (base[half - 1] < x)
            base += half;
        len -= half; // = ceil(len / 2)
    }
    return *base;
}
```

이렇게 하면 [조건부 이동(conditional move, cmov)](/hpc/pipelining/branchless/)을 사용하여 탐색 구간의 첫 번째 요소만 업데이트하고 매 반복마다 크기를 절반으로 줄이면 됩니다.

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        base += (base[half - 1] < x) * half; // "cmov"로 대체됨
        len -= half;
    }
    return *base;
}
```

<!-- pre-compute base pointer for next iteration? -->

이 루프가 항상 표준 이진 탐색과 동일한 것은 아닙니다. 탐색 구간의 크기를 항상 올림하기 때문에 약간 다른 요소에 접근하며 필요한 것보다 한 번 더 비교를 수행할 수도 있습니다. 각 반복에서의 계산을 단순화하는 것 외에도, 배열 크기가 일정하면 반복 횟수도 일정하게 만들어 분기 예측 실패를 완전히 제거합니다.

이러한 트릭은 컴파일러 최적화에 매우 민감합니다. 컴파일러나 함수 호출 방식에 따라 여전히 분기가 남거나 비효율적인 코드가 생성될 수 있습니다. Clang 10에서는 잘 작동하여 작은 배열에서 2.5~3배의 성능 향상을 보여줍니다.

<!-- todo: update numbers -->

![](../img/search-branchless.svg)

한 가지 흥미로운 점은 큰 배열에서는 성능이 더 나쁘다는 것입니다. 이상해 보일 수 있습니다. 전체 지연 시간은 RAM 지연 시간에 의해 지배되며, 표준 이진 탐색과 거의 동일한 메모리 접근을 수행하므로 성능도 비슷하거나 약간 더 좋아야 하기 때문입니다.

여기서 던져야 할 진짜 질문은 "왜 분기 없는 구현이 더 나쁜가"가 아니라 "왜 분기가 있는 버전이 더 좋은가"입니다. 분기가 있으면 CPU가 분기 중 하나를 [추측(speculate)](/hpc/pipelining/branching/)하고 그것이 맞는지 확인하기도 전에 왼쪽 또는 오른쪽 키를 가져오기 시작할 수 있기 때문입니다. 이는 사실상 암시적인 [프리페칭(prefetching)](/hpc/cpu-cache/prefetching) 역할을 합니다.

분기 없는 구현에서는 `cmov`가 다른 일반 인스트럭션처럼 처리되므로 이런 일이 일어나지 않으며, 분기 예측기가 미래를 예측하기 위해 피연산자를 들여다보지 않습니다. 이를 보완하기 위해 왼쪽과 오른쪽 자식 키를 명시적으로 요청하여 소프트웨어적으로 데이터를 프리페치할 수 있습니다.

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        len -= half;
        __builtin_prefetch(&base[len / 2 - 1]);
        __builtin_prefetch(&base[half + len / 2 - 1]);
        base += (base[half - 1] < x) * half;
    }
    return *base;
}
```

<!-- todo: rerun this too -->

프리페칭을 사용하면 큰 배열에서도 성능이 거의 동일해집니다.

![](../img/search-branchless-prefetch.svg)

분기가 있는 버전은 "손자", "증손자" 노드 등도 프리페치하므로 그래프가 여전히 더 가파르게 상승합니다. 비록 예측이 맞을 확률이 점점 낮아짐에 따라 새로운 추측성 읽기의 유용성은 기하급수적으로 감소하지만 말이죠.

분기 없는 버전에서도 한 레이어 이상 앞서 가져올 수 있지만, 필요한 프리페치 횟수도 기하급수적으로 늘어납니다. 대신 메모리 작업을 최적화하기 위해 다른 접근 방식을 시도해 보겠습니다.

## 레이아웃 최적화 (Optimizing the Layout)

이진 탐색 중에 수행하는 메모리 요청은 매우 특정한 접근 패턴을 형성합니다.

![](../img/binary-search.png)

각 요청의 요소가 캐싱되어 있을 확률은 얼마나 될까요? [데이터 지역성(data locality)](/hpc/external-memory/locality/)은 얼마나 좋을까요?

- *공간 지역성(Spatial locality)*은 동일한 [캐시 라인](/hpc/cpu-cache/cache-lines)에 있을 가능성이 높은 마지막 3~4개 요청에 대해서는 괜찮아 보입니다. 하지만 이전의 모든 요청은 거대한 메모리 점프를 필요로 합니다.
- *시간 지역성(Temporal locality)*은 처음 10여 개의 요청에 대해 괜찮아 보입니다. 이 정도 길이의 서로 다른 비교 시퀀스는 그리 많지 않으므로, 캐싱되어 있을 가능성이 높은 동일한 중간 요소들과 반복적으로 비교하게 될 것이기 때문입니다.

두 번째 유형의 캐시 공유가 얼마나 중요한지 설명하기 위해, 매 반복마다 중간 요소 대신 탐색 구간 내에서 무작위로 요소를 선택해 보겠습니다.

```c++
int lower_bound(int x) {
    int l = 0, r = n - 1;
    while (l < r) {
        int m = l + rand() % (r - l);
        if (t[m] >= x)
            r = m;
        else
            l = m + 1;
    }
    return t[l];
}
```

[이론적으로](#appendix-random-binary-search) 이 무작위 이진 탐색은 일반적인 방식보다 약 30~40% 더 많은 비교를 수행할 것으로 예상되지만, 실제 컴퓨터에서는 큰 배열에 대해 실행 시간이 약 6배 증가합니다.

![](../img/search-random.svg)

이는 단순히 `rand()` 호출이 느리기 때문만은 아닙니다. 메모리 지연 시간이 난수 생성 및 [나머지 연산(modulo)](/hpc/arithmetic/division)의 비용을 압도하는 L2-L3 경계 지점을 명확히 볼 수 있습니다. 가져온 요소 중 일부뿐만 아니라 거의 모든 요소가 캐싱되어 있을 가능성이 낮기 때문에 성능이 저하되는 것입니다.

또 다른 잠재적인 부정적 효과는 [캐시 연관성(cache associativity)](/hpc/cpu-cache/associativity)입니다. 배열 크기가 2의 큰 거듭제곱의 배수라면, "핫(hot)"한 요소들의 인덱스도 2의 큰 거듭제곱으로 나누어떨어져 동일한 캐시 라인에 매핑되고 서로를 밀어내게 됩니다. 예를 들어 크기가 $2^{20}$인 배열에 대한 이진 탐색은 쿼리당 약 360ns가 걸리는 반면, 크기가 $(2^{20} + 123)$인 배열은 약 300ns가 걸립니다. 20%나 차이가 나죠. 이 문제를 해결하는 [방법](https://en.wikipedia.org/wiki/Fibonacci_search_technique)이 있지만, 더 시급한 문제에서 눈을 돌리지 않기 위해 그냥 무시하겠습니다. 우리가 사용하는 모든 배열 크기는 $\lfloor 1.17^k \rfloor$ 형태이므로 캐시 부수 효과가 발생할 가능성은 낮습니다.

우리 메모리 레이아웃의 진짜 문제는 핫한 요소와 콜드(cold)한 요소를 함께 그룹화하기 때문에 시간 지역성을 가장 효율적으로 활용하지 못한다는 것입니다. 예를 들어 매 쿼리마다 가장 먼저 요청하는 요소인 $\lfloor n/2 \rfloor$를 거의 요청하지 않는 $\lfloor n/2 \rfloor + 1$과 동일한 캐시 라인에 저장할 가능성이 높습니다.

<!--  (sometimes literally never — if it is the first element in a search range of three, and it is indeed the lower bound, we just compare against the middle and deduce it has to be the first element without ever even fetching it) — this is not true -->

다음은 31개 요소 배열에 대한 예상 비교 빈도를 시각화한 히트맵입니다.

![](../img/binary-heat.png)

따라서 이상적으로는 핫한 요소는 핫한 요소끼리, 콜드한 요소는 콜드한 요소끼리 그룹화하는 메모리 레이아웃을 원할 것입니다. 그리고 요소들의 번호를 다시 매겨서 더 캐시 친화적인 방식으로 배열을 재배치하면 이를 달성할 수 있습니다. 우리가 사용할 번호 체계는 사실 500년이나 된 것이며, 이미 알고 계실 확률이 높습니다.

### 에이칭어 레이아웃 (Eytzinger Layout)

**미하엘 에이칭어(Michaël Eytzinger)**는 16세기 오스트리아의 귀족으로 계보학에 관한 업적, 특히 *안넨타펠(ahnentafel, 독일어로 조상 표)*이라고 불리는 조상 번호 매기기 체계로 잘 알려져 있습니다.

그 당시에는 가문이 매우 중요했지만 계보 데이터를 기록하는 것은 비용이 많이 들었습니다. *안넨타펠*은 다이어그램을 그려 공간을 낭비하지 않고도 한 사람의 계보를 콤팩트하게 표시할 수 있게 해줍니다.

한 사람의 직계 조상을 고정된 상승 순서로 나열합니다. 먼저 본인을 1번으로 나열하고, 재귀적으로 $k$번으로 번호가 매겨진 각 사람에 대해 그 아버지는 $2k$, 어머니는 $2k+1$로 나열합니다.

여기 [표트르 대제](https://en.wikipedia.org/wiki/Peter_the_Great)의 증손자인 [파벨 1세](https://en.wikipedia.org/wiki/Paul_I_of_Russia)의 예가 있습니다.

1. 파벨 1세
2. 표트르 3세 (파벨의 아버지)
3. [예카테리나 2세](https://en.wikipedia.org/wiki/Catherine_the_Great) (파벨의 어머니)
4. 카를 프리드리히 (표트르의 아버지, 파벨의 친할아버지)
5. 안나 페트로브나 (표트르의 어머니, 파벨의 친할머니)
6. 크리스티안 아우구스트 (예카테리나의 아버지, 파벨의 외할아버지)
7. 요한나 엘리자베트 (예카테리나의 어머니, 파벨의 외할머니)

콤팩트하다는 것 외에도, 모든 짝수 번호는 남성이고 모든 홀수 번호(1번 제외 가능)는 여성이라는 등의 좋은 속성을 가지고 있습니다. 또한 후손들의 성별만 알면 특정 조상의 번호를 찾을 수도 있습니다. 예를 들어 표트르 대제의 혈통은 파벨 1세 → 표트르 3세 → 안나 페트로브나 → 표트르 대제이므로, 그의 번호는 $((1 \times 2) \times 2 + 1) \times 2 = 10$이 됩니다.

**컴퓨터 과학에서** 이 번호 매기기 방식은 힙(heap), 세그먼트 트리(segment tree) 등 포인터가 없는 이진 트리 구조의 구현에 널리 사용되어 왔습니다.

이진 탐색에 이 레이아웃을 적용하면 다음과 같습니다.

![트리가 약간 불균형하다는 점에 유의하세요 (마지막 레이어가 연속적이기 때문)](../img/eytzinger.png)

이 레이아웃에서 탐색할 때는 배열의 첫 번째 요소부터 시작하여, 비교 결과에 따라 매번 $2k$ 또는 $2k+1$로 점프하면 됩니다.

![](../img/eytzinger-search.png)

루트에 가까운 요소일수록 배열의 시작 부분에 가까워지므로 캐시에서 가져올 가능성이 높아져 시간 지역성이 더 좋아짐(실제로 이론적으로 최적임)을 즉시 알 수 있습니다.

![](../img/eytzinger-heat.png)

또 다른 관점에서 보면, 새 배열의 끝에 모든 짝수 인덱스 요소를 쓰고, 남은 요소 중 모든 짝수 인덱스 요소를 그 바로 앞에 쓰는 식으로 루트를 첫 번째 요소로 배치할 때까지 반복하는 것과 같습니다.

### 구조 (Construction)

에이칭어 배열을 만들기 위해 $O(\log n)$번의 필터링을 수행할 수도 있지만(아마 이것이 가장 빠른 접근 방식일 것입니다), 간결함을 위해 원래의 탐색 트리를 순회하며 구축해 보겠습니다.

```c++
int a[n], t[n + 1]; // 원래의 정렬된 배열과 우리가 구축할 에이칭어 배열
//              ^ 1부터 시작하는 인덱싱 때문에 요소가 하나 더 필요함

void eytzinger(int k = 1) {
    static int i = 0; // <- 여러 배열에 대해 실행할 때 주의
    if (k <= n) {
        eytzinger(2 * k);
        t[k] = a[i++];
        eytzinger(2 * k + 1);
    }
}
```

이 함수는 현재 노드 번호 `k`를 받아 탐색 구간 중간의 왼쪽에 있는 모든 요소를 재귀적으로 쓰고, 현재 요소를 쓴 다음, 오른쪽에 있는 모든 요소를 재귀적으로 씁니다. 약간 복잡해 보일 수 있지만, 이것이 작동한다는 것을 확신하려면 다음 세 가지 관찰만 있으면 됩니다.

- 1부터 $n$까지의 각 `k`에 대해 `if` 본문에 단 한 번만 진입하므로 정확히 $n$개의 요소를 씁니다.
- 매번 `i` 포인터를 증가시키므로 원래 배열의 요소를 순차적으로 씁니다.
- 노드 `k`에 요소를 쓰는 시점에는 이미 그 왼쪽의 모든 요소(정확히 `i`개)를 쓴 상태입니다.

재귀적임에도 불구하고 모든 메모리 읽기가 순차적이고 메모리 쓰기도 한 번에 $O(\log n)$개의 서로 다른 블록에서만 발생하므로 실제로는 꽤 빠릅니다. 하지만 이 레이아웃을 유지하는 것은 논리적으로나 계산적으로나 더 어렵습니다. 정렬된 배열에 요소를 추가하는 것은 접미사를 한 칸씩 옮기기만 하면 되지만, 에이칭어 배열은 사실상 처음부터 다시 구축해야 합니다.

이 순회와 그 결과인 순열이 바닐라 이진 탐색의 "트리"와 정확히 동일하지는 않다는 점에 유의하세요. 예를 들어 왼쪽 자식 서브트리가 오른쪽보다 최대 2배까지 클 수 있습니다. 하지만 두 방식 모두 동일한 $\lceil \log_2 n \rceil$ 트리 깊이를 가지므로 큰 문제는 아닙니다.

또한 에이칭어 배열은 1부터 시작하는 인덱스를 사용하며, 이는 나중에 성능 면에서 중요해집니다. 0번 요소에는 하한(lower bound)이 존재하지 않을 때 반환할 값(예: `std::lower_bound`의 `a.end()`)을 넣어둘 수 있습니다.

### 탐색 구현 (Search Implementation)

이제 인덱스만 사용하여 이 배열을 내려갈 수 있습니다. $k=1$로 시작하여 왼쪽으로 가야 하면 $k := 2k$, 오른쪽으로 가야 하면 $k := 2k+1$을 실행하면 됩니다. 탐색 경계를 저장하고 재계산할 필요조차 없습니다. 이러한 단순함 덕분에 분기를 피할 수 있습니다.

```c++
int k = 1;
while (k <= n)
    k = 2 * k + (t[k] < x);
```

문제는 $k$가 결과 요소를 직접 가리키지 않기 때문에 인덱스를 복구해야 한다는 점입니다. 다음 예를 보십시오 (해당 트리는 위에 나열되어 있습니다).

<!--
    array:  0 1 2 3 4 5 6 7 8 9                           
eytzinger:  6 3 7 1 5 8 9 0 2 4                           
1st range:  -------------------  k := 1                    
2nd range:  -------------        k := 2*k     = 2   (6 ≥ 3)
3rd range:  -------              k := 2*k     = 4   (3 ≥ 3)
4th range:      ---              k := 2*k + 1 = 9   (1 < 3)
5th range:        -              k := 2*k + 1 = 19  (2 < 3)
-->

<pre class='center-pre'>
    array:  0 1 2 3 4 5 6 7 8 9                            
eytzinger:  <u>6</u> <u>3</u> 7 <u>1</u> 5 8 9 0 <u>2</u> 4                            
1st range:  ------------?------  k := 2*k     = 2   (6 ≥ 3)
2nd range:  ------?------        k := 2*k     = 4   (3 ≥ 3)
3rd range:  --?----              k := 2*k + 1 = 9   (1 < 3)
4th range:      ?--              k := 2*k + 1 = 19  (2 < 3)
5th range:        !                                        
</pre>

<!-- do we need the last comparison? -->

여기서는 $[0, …, 9]$ 배열에서 $x=3$에 대한 하한을 쿼리합니다. 6, 3, 1, 2와 비교하고 왼쪽-왼쪽-오른쪽-오른쪽으로 이동하여 $k = 19$로 끝나는데, 이는 유효한 배열 인덱스조차 아닙니다.

여기서 트릭은, 답이 배열의 마지막 요소가 아닌 한 언젠가는 $x$와 비교하게 된다는 것을 알아차리는 것입니다. 그 요소가 $x$보다 작지 않다는 것을 알게 된 후에는 왼쪽으로 정확히 한 번 이동한 다음 리프 노드에 도달할 때까지 계속 오른쪽으로 이동하게 됩니다 (그 이후로는 $x$보다 작은 요소들과만 비교하기 때문입니다). 따라서 답을 복구하려면 몇 번의 우회전과 그 앞의 한 번의 우회전을 "취소"하면 됩니다.

이는 $k$의 이진 표현에서 우회전이 1비트로 기록된다는 점을 이용해 우아하게 처리할 수 있습니다. 즉, 이진 표현에서 마지막에 연속된 1의 개수를 찾아 그만큼 비트를 오른쪽으로 시프트하고 한 번 더 시프트하면 됩니다. 이를 위해 숫자를 반전시키고(`~k`) "첫 번째 세트 비트 찾기(find first set)" 인스트럭션을 호출할 수 있습니다.

```c++
int lower_bound(int x) {
    int k = 1;
    while (k <= n)
        k = 2 * k + (t[k] < x);
    k >>= __builtin_ffs(~k);
    return t[k];
}
```

실행해 보면… 결과가 그리 좋지는 않습니다.

![](../img/search-eytzinger.svg)

작은 배열에서의 지연 시간은 분기 없는 이진 탐색 구현과 비슷하지만, 훨씬 빨리 성능이 저하되기 시작합니다. 에이칭어 이진 탐색은 공간 지역성의 이점을 누리지 못하기 때문입니다. 마지막 3~4개 요소가 더 이상 동일한 캐시 라인에 있지 않아 각각 따로 가져와야 합니다.

깊이 생각해 보면, 향상된 시간 지역성이 이를 보완해야 한다고 반박할 수 있습니다. 이전에는 캐시 라인의 $1/16$만 사용하여 하나의 핫한 요소를 저장했지만, 이제는 라인 전체를 사용하므로 유효 캐시 크기가 16배 커진 셈이고, 이는 $\log_2 16 = 4$개의 초기 요청을 더 커버할 수 있게 해줍니다.

하지만 더 생각해 보면 그것만으로는 충분하지 않다는 것을 알게 됩니다. 나머지 15개 요소를 캐싱하는 것이 완전히 무용지물은 아니었으며, 하드웨어 프리페처가 우리 요청의 인접 캐시 라인을 가져올 수도 있었기 때문입니다. 탐색의 마지막 단계였다면 읽고 있는 나머지도 캐싱된 요소일 가능성이 높습니다. 따라서 실제로는 마지막 6~7번의 접근이 캐싱되어 있을 가능성이 높지, 3~4번이 아닙니다.

이 레이아웃으로 바꾼 것이 전체적으로 어리석은 짓이었던 것 같지만, 이를 가치 있게 만드는 방법이 있습니다.

### 프리페칭 (Prefetching)

메모리 지연 시간을 숨기기 위해 분기 없는 이진 탐색에서 했던 것처럼 소프트웨어 프리페칭을 사용할 수 있습니다. 하지만 왼쪽과 오른쪽 자식 노드에 대해 두 개의 별도 프리페치 인스트럭션을 발행하는 대신, 에이칭어 배열에서는 두 자식이 서로 이웃해 있다는 점을 이용할 수 있습니다. 하나는 인덱스가 $2k$이고 다른 하나는 $2k+1$이므로 동일한 캐시 라인에 있을 가능성이 높으며, 하나의 인스트럭션만 사용하면 됩니다.

이 관찰은 노드 $k$의 손자 노드들에게도 확장됩니다. 그들 역시 순차적으로 저장됩니다.

```
2 * 2 * k           = 4 * k
2 * 2 * k + 1       = 4 * k + 1
2 * (2 * k + 1)     = 4 * k + 2
2 * (2 * k + 1) + 1 = 4 * k + 3
```

<!--
\begin{aligned}
   2 \cdot 2 \cdot k       &= 4 \cdot k
\\ 2 \cdot 2 \cdot k + 1   &= 4 \cdot k + 1
\\ 2 \cdot (2 \cdot k) + 1 &= 4 \cdot k + 2
\\ 2 \cdot (2 \cdot k + 1) + 1 &= 4 \cdot k + 3
\end{aligned}
-->

그들의 캐시 라인도 하나의 인스트럭션으로 가져올 수 있습니다. 그렇다면 직접적인 자식 대신 한 캐시 라인에 채울 수 있는 만큼의 자손들을 미리 가져오면 어떨까요? $64/4 = 16$개 요소, 즉 인덱스가 $16k$에서 $16k+15$까지인 현손(great-great-grandchildren) 노드들입니다.

이 16개 요소 중 하나만 프리페치하면 캐시 라인 경계에 걸쳐 있을 수 있으므로 일부만 가져오게 될 수도 있습니다. 첫 번째와 마지막 요소를 프리페치할 수도 있지만, 단 하나의 메모리 요청으로 해결하려면 첫 번째 요소의 인덱스인 $16k$가 16으로 나누어떨어진다는 점을 이용해야 합니다. 그러면 메모리 주소는 배열의 시작 주소에 $16 \times 4 = 64$의 배수를 더한 것이 되어 캐시 라인 크기와 일치하게 됩니다. 배열이 캐시 라인에 [정렬(aligned)](/hpc/cpu-cache/alignment)되어 있다면, 이 16개의 현손 요소들은 단일 캐시 라인에 있음이 보장됩니다.

따라서 배열을 정렬하기만 하면 됩니다.

```c++
t = (int*) std::aligned_alloc(64, 4 * (n + 1));
```

그리고 매 반복마다 인덱스 $16k$인 요소를 프리페치합니다.

```c++
int lower_bound(int x) {
    int k = 1;
    while (k <= n) {
        __builtin_prefetch(t + k * 16);
        k = 2 * k + (t[k] < x);
    }
    k >>= __builtin_ffs(~k);
    return t[k];
}
```

큰 배열에서의 성능은 이전 버전보다 3~4배, `std::lower_bound`보다 약 2배 향상됩니다. 단 두 줄의 코드를 추가한 것치고는 나쁘지 않죠.

![](../img/search-eytzinger-prefetch.svg)

본질적으로 우리가 하는 일은 네 단계 앞을 프리페치하여 메모리 요청을 중첩시킴으로써 지연 시간을 숨기는 것입니다. 이론적으로 연산 비용이 문제가 되지 않는다면 4배의 속도 향상을 기대할 수 있겠지만, 실제로는 좀 더 완만한 속도 향상을 얻습니다.

네 단계보다 더 멀리 프리페치할 수도 있으며, 이를 위해 하나 이상의 프리페치 인스트럭션을 사용할 필요도 없습니다. 첫 번째 캐시 라인만 요청하고 나머지는 하드웨어가 인접한 라인을 프리페치하도록 맡길 수 있습니다. 하드웨어에 따라 성능이 개선될 수도 있고 아닐 수도 있습니다.

```c++
__builtin_prefetch(t + k * 32);
```

또한 마지막 몇 번의 프리페치 요청은 실제로는 필요하지 않으며, 심지어 프로그램에 할당된 메모리 영역 밖일 수도 있습니다. 대부분의 현대적인 CPU에서 유효하지 않은 프리페치 인스트럭션은 no-op으로 변환되므로 문제가 되지 않지만, 일부 플랫폼에서는 속도 저하를 일으킬 수 있습니다. 따라서 루프에서 마지막 4회 정도의 반복을 분리하여 프리페치를 제거하는 것이 합리적일 수 있습니다.

이 프리페칭 기술은 최대 4단계 앞의 요소를 읽을 수 있게 해주지만, 공짜는 아닙니다. 초과 메모리 [대역폭(bandwidth)](/hpc/cpu-cache/bandwidth)을 대가로 [지연 시간(latency)](/hpc/cpu-cache/latency)을 줄이는 것이기 때문입니다. 별도의 하드웨어 스레드에서 두 개 이상의 인스턴스를 실행하거나 백그라운드에서 메모리 집약적인 다른 연산을 수행한다면 벤치마크 성능에 큰 [영향](/hpc/cpu-cache/sharing)을 미칠 것입니다.

하지만 더 잘할 수 있습니다. 4개의 캐시 라인을 한꺼번에 가져오는 대신, 4배 *적은* 캐시 라인을 가져올 수도 있습니다. [다음 섹션](../s-tree)에서 그 방법을 살펴보겠습니다.

<!--

But that was a small detour. Let's get back to optimizing for *large* arrays.

[Part 2](https://algorithmica.org/en/b-tree) explores efficient implementation of implicit static B-trees in bandwidth-constrained environment.

-->

### 마지막 분기 제거 (Removing the Last Branch)

마지막 마무리: 에이칭어 탐색 그래프의 울퉁불퉁한 부분을 보셨나요? 이는 무작위 노이즈가 아닙니다. 확대해 보겠습니다.

![](../img/search-eytzinger-small.svg)

배열 크기가 $1.5 \cdot 2^k$ 형태일 때 지연 시간이 약 10ns 더 높습니다. 이는 루프 자체의 마지막 분기에서 발생한 분기 예측 실패 때문입니다. 정확히는 마지막 분기입니다. 배열 크기가 2의 거듭제곱에서 멀어지면 루프가 $\lfloor \log_2 n \rfloor$번 반복될지 $\lfloor \log_2 n \rfloor + 1$번 반복될지 예측하기 어려워져, 50%의 확률로 정확히 한 번의 분기 예측 실패를 겪게 됩니다.

이를 해결하는 한 가지 방법은 배열을 가장 가까운 2의 거듭제곱 크기로 무한대 값을 채워 패딩하는 것이지만, 이는 메모리를 낭비합니다. 대신 항상 일정한 최소 횟수만큼 루프를 실행하고, 조건부 실행을 사용하여 선택적으로 더미(dummy) 요소와 마지막 비교를 수행함으로써 마지막 분기를 제거할 수 있습니다.

```c++
t[0] = -1; // x보다 작은 요소
iters = std::__lg(n + 1);

int lower_bound(int x) {
    int k = 1;

    for (int i = 0; i < iters; i++)
        k = 2 * k + (t[k] < x);

    int *loc = (k <= n ? t + k : t);
    k = 2 * k + (*loc < x);

    k >>= __builtin_ffs(~k);

    return t[k];
}
```

이제 그래프가 매끄러워졌으며, 작은 배열에서는 분기 없는 이진 탐색보다 불과 몇 사이클 느릴 뿐입니다.

![](../img/search-eytzinger-branchless.svg)

흥미롭게도 이제 GCC는 분기를 `cmov`로 교체하지 못하지만 Clang은 성공합니다. 1-1입니다.

### 부록: 무작위 이진 탐색 (Appendix: Random Binary Search)

무작위 이진 탐색에서 예상되는 정확한 비교 횟수를 찾는 것은 그 자체로 매우 흥미로운 수학 문제입니다. 직접 한번 풀어보세요!

*알고리즘적으로* 계산하는 방법은 동적 프로그래밍을 이용하는 것입니다. 크기 $n$의 탐색 구간에서 무작위 하한을 찾는 데 필요한 예상 비교 횟수를 $f_n$이라고 하면, 가능한 모든 $(n-1)$개의 분할을 고려하여 이전의 $f_n$들로부터 계산할 수 있습니다.

$$
f_n = \sum_{l = 1}^{n - 1} \frac{1}{n-1} \cdot \left( f_l \cdot \frac{l}{n} + f_{n - l} \cdot \frac{n - l}{n} \right) + 1
$$

이 공식을 직접 적용하면 $O(n^2)$ 알고리즘이 되지만, 합을 다음과 같이 재배치하여 최적화할 수 있습니다.

$$
\begin{aligned}
f_n &= \sum_{i = 1}^{n - 1} \frac{ f_i \cdot i + f_{n - i} \cdot (n - i) }{ n \cdot (n - 1) } + 1
\\  &= \frac{2}{n \cdot (n - 1)} \cdot \sum_{i = 1}^{n - 1} f_i \cdot i + 1
\end{aligned}
$$

$f_n$을 업데이트하려면 모든 $i < n$에 대한 $f_i \cdot i$의 합만 계산하면 됩니다. 이를 위해 두 개의 새로운 변수를 도입해 보겠습니다.

$$
g_n = f_n \cdot n,
\;\;
s_n = \sum_{i=1}^{n} g_n
$$

이제 다음과 같이 순차적으로 계산할 수 있습니다.

$$
\begin{aligned}
g_n &= f_n \cdot n
     = \frac{2}{n-1} \cdot \sum_{i = 1}^{n - 1} g_i + n
     = \frac{2}{n - 1} \cdot s_{n - 1} + n
\\ s_n &= s_{n - 1} + g_n
\end{aligned}
$$

이렇게 하면 $O(n)$ 알고리즘을 얻게 되지만, 더 개선할 수 있습니다. $s_n$의 업데이트 공식에 $g_n$을 대입해 봅시다.

$$
\begin{aligned}
s_n &= s_{n - 1} + \frac{2}{n - 1} \cdot s_{n - 1} + n
\\  &= (1 + \frac{2}{n - 1}) \cdot s_{n - 1} + n
\\  &= \frac{n + 1}{n - 1} \cdot s_{n - 1} + n
\end{aligned}
$$

<!-- todo: can we simplify the proof and get rid of r? -->

다음 트릭은 좀 더 복잡합니다. $r_n$을 다음과 같이 정의합니다.

$$
\begin{aligned}
r_n &= \frac{s_n}{n}
\\  &= \frac{1}{n} \cdot \left(\frac{n + 1}{n - 1} \cdot s_{n - 1} + n\right)
\\  &= \frac{n + 1}{n} \cdot \frac{s_{n - 1}}{n - 1} + 1
\\  &= \left(1 + \frac{1}{n}\right) \cdot r_{n - 1} + 1
\end{aligned}
$$

이를 이전에 얻은 $g_n$ 공식에 대입할 수 있습니다.

$$
g_n = \frac{2}{n - 1} \cdot s_{n - 1} + n = 2 \cdot r_{n - 1} + n
$$

$g_n = f_n \cdot n$임을 상기하면, $f_n$을 사용하여 $r_{n - 1}$을 표현할 수 있습니다.

$$
f_n \cdot n = 2 \cdot r_{n - 1} + n
\implies
r_{n - 1} = \frac{(f_n - 1) \cdot n}{2}
$$

마지막 단계입니다. 방금 $r_n$을 $r_{n - 1}$을 통해 표현했고, $r_{n - 1}$을 $f_n$을 통해 표현했습니다. 이를 통해 $f_{n + 1}$을 $f_n$을 통해 표현할 수 있습니다.

$$
\begin{aligned}
&&\quad r_n &= \left(1 + \frac{1}{n}\right) \cdot r_{n - 1} + 1
\\ &\Rightarrow & \frac{(f_{n + 1} - 1) \cdot (n + 1)}{2} &= \left(1 + \frac{1}{n}\right) \cdot \frac{(f_n - 1) \cdot n}{2} + 1
\\ &&&= \frac{n + 1}{2} \cdot (f_n - 1) + 1
\\ &\Rightarrow & (f_{n + 1} - 1) &= (f_{n} - 1) + \frac{2}{n + 1}
\\ &\Rightarrow &f_{n + 1} &= f_{n} + \frac{2}{n + 1}
\\ &\Rightarrow &f_{n} &= f_{n - 1} + \frac{2}{n}
\\ &\Rightarrow &f_{n} &= \sum_{k = 2}^{n} \frac{2}{k}
\end{aligned}
$$

마지막 식은 [조화 급수(harmonic series)](https://en.wikipedia.org/wiki/Harmonic_series_(mathematics))의 두 배이며, 이는 $n \to \infty$일 때 $\ln n$에 근사하는 것으로 잘 알려져 있습니다. 따라서 무작위 이진 탐색은 일반적인 방식보다 $\frac{2 \ln n}{\log_2 n} = 2 \ln 2 \approx 1.386$배 더 많은 비교를 수행하게 됩니다.

### 감사의 말 (Acknowledgements)

이 글은 Paul-Virak Khuong과 Pat Morin의 "[Array Layouts for Comparison-Based Searching](https://arxiv.org/pdf/1509.05053.pdf)"을 기반으로 하고 있습니다. 46페이지에 걸쳐 이 방식들과 다른 여러 (덜 성공적인) 접근 방식들을 자세히 다루고 있습니다. 제가 가장 좋아하는 성능 공학 논문 중 하나이므로 꼭 한번 읽어보시기를 추천합니다.

무작위 이진 탐색의 증명을 [제공](https://github.com/algorithmica-org/algorithmica/issues/57)해 주신 Marshall Lochbaum에게 감사드립니다. 저 혼자서는 절대 해내지 못했을 것입니다.

오래전 어느 블로그에서 이 멋진 레이아웃 시각화 자료를 가져왔는데, 블로그 이름이나 라이선스가 기억나지 않고 이미지 역검색으로도 더 이상 찾을 수 없었습니다. 누군지는 모르겠지만 소송을 걸지 않으신다면 감사하겠습니다!
