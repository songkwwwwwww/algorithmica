---
title: 세그먼트 트리 (Segment Trees)
weight: 4
---

[이진 탐색](../binary-search)과 [정적 B-트리](../s-tree)를 최적화하며 얻은 교훈은 광범위한 자료 구조에 적용될 수 있습니다.

이 글에서는 STL에 있는 무언가를 다시 최적화하는 대신, 일반적인 프로그래머나 심지어 대부분의 컴퓨터 과학 연구자들에게도 생소할 수 있지만[^tcs] 프로그래밍 대회에서는 그 속도와 구현의 단순함 덕분에 [매우 광범위하게](https://www.google.com/search?q=segment+tree+site%3Acodeforces.com) 사용되는 *세그먼트 트리(segment trees)*에 집중해 보겠습니다.

[^tcs]: 세그먼트 트리는 비교적 최근(~2000년)에 발명되었고, [다른 이진 트리](https://en.wikipedia.org/wiki/Tree_(data_structure))가 할 수 없는 일을 딱히 하지 않으며, *점근적으로* 더 빠르지도 않기 때문에 이론 컴퓨터 과학 문헌에서는 거의 언급되지 않습니다. 하지만 실제로는 속도 면에서 압승하는 경우가 많습니다.

(이미 문맥을 알고 계신다면, 페닉 트리보다 4~12배 더 빠른 새로운 구조인 *와이드 세그먼트 트리(wide segment tree)*에 대한 내용이 있는 [마지막 섹션](#wide-segment-trees)으로 바로 넘어가셔도 좋습니다.)

### 동적 구간 합 (Dynamic Prefix Sum)

<!--

This is a long article, and to make it less long, we will mostly be focusing on its simplest application -->

세그먼트 트리는 멋지고 많은 일을 할 수 있지만, 이 글에서는 가장 단순하면서도 중요한 응용 사례인 *동적 구간 합 문제(dynamic prefix sum problem)*에 집중하겠습니다.

```cpp
void add(int k, int x); // a[k] += x 에 대응 (0-기반 인덱싱)
int sum(int k);         // 처음 k개 요소의 합을 반환 (0부터 k - 1까지)
```

두 종류의 쿼리를 지원해야 하므로 최적화 문제는 다차원이 되며, 최적의 솔루션은 쿼리의 분포에 따라 달라집니다. 예를 들어, 한 종류의 쿼리가 매우 드물다면 다른 쪽만 최적화하면 되는데 이는 비교적 쉽습니다.

- *배열 업데이트* 비용만 신경 쓴다면, 배열을 있는 그대로 저장하고 `sum` 쿼리가 올 때마다 직접 [합을 계산](/hpc/simd/reduction)하면 됩니다.
- *구간 합 쿼리* 비용만 신경 쓴다면, 구간 합을 미리 계산해 두고 업데이트가 발생할 때마다 [처음부터 다시 계산](/hpc/algorithms/prefix)하면 됩니다.

이 두 옵션 모두 한 쪽 쿼리에는 $O(1)$의 작업을 수행하지만 다른 쪽에는 $O(n)$의 작업을 수행합니다. 쿼리 빈도가 비슷할 때는 한 쪽의 성능을 조금 희생해서 다른 쪽의 성능을 높이는 절충안을 찾을 수 있습니다. 세그먼트 트리가 바로 그 역할을 하며, 두 쿼리 모두 $O(\log n)$의 작업량을 달성합니다.

### 세그먼트 트리 구조 (Segment Tree Structure)

세그먼트 트리의 핵심 아이디어는 다음과 같습니다.

- 전체 배열의 합을 계산하여 어딘가에 적어둡니다.
- 배열을 두 반으로 나누고, 각 반의 합을 계산하여 어딘가에 적어둡니다.
- 이 반절들을 다시 반으로 나누고, 총 4개의 합을 계산하여 적어둡니다.
- …이 과정을 재귀적으로 길이가 1인 구간에 도달할 때까지 반복합니다.

이렇게 계산된 부분 구간 합들은 논리적으로 이진 트리로 표현될 수 있으며, 이것이 우리가 *세그먼트 트리*라고 부르는 것입니다.

![A segment tree with with the nodes relevant for the sum(11) and add(10) queries highlighted](../img/segtree-path.png)

세그먼트 트리는 다음과 같은 좋은 속성을 갖습니다.

- 기본 배열의 요소가 $n$개일 때, 세그먼트 트리는 정확히 $(2n - 1)$개의 노드(리프 $n$개, 내부 노드 $n-1$개)를 갖습니다. 각 내부 노드가 구간을 둘로 나누기 때문입니다.
- 트리의 높이는 $\Theta(\log n)$입니다. 루트에서 시작하여 다음 레벨로 갈수록 노드 수는 대략 두 배가 되고 구간 크기는 대략 절반이 됩니다.
- 각 구간은 세그먼트 트리의 노드에 대응하는 $O(\log n)$개의 겹치지 않는 구간으로 나눌 수 있습니다. 각 레이어에서 최대 두 개씩만 필요합니다.

$n$이 2의 거듭제곱이 아닐 때 모든 레벨이 완전히 채워지지 않을 수도 있지만, 이러한 속성들은 변함없이 유지됩니다. 첫 번째 속성 덕분에 트리를 저장하는 데 $O(n)$의 메모리만 사용하면 되고, 마지막 두 속성 덕분에 문제를 $O(\log n)$ 시간에 해결할 수 있습니다.

- `add(k, x)` 쿼리는 요소 `k`를 포함하는 구간을 가진 모든 노드에 값 `x`를 더함으로써 처리할 수 있으며, 이러한 노드는 $O(\log n)$개뿐입니다.
- `sum(k)` 쿼리는 `[0, k)` 구간을 구성하는 노드들을 찾아 그 값들을 합산함으로써 응답할 수 있으며, 이 역시 최대 $O(\log n)$개의 노드만 방문하면 됩니다.

하지만 이것은 여전히 이론입니다. 나중에 보게 되겠지만, 이 자료 구조를 구현하는 방법은 놀라울 정도로 다양합니다.

<!--

Note that by the same logic that each prefix can be covered by $O(\log n)$ nodes, each possible segment can also be covered by $O(\log n)$ nodes: you just possibly need at most two of them on each level. This lets us compute sums on any segment, although in this article we are not going to do it, instead reducing it to computing two prefix sums (from the right border, and the subtracting the prefix sum on the left border).

This is a general idea. Many different implementations possible, which we will explore one by one in this article.

Segment trees are built recursively: build a tree for left and right halves and merge results to get root.

Depending on the relative frequencies of the query types, the optimal solution may differ.

One way to do this is through a trick commonly called *square root decomposition*: we split the array (of size $n$) into blocks of approximately $\sqrt n$ elements,

sqrt

Enable [hugepages](/hpc/cpu-cache/paging) system-wide and forget about it.

Most of examples in this section are about optimizing some algorithms that are either included in standard library or take under 10 lines of code to implement naively, but we will start off with a bit more obscure example.

There are many things segment trees can do. Persistent structures, computational geometry. But for most of this article, we will focus on the dynamic (as opposed to static) prefix sum problem.

Segment trees are used for windowing queries or range queries in general, either by themselves or as part of a larger algorithm.

Functional programming, e.g., for implementing persistent arrays and derived structures.

-->

### 포인터 기반 구현 (Pointer-Based Implementation)

세그먼트 트리를 구현하는 가장 직접적인 방법은 구간 경계, 합, 자식 노드 포인터를 포함하여 필요한 모든 정보를 노드에 명시적으로 저장하는 것입니다.

이진 트리를 재귀적으로 구현하는 일반적인 방식은 다음과 같습니다.

```c++
struct segtree {
    int lb, rb;                         // 이 노드가 담당하는 범위 
    int s = 0;                          // 구간 [lb, rb)의 합
    segtree *l = nullptr, *r = nullptr; // 자식 노드 포인터

    segtree(int lb, int rb) : lb(lb), rb(rb) {
        if (lb + 1 < rb) { // 리프 노드가 아니면 자식 생성
            int m = (lb + rb) / 2;
            l = new segtree(lb, m);
            r = new segtree(m, rb);
        }
    }

    void add(int k, int x) { /* a[k] += x 에 대응 */ }
    int sum(int k) { /* 처음 k개 요소의 합 계산 */ }
};
```

이 객체 지향적 구현은 소프트웨어 공학 측면에서는 괜찮을지 몰라도 성능 면에서는 끔찍합니다.

- 두 쿼리 구현 모두 [재귀(recursion)](/hpc/architecture/functions)를 사용합니다.
- 예측 불가능한 [분기(branching)](/hpc/pipelining/branching)를 사용하여 CPU 파이프라인을 멈추게 합니다.
- 노드가 불필요한 메타데이터를 저장합니다. 실제로는 정수 합을 담을 4바이트만 필요한데, 메모리 정렬 등의 이유로 노드당 32바이트를 사용하게 됩니다.
- 무엇보다도 [포인터 추적(pointer chasing)](/hpc/cpu-cache/latency/)이 많이 발생합니다. 쿼리만으로 필요한 구간을 미리 알 수 있음에도 불구하고 자식 노드로 내려가기 위해 포인터를 계속 가져와야 합니다.

포인터 추적의 문제는 다른 모든 이슈보다 훨씬 큽니다. 이를 해결하려면 포인터를 없애고 구조를 *암시적(implicit)*으로 만들어야 합니다.

### 암시적 세그먼트 트리 (Implicit Segment Trees)

세그먼트 트리는 이진 트리의 일종이므로 [에이칭어 레이아웃](../binary-search#eytzinger-layout)을 사용하여 노드들을 하나의 큰 배열에 저장하고 명시적인 포인터 대신 인덱스 산술 연산을 사용하여 탐색할 수 있습니다.

더 공식적으로, 루트 노드를 1번으로 정의하고 전체 배열 $[0, n)$의 합을 저장합니다. 그런 다음 범위 $[l, r]$에 대응하는 모든 노드 $v$에 대해 다음과 같이 정의합니다.

- 노드 $2v$는 범위 $[l, \lfloor \frac{l+r}{2} \rfloor)$에 대응하는 왼쪽 자식입니다.
- 노드 $(2v+1)$은 범위 $[\lfloor \frac{l+r}{2} \rfloor, r)$에 대응하는 오른쪽 자식입니다.

$n$이 2의 거듭제곱일 때 이 레이아웃은 전체 트리를 아주 깔끔하게 채웁니다.

![The memory layout of the implicit segment tree with the same query path highlighted](../img/segtree-layout.png)

하지만 $n$이 2의 거듭제곱이 아니면 레이아웃이 콤팩트하지 않게 됩니다. 여전히 $(2n-1)$개의 노드를 갖지만 이들이 $[1, 2n)$ 범위에 완벽하게 매핑되지 않기 때문입니다. 일단은 노드 저장을 위해 더 큰 배열(보통 $4n$)을 할당하여 이 문제를 해결할 수 있습니다.

```c++
int t[4 * N]; // 노드 합 저장
```

이제 `add`를 구현하기 위해, 포인터 대신 인덱스 산술 연산을 사용하는 재귀 함수를 만듭니다. 노드에 구간 경계를 저장하지 않으므로 매 재귀 호출마다 이를 계산하여 파라미터로 전달해야 합니다.

```c++
void add(int k, int x, int v = 1, int l = 0, int r = N) {
    t[v] += x;
    if (l + 1 < r) {
        int m = (l + r) / 2;
        if (k < m)
            add(k, x, 2 * v, l, m);
        else
            add(k, x, 2 * v + 1, m, r);
    }
}
```

구간 합 쿼리 구현도 비슷합니다.

```c++
int sum(int k, int v = 1, int l = 0, int r = N) {
    if (l >= k)
        return 0;
    if (r <= k)
        return t[v];
    int m = (l + r) / 2;
    return sum(k, 2 * v, l, m)
         + sum(k, 2 * v + 1, m, r);
}
```

재귀 함수에서 5개의 변수를 전달하는 것이 번거로워 보일 수 있지만 성능 이득은 확실합니다.

![](../img/segtree-topdown.svg)

메모리를 훨씬 적게 사용하여 CPU 캐시에 더 잘 들어맞는 것 외에도, 이 구현의 주요 장점은 [메모리 병렬성(mlp)](/hpc/cpu-cache/mlp)을 활용하여 필요한 노드들을 병렬로 가져올 수 있어 두 쿼리의 실행 시간을 상당히 개선할 수 있다는 점입니다.

성능을 더 높이려면 다음과 같이 할 수 있습니다.

- 인덱스 산술 연산 수동 최적화 (예: `v`에 2를 곱하는 공통 작업 확인)
- 나누기 2를 명시적인 비트 시프트로 대체 (컴파일러가 항상 수행하지는 않음)
- 재귀를 제거하고 완전히 반복문(iterative) 기반으로 구현

`add`는 꼬리 재귀(tail-recursive)이므로 단일 `while` 루프로 쉽게 바꿀 수 있습니다.

```c++
void add(int k, int x) {
    int v = 1, l = 0, r = N;
    while (l + 1 < r) {
        t[v] += x;
        v <<= 1;
        int m = (l + r) >> 1;
        if (k < m)
            r = m;
        else
            l = m, v++;
    }
    t[v] += x;
}
```

`sum` 쿼리는 재귀 호출이 두 번 있어서 약간 더 어렵지만, 쿼리 범위가 절반 중 하나에만 걸치게 되면 한쪽 호출은 즉시 종료된다는 점을 이용해 최적화할 수 있습니다.

```c++
int sum(int k) {
    int v = 1, l = 0, r = N, s = 0;
    while (true) {
        int m = (l + r) >> 1;
        v <<= 1;
        if (k >= m) {
            s += t[v++];
            if (k == m)
                break;
            l = m;
        } else {
            r = m;
        }
    }
    return s;
}
```

이 구현은 구간 합 쿼리의 실행 시간을 거의 절반으로 줄여줍니다.

![](../img/segtree-iterative.svg)

하지만 여전히 메모리를 필요 이상으로 사용하고 분기 문제가 있으며 반복마다 경계를 재계산해야 합니다. 이를 해결하기 위해 접근 방식을 바꿉니다.

### 바텀업 구현 (Bottom-Up Implementation)

암시적 세그먼트 트리 레이아웃의 정의를 바꿔보겠습니다. 부모-자식 관계에 의존하는 대신, 먼저 모든 리프 노드에 $[n, 2n)$ 범위의 번호를 강제로 할당하고, 부모 노드 $k$를 $\lfloor \frac{k}{2} \rfloor$로 정의합니다.

이 방식의 장점은 마지막 레이어가 연속적으로 $n$부터 시작하도록 강제되었으므로 절반 크기의 배열만 사용해도 된다는 것입니다.

```c++
int t[2 * N];
```

$n$이 2의 거듭제곱일 때 트리의 구조는 이전과 동일하며, $k$번째 리프 노드($N + k$ 인덱스)에서 시작하여 루트까지 올라가는 바텀업 방식으로 쿼리를 구현할 수 있습니다.

```c++
void add(int k, int x) {
    k += N;
    while (k != 0) {
        t[k] += x;
        k >>= 1;
    }
}
```

$[l, r)$ 구간 합을 계산하기 위해 첫 번째와 마지막 요소의 포인터를 유지하며 트리를 올라가는 방식을 사용합니다.

```c++
int sum(int l, int r) {
    l += N;
    r += N - 1;
    int s = 0;
    while (l <= r) {
        if ( l & 1) s += t[l++]; // l이 오른쪽 자식이면 더하고 사촌으로 이동
        if (~r & 1) s += t[r--]; // r이 왼쪽 자식이면 더하고 사촌으로 이동
        l >>= 1, r >>= 1;
    }
    return s;
}
```

놀랍게도 $n$이 2의 거듭제곱이 아닐 때도 이 방식은 올바르게 작동합니다. 리프 노드들이 약간 꼬인 형태가 되지만 산술 연산 결과는 여전히 유효하기 때문입니다.

탑다운 방식에 비해 메모리를 절반만 사용하고 쿼리 범위를 유지할 필요가 없어 코드가 단순해지고 빨라집니다.

![](../img/segtree-bottomup.svg)

구간 합 쿼리의 성능을 더 높이기 위해 분기 없는(branchless) 구현을 시도할 수 있습니다.

```c++
int sum(int k) {
    k = leaf(k - 1);
    int s = 0;
    while (k != 0) {
        s += (~k & 1) ? t[k] : 0; // "cmov"로 대체됨
        k = (k - 1) >> 1;
    }
    return s;
}
```

이러한 최적화들을 결합하면 쿼리가 훨씬 빨라집니다.

![](../img/segtree-branchless.svg)

하지만 여전히 메모리 대역폭과 캐시 문제를 해결해야 합니다.

### 페닉 트리 (Fenwick Trees)

세그먼트 트리를 정보 이론적으로 최소한의 공간만 사용하는 *간결한(succinct)* 구조로 만들 수 있습니다. 오른쪽 자식 노드에 저장된 합이 구간 합 계산에 필수적이지 않다는 점을 이용해 이들을 제거하면 노드 수를 원래 배열 크기와 동일하게 만들 수 있습니다.

이를 *페닉 트리(Fenwick tree)*라고 합니다. 1-기반 인덱싱과 `k & -k` 비트 트릭을 사용하여 담당 구간을 효율적으로 계산합니다.

```c++
int sum(int k) {
    int s = 0;
    for (; k != 0; k &= k - 1)
        s += t[k];
    return s;
}

void add(int k, int x) {
    for (k += 1; k <= N; k += k & -k)
        t[k] += x;
}
```

페닉 트리의 성능은 최적화된 바텀업 세그먼트 트리와 비슷하거나 더 빠릅니다.

![](../img/segtree-fenwick.svg)

레이아웃에 "구멍(holes)"을 뚫어 캐시 연관성 문제를 해결하면 지연 시간을 최대 3배까지 줄일 수 있습니다.

![](../img/segtree-fenwick-holes.svg)

### 와이드 세그먼트 트리 (Wide Segment Trees)

핵심 아이디어: 어차피 캐시 라인 전체를 가져온다면 그 라인을 쿼리 처리에 유용한 정보로 가득 채우자는 것입니다. 한 노드에 하나 이상의 데이터 포인트를 저장하여 트리의 높이를 낮추고 반복 횟수를 줄입니다.

이를 *와이드 세그먼트 트리(wide segment tree)*라고 합니다. 노드당 $B$개의 요소를 두면 높이를 $\log_2 B$배만큼 줄일 수 있습니다. 노드에 접두사 합을 저장하고 [SIMD](/hpc/simd)를 활용하여 `add` 쿼리를 가속화합니다.

이 방식은 구간 합 쿼리는 10배 이상, 업데이트 쿼리는 최대 4배 더 빠르게 만듭니다.

![](../img/segtree-simd.svg)

블록 크기를 키우면 업데이트 시간은 늘어나지만 쿼리 시간은 트리가 낮아짐에 따라 줄어듭니다.

![](../img/segtree-simd-others.svg)

### 비교 (Comparisons)

와이드 세그먼트 트리는 다른 대중적인 구현보다 압도적으로 빠릅니다.

![](../img/segtree-popular.svg)

상대적 속도 향상은 수십 배에서 수백 배에 달합니다.

![](../img/segtree-popular-relative.svg)

### 변형 (Modifications)

와이드 세그먼트 트리는 다른 데이터 타입, 가역적 연산(곱셈, XOR 등), 비가역적 연산(모노이드 - 구간 최소값 등), 그리고 지연 전파(lazy propagation)를 지원하도록 확장될 수 있습니다. 비가역적 연산이나 지연 전파 구현 시 속도는 약간 느려지겠지만 여전히 바텀업 세그먼트 트리보다는 빠를 것입니다.

### 감사의 말 (Acknowledgements)

Rossano Venturini와 함께 저술한 "[Practical Trade-Offs for the Prefix-Sum Problem](https://arxiv.org/pdf/2006.14552.pdf)"(2020) 논문을 기반으로 공동 작업해주신 Giulio Ermanno Pibiri에게 큰 감사를 표합니다.

바텀업 세그먼트 트리에 관한 코드와 아이디어는 Oleksandr Bacherikov의 2015년 블로그 포스트 "[Efficient and easy segment trees](https://codeforces.com/blog/entry/18051)"를 참고했습니다.
