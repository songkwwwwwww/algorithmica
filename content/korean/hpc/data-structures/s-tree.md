---
title: 정적 B-트리 (Static B-Trees)
weight: 2
---

이 섹션은 분기를 제거하고 메모리 레이아웃을 개선하여 이진 탐색을 최적화했던 [이전 섹션](../binary-search)의 후속입니다. 여기서도 정렬된 배열에서의 탐색을 다루지만, 이번에는 한 번에 하나의 요소만 가져오고 비교하는 것에 국한되지 않습니다.

이 섹션에서는 이진 탐색을 위해 개발한 기술을 *정적 B-트리(static B-trees)*로 일반화하고 [SIMD 인스트럭션](/hpc/simd)을 사용하여 더욱 가속화합니다. 특히 두 가지 새로운 암시적(implicit) 자료 구조를 개발합니다.

- [첫 번째](#b-tree-layout)는 B-트리의 메모리 레이아웃을 기반으로 하며, 배열 크기에 따라 `std::lower_bound`보다 최대 8배 빠릅니다. 배열과 동일한 공간을 사용하며 요소의 순열(permutation)만 필요로 합니다.
- [두 번째](#b-tree-layout-1)는 B+ 트리의 메모리 레이아웃을 기반으로 하며, `std::lower_bound`보다 최대 15배 빠릅니다. 단 6-7%의 추가 메모리만 사용하며, 원래의 정렬된 배열을 유지할 수 있다면 메모리의 6-7%**만** 사용하게 됩니다.

포인터가 있고 노드당 수백에서 수천 개의 키를 가지며 빈 공간이 있는 일반적인 B-트리와 구분하기 위해, 이러한 특정 메모리 레이아웃을 지칭하는 용어로 각각 *S-트리(S-tree)*와 *S+ 트리(S+ tree)*라는 이름을 사용하겠습니다[^name].

[^name]: [B-트리의 기원](https://en.wikipedia.org/wiki/B-tree#Origin)과 비슷하게, "S-트리의 S가 무엇을 의미하는지 더 많이 생각할수록 S-트리를 더 잘 이해하게 될 것입니다."

<!--

Similar to how the B in B-trees stands for many thing in We even have more claim to it than Bayer had on B-tree: it is succinct, static, simd, my name, my surname.

- *S-tree*: an approach based on the implicit (pointer-free) B-layout accelerated with SIMD operations to perform search efficiently while using less memory bandwidth and is ~8x faster on small arrays and 5x faster on large arrays.
- *S+ tree*: an approach similarly based on the B+ layout and achieves up to 15x faster for small arrays and ~7x faster on large arrays. Uses 6-7% of the array memory.

There is a an obscure data structure in computer vision.

The last two approaches use SIMD, which technically disqualifies it from being binary search. This is technically not a drop-in replacement, since it requires some preprocessing, but I can't recall a lot of scenarios where you obtain a sorted array but can't spend linear time on preprocessing.

-->

제가 아는 한, 이는 기존의 [접근 방식](http://kaldewey.com/pubs/FAST__SIGMOD10.pdf)에 비해 상당한 개선입니다. 이전과 마찬가지로 Zen 2 CPU를 대상으로 Clang 10을 사용하지만, 성능 향상은 Arm 기반 칩을 포함한 대부분의 다른 플랫폼에도 거의 그대로 적용될 것입니다. 자신의 머신에서 테스트해보고 싶다면 최종 구현의 [단일 소스 벤치마크](https://github.com/sslotin/amh-code/blob/main/binsearch/standalone.cc)를 사용하세요.

이 글은 내용이 길고 [교과서적인](/hpc/) 사례 연구 역할도 하므로, 교육적인 목적을 위해 알고리즘을 점진적으로 개선해 나갈 것입니다. 이미 전문가이고 문맥 없이도 [인트린직(intrinsic)](/hpc/simd/intrinsics)이 많이 포함된 코드를 읽는 데 무리가 없다면 바로 [최종 구현](#implicit-b-tree-1)으로 넘어가셔도 좋습니다.

## B-트리 레이아웃 (B-Tree Layout)

B-트리는 노드가 두 개 이상의 자식을 가질 수 있도록 하여 이진 탐색 트리의 개념을 일반화한 것입니다. 단일 키 대신, $k$차 B-트리의 노드는 정렬된 순서로 저장된 최대 $B = (k - 1)$개의 키와 최대 $k$개의 자식 노드 포인터를 포함할 수 있습니다. 각 자식 $i$는 해당 서브트리의 모든 키가 부모 노드의 키 $(i-1)$과 $i$ 사이에 있다는 속성을 만족합니다(존재하는 경우).

![A B-tree of order 4](../img/b-tree.jpg)

이 접근 방식의 주요 장점은 트리의 높이를 $\frac{\log_2 n}{\log_k n} = \frac{\log k}{\log 2} = \log_2 k$배만큼 줄인다는 것입니다. 그러면서도 각 노드를 가져오는 시간은 단일 [메모리 블록](/hpc/external-memory/hierarchy/)에 들어가는 한 거의 동일하게 유지됩니다.

B-트리는 원래 무작위로 1바이트를 가져오는 지연 시간이 다음 1MB의 데이터를 순차적으로 읽는 시간과 맞먹는 디스크 기반 데이터베이스 관리를 위해 개발되었습니다. 우리의 사례에서는 $B = 16$개 요소(즉, 캐시 라인 크기인 64바이트)의 블록 크기를 사용할 것입니다. 이는 이진 탐색에 비해 트리 높이와 쿼리당 총 캐시 라인 페치 횟수를 약 $\log_2 17 \approx 4$배 줄여줍니다.

### 암시적 B-트리 (Implicit B-Tree)

B-트리 노드에 포인터를 저장하고 가져오는 것은 귀중한 캐시 공간을 낭비하고 성능을 저하시키지만, 삽입과 삭제 시 트리 구조를 변경하는 데는 필수적입니다. 하지만 업데이트가 없고 트리 구조가 *정적(static)*인 경우 포인터를 제거하여 구조를 *암시적(implicit)*으로 만들 수 있습니다.

이를 달성하는 한 가지 방법은 [에이칭어 번호 매기기(Eytzinger numeration)](../binary-search#eytzinger-layout)를 $(B+1)$진 트리로 일반화하는 것입니다.

- 루트 노드의 번호는 0입니다.
- 노드 $k$는 $i \in [0, B]$에 대해 $\\{k \cdot (B + 1) + i + 1\\}$로 번호가 매겨진 $(B+1)$개의 자식 노드를 가집니다.

이렇게 하면 하나의 큰 2차원 키 배열을 할당하고 인덱스 산술 연산을 통해 트리에서 자식 노드를 찾음으로써 $O(1)$의 추가 메모리만 사용할 수 있습니다.

```c++
const int B = 16;

int nblocks = (n + B - 1) / B;
int btree[nblocks][B];

int go(int k, int i) { return k * (B + 1) + i + 1; }
```

<!-- todo: exact height -->

이 번호 매기기는 자동으로 B-트리를 $\Theta(\log_{B + 1} n)$ 높이의 완전 트리 또는 거의 완전 트리로 만듭니다. 초기 배열의 길이가 $B$의 배수가 아니면 마지막 블록은 해당 데이터 타입의 최대값으로 패딩됩니다.

### 구조 (Construction)

에이칭어 배열을 만들 때와 비슷하게 탐색 트리를 순회하며 B-트리를 구축할 수 있습니다.

```c++
void build(int k = 0) {
    static int t = 0;
    if (k < nblocks) {
        for (int i = 0; i < B; i++) {
            build(go(k, i));
            btree[k][i] = (t < n ? a[t++] : INT_MAX);
        }
        build(go(k, B));
    }
}
```

초기 배열의 각 값이 결과 배열의 고유한 위치로 복사되고, 자식 노드로 내려갈 때마다 $k$에 $(B + 1)$이 곱해지므로 트리 높이는 $\Theta(\log_{B+1} n)$이 되어 정확합니다.

이 번호 매기기는 약간의 불균형을 초래할 수 있습니다. 왼쪽 자식들이 더 큰 서브트리를 가질 수 있지만, 이는 오직 $O(\log_{B+1} n)$개의 부모 노드에 대해서만 해당됩니다.

### 탐색 (Searches)

하한(lower bound)을 찾으려면 노드에서 $B$개의 키를 가져와 $x$보다 작지 않은 첫 번째 키 $a_i$를 찾고, $i$번째 자식으로 내려가는 과정을 리프 노드에 도달할 때까지 반복해야 합니다. 이 첫 번째 키를 찾는 방법은 다양합니다. 예를 들어 $O(\log B)$번 반복하는 작은 내부 이진 탐색을 하거나, 운 좋게 루프를 빨리 빠져나오기를 바라며 $O(B)$ 시간에 각 키를 순차적으로 비교할 수 있습니다.

하지만 우리는 그렇게 하지 않을 것입니다. [SIMD](/hpc/simd)를 사용할 수 있기 때문입니다. SIMD는 분기(branching)와 잘 맞지 않으므로, 기본적으로 모든 $B$개 요소를 무조건 비교하고 그 결과로부터 비트마스크를 생성한 뒤, `ffs` 인스트럭션을 사용하여 작지 않은 첫 번째 요소에 해당하는 비트를 찾는 방식을 사용합니다.

```cpp
int mask = (1 << B);

for (int i = 0; i < B; i++)
    mask |= (btree[k][i] >= x) << i;

int i = __builtin_ffs(mask) - 1;
// 이제 i는 올바른 자식 노드의 번호입니다.
```

불행히도 컴파일러는 아직 이 코드를 [자동 벡터화(auto-vectorize)](/hpc/simd/auto-vectorization/)할 만큼 똑똑하지 않으므로 수동으로 최적화해야 합니다. AVX2에서는 8개 요소를 로드하여 검색 키와 비교해 [벡터 마스크(vector mask)](/hpc/simd/masking/)를 생성한 다음, `movemask`로 스칼라 마스크를 추출할 수 있습니다. 다음은 우리가 하려는 작업의 요약된 그림입니다.

```center
       y = 4        17       65       103     
       x = 42       42       42       42      
   y ≥ x = 00000000 00000000 11111111 11111111
           ├┬┬┬─────┴────────┴────────┘       
movemask = 0011                               
           ┌─┘                                
     ffs = 3                                  
```

한 번에 8개 요소(블록/캐시 라인 크기의 절반)만 처리할 수 있으므로 요소를 두 그룹으로 나누고 두 개의 8비트 마스크를 결합해야 합니다. 이를 위해 `x > y` 조건을 사용하고 반전된 마스크를 계산하는 것이 약간 더 쉽습니다.

```c++
typedef __m256i reg;

int cmp(reg x_vec, int* y_ptr) {
    reg y_vec = _mm256_load_si256((reg*) y_ptr); // 8개의 정렬된 요소 로드
    reg mask = _mm256_cmpgt_epi32(x_vec, y_vec); // 키와 비교
    return _mm256_movemask_ps((__m256) mask);    // 8비트 마스크 추출
}
```

이제 전체 블록을 처리하기 위해 이를 두 번 호출하고 마스크를 결합합니다.

```c++
int mask = ~(
    cmp(x, &btree[k][0]) +
    (cmp(x, &btree[k][8]) << 8)
);
```

트리를 내려가기 위해 해당 마스크에 `ffs`를 사용하여 올바른 자식 번호를 얻고 앞서 정의한 `go` 함수를 호출합니다.

```c++
int i = __builtin_ffs(mask) - 1;
k = go(k, i);
```

실제로 마지막에 결과를 반환할 때, 마지막으로 방문한 노드에서 `btree[k][i]`를 가져오고 싶겠지만, 가끔 로컬 하한이 존재하지 않는 경우($i \ge B$)가 발생합니다. $x$가 노드의 모든 키보다 크기 때문입니다. 이론적으로는 [에이칭어 이진 탐색](../binary-search/#search-implementation)에서 했던 것과 같은 방식을 사용하여 마지막 인덱스를 계산한 후 올바른 요소를 복구할 수도 있습니다. 하지만 이번에는 깔끔한 비트 트릭이 없어서 [17로 나누는 연산](/hpc/arithmetic/division)을 많이 해야 하므로 속도가 느려지고 그럴 가치가 거의 없을 것입니다.

대신 트리를 내려갈 때 만난 마지막 로컬 하한을 기억했다가 반환할 수 있습니다.

```c++
int lower_bound(int _x) {
    int k = 0, res = INT_MAX;
    reg x = _mm256_set1_epi32(_x);
    while (k < nblocks) {
        int mask = ~(
            cmp(x, &btree[k][0]) +
            (cmp(x, &btree[k][8]) << 8)
        );
        int i = __builtin_ffs(mask) - 1;
        if (i < B)
            res = btree[k][i];
        k = go(k, i);
    }
    return res;
}
```

이 구현은 이전의 모든 이진 탐색 구현보다 훨씬 뛰어난 성능을 보여줍니다.

![](../img/search-btree.svg)

매우 훌륭하지만, 여기서 더 최적화할 수 있습니다.

### 최적화 (Optimization)

다른 무엇보다 먼저 배열을 위한 메모리를 [휴즈 페이지(hugepage)](/hpc/cpu-cache/paging)에 할당해 보겠습니다.

```c++
const int P = 1 << 21;                        // 페이지 크기 (바이트, 2MB)
const int T = (64 * nblocks + P - 1) / P * P; // 페이지 단위로만 할당 가능
btree = (int(*)[16]) std::aligned_alloc(P, T);
madvise(btree, T, MADV_HUGEPAGE);
```

이는 큰 배열 크기에서 성능을 약간 향상시킵니다.

![](../img/search-btree-hugepages.svg)

공정한 비교를 위해 이전의 모든 [구현](../binary-search)에도 휴즈 페이지를 활성화해야 하겠지만, 그들은 모두 이 문제를 완화하는 어떤 형태의 프리페칭을 가지고 있으므로 그리 중요하지는 않습니다.

이제 본격적인 최적화를 시작해 보겠습니다. 먼저, 가능한 한 변수 대신 컴파일 타임 상수를 사용하고 싶습니다. 컴파일러가 이를 머신 코드에 임베드하고, 루프를 풀고, 산술 연산을 최적화하는 등 여러 가지 이점을 무료로 제공해주기 때문입니다. 구체적으로 트리 높이를 미리 알고 싶습니다.

```c++
constexpr int height(int n) {
    // 트리의 크기가 n을 초과할 때까지 키움
    int s = 0, // 현재까지의 총 크기
        l = B, // 다음 레이어의 크기
        h = 0; // 현재까지의 높이
    while (s + l - B < n) {
        s += l;
        l *= (B + 1);
        h++;
    }
    return h;
}

const int H = height(N);
```

<!--

```c++
constexpr std::pair<int, int> precalc(int n) {
    int s = 0, // total size
        l = B, // size of next layer
        h = 0; // height so far
    while (s + l - B < n) {
        s += l;
        l *= (B + 1);
        h++;
    }
    int r = (n - s + B - 1) / B; // remaining blocks on the last layer
    return {h, s / B + (r + B) / (B + 1) * (B + 1)};
}

const int [height, nblocks] = precalc(N);
```

-->

다음으로 노드에서 로컬 하한을 더 빨리 찾을 수 있습니다. 두 개의 8요소 블록에 대해 별도로 계산하고 마스크를 병합하는 대신, [packs](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html) 인스트럭션을 사용하여 벡터 마스크를 결합하고 `movemask`를 한 번만 사용하여 바로 추출할 수 있습니다.

```c++
unsigned rank(reg x, int* y) {
    reg a = _mm256_load_si256((reg*) y);
    reg b = _mm256_load_si256((reg*) (y + 8));

    reg ca = _mm256_cmpgt_epi32(a, x);
    reg cb = _mm256_cmpgt_epi32(b, x);

    reg c = _mm256_packs_epi32(ca, cb);
    int mask = _mm256_movemask_epi8(c);

    // movemask_epi8을 16비트 마스크에 대해 호출했으므로 결과를 2로 나누어야 합니다:
    return __tzcnt_u32(mask) >> 1;
}
```

이 인스트럭션은 두 레지스터에 저장된 32비트 정수를 하나의 레지스터에 저장된 16비트 정수로 변환하며, 우리의 경우 벡터 마스크를 하나로 합치는 효과가 있습니다. 비교 순서를 바꿨는데, 이렇게 하면 마지막에 마스크를 반전시킬 필요가 없지만 하한이 올바르게 작동하도록 처음에 검색 키에서 1을 빼야 합니다[^float](그렇지 않으면 `upper_bound`처럼 작동합니다).

[^float]: [부동 소수점(floating-point)](/hpc/arithmetic/float) 키를 다루어야 한다면 `upper_bound`로 충분한지 고려해 보십시오. 만약 `lower_bound`가 꼭 필요하다면 검색 키에서 1이나 머신 엡실론을 빼는 방식은 작동하지 않습니다. 대신 [표현 가능한 이전 숫자](https://stackoverflow.com/questions/10160079/how-to-find-nearest-next-previous-double-value-numeric-limitsepsilon-for-give)를 가져와야 합니다. 몇몇 특수한 경우를 제외하면, 이는 기본적으로 비트를 정수로 재해석하고 1을 뺀 다음 다시 실수로 재해석하는 것과 같습니다 ([IEEE-754 부동 소수점](/hpc/arithmetic/ieee-754)가 저장되는 방식 덕분에 마법처럼 작동합니다).

문제는 이 방식이 우리가 원하는 `a1 a2 b1 b2` 순서가 아니라 `a1 b1 a2 b2`와 같은 이상한 인터리빙 방식으로 결과를 쓴다는 점입니다. 많은 AVX2 인스트럭션이 이런 경향이 있습니다. 이를 바로잡으려면 결과 벡터를 [치환(permute)](/hpc/simd/shuffling)해야 하지만, 쿼리 시점에 하는 대신 전처리 과정에서 모든 노드를 치환해둘 수 있습니다.

```c++
void permute(int *node) {
    const reg perm = _mm256_setr_epi32(4, 5, 6, 7, 0, 1, 2, 3);
    reg* middle = (reg*) (node + 4);
    reg x = _mm256_loadu_si256(middle);
    x = _mm256_permutevar8x32_epi32(x, perm);
    _mm256_storeu_si256(middle, x);
}
```

이제 노드 구축을 마친 직후에 `permute(&btree[k])`를 호출하면 됩니다. 전처리 시간은 지금 그리 중요하지 않으므로 중간 요소를 교체하는 더 빠른 방법이 있겠지만 여기까지만 하겠습니다.

이 새로운 SIMD 루틴은 훨씬 더 빠릅니다. 추가적인 `movemask`가 느리고 두 마스크를 블렌딩하는 데 꽤 많은 인스트럭션이 들기 때문입니다. 불행히도 이제 요소들이 치환되었기 때문에 `res = btree[k][i]` 업데이트를 단순히 할 수 없습니다. `i`에 관한 비트 수준의 트릭으로 해결할 수도 있지만, 작은 조회 테이블(lookup table)을 인덱싱하는 것이 더 빠르고 새로운 분기도 필요하지 않습니다.

```c++
const int translate[17] = {
    0, 1, 2, 3,
    8, 9, 10, 11,
    4, 5, 6, 7,
    12, 13, 14, 15,
    0
};

void update(int &res, int* node, unsigned i) {
    int val = node[translate[i]];
    res = (i < B ? val : res);
}
```

이 `update` 절차는 시간이 좀 걸리지만 반복 사이의 크리티컬 패스(critical path)에 있지 않으므로 실제 성능에 큰 영향을 미치지는 않습니다.

모두 결합하면 (몇몇 사소한 최적화는 생략):

```c++
int lower_bound(int _x) {
    int k = 0, res = INT_MAX;
    reg x = _mm256_set1_epi32(_x - 1);
    for (int h = 0; h < H - 1; h++) {
        unsigned i = rank(x, &btree[k]);
        update(res, &btree[k], i);
        k = go(k, i);
    }
    // 마지막 분기:
    if (k < nblocks) {
        unsigned i = rank(x, btree[k]);
        update(res, &btree[k], i);
    }
    return res;
}
```

이 모든 작업으로 15~20% 정도의 성능 향상을 얻었습니다.

![](../img/search-btree-optimized.svg)

현재 구현에는 두 가지 주요 문제가 있습니다.

- `update` 절차가 상당히 비용이 많이 듭니다. 특히 17번 중 16번은 마지막 블록에서 결과를 가져오면 되기 때문에 대개의 경우 쓸모가 없다는 점을 고려하면 더욱 그렇습니다.
- 반복 횟수가 일정하지 않아 [에이칭어 이진 탐색](../binary-search/#removing-the-last-branch)에서와 비슷한 분기 예측 문제를 일으킵니다. 그래프를 보면 $2^4$ 주기로 지연 시간이 튀는 것을 볼 수 있습니다.

이 문제들을 해결하기 위해 레이아웃을 약간 변경해야 합니다.

## B+ 트리 레이아웃 (B+ Tree Layout)

사람들이 B-트리에 대해 이야기할 때 대개는 *B+ 트리*를 의미합니다. 이는 두 가지 유형의 노드를 구분하는 변형입니다.

- *내부 노드(Internal nodes)*는 최대 $B$개의 키와 $(B + 1)$개의 자식 노드 포인터를 저장합니다. 키 $i$는 항상 $(i + 1)$번째 자식 노드 서브트리의 최소 키와 같습니다.
- *데이터 노드(Data nodes)* 또는 *리프(leaves)*는 최대 $B$개의 키, 다음 리프 노드에 대한 포인터, 그리고 선택적으로 각 키와 연관된 값을 저장합니다.

이 접근 방식의 장점은 탐색 시간이 더 빠르고(내부 노드가 키만 저장하기 때문) 범위 쿼리를 빠르게 수행할 수 있다는 점입니다. 하지만 내부 노드에 키의 복사본을 저장해야 하므로 약간의 메모리 오버헤드가 발생합니다.

![A B+ tree of order 4](../img/bplus.png)

우리의 사례에서 이 레이아웃은 두 가지 문제를 해결하는 데 도움이 됩니다.

- 우리가 내려가는 마지막 노드가 로컬 하한을 가지고 있거나 다음 리프 노드의 첫 번째 키가 하한이 되므로, 매 반복마다 `update`를 호출할 필요가 없습니다.
- B+ 트리는 리프가 아닌 루트에서 자라기 때문에 모든 리프의 깊이가 일정하여 분기할 필요가 없습니다.

단점은 이 레이아웃이 *간결(succinct)*하지 않다는 것입니다. 내부 노드를 저장하기 위해 원래 배열 크기의 약 1/16 정도의 추가 메모리가 필요합니다. 하지만 성능 향상은 그럴만한 가치가 충분할 것입니다.

### 암시적 B+ 트리 (Implicit B+ Tree)

포인터 산술을 더 명확하게 하기 위해 전체 트리를 하나의 1차원 배열에 저장할 것입니다. 실행 시 인덱스 계산을 최소화하기 위해 각 레이어를 이 배열에 순차적으로 저장하고 컴파일 타임에 계산된 오프셋을 사용하여 레이어에 접근합니다. 레이어 $h$의 노드 $k$에 있는 키들은 `btree[offset(h) + k * B]`에서 시작하고, $i$번째 자식은 `btree[offset(h - 1) + (k * (B + 1) + i) * B]`에 있게 됩니다.

이를 구현하기 위해 몇 가지 `constexpr` 함수가 더 필요합니다.

```c++
// n개의 키가 있는 레이어의 B-요소 블록 수
constexpr int blocks(int n) {
    return (n + B - 1) / B;
}

// n개의 키가 있는 레이어의 이전 레이어 키 수
constexpr int prev_keys(int n) {
    return (blocks(n) + B) / (B + 1) * B;
}

// 균형 잡힌 n-키 B+ 트리의 높이
constexpr int height(int n) {
    return (n <= B ? 1 : height(prev_keys(n)) + 1);
}

// 레이어 h가 시작되는 위치 (0번 레이어가 가장 큼)
constexpr int offset(int h) {
    int k = 0, n = N;
    while (h--) {
        k += blocks(n) * B;
        n = prev_keys(n);
    }
    return k;
}

const int H = height(N);
const int S = offset(H); // 트리 크기는 레이어 H의 오프셋

int *btree; // 트리는 하나의 Hugepage 정렬된 크기 S 배열에 저장됨
```

레이어를 역순으로 저장하고 아래에서 위로 번호를 매깁니다. 리프가 0번 레이어이고 루트가 $H - 1$번 레이어입니다.

### 구조 (Construction)

정렬된 배열 `a`로부터 트리를 구축하려면 먼저 이를 0번 레이어에 복사하고 무한대 값으로 패딩해야 합니다.

```c++
memcpy(btree, a, 4 * N);

for (int i = N; i < S; i++)
    btree[i] = INT_MAX;
```

이제 내부 노드들을 레이어별로 구축합니다. 각 키에 대해 해당 키의 오른쪽으로 내려간 다음 리프에 도달할 때까지 항상 왼쪽으로 이동하여 첫 번째 키를 가져옵니다. 이것이 서브트리에서 가장 작은 키가 됩니다.

```c++
for (int h = 1; h < H; h++) {
    for (int i = 0; i < offset(h + 1) - offset(h); i++) {
        // i = k * B + j
        int k = i / B,
            j = i - k * B;
        k = k * (B + 1) + j + 1; // 키의 오른쪽과 비교
        // 그 후 항상 왼쪽으로 이동
        for (int l = 0; l < h - 1; l++)
            k *= (B + 1);
        // 키가 존재하지 않으면 무한대 값으로 패딩
        btree[offset(h) + i] = (k * B < N ? btree[k * B] : INT_MAX);
    }
}
```

마지막 마무리로 내부 노드들의 키를 치환(permute)합니다.

```c++
for (int i = offset(1); i < S; i += B)
    permute(btree + i);
```

우리는 1번 레이어부터 시작하며 리프 노드는 치환하지 않습니다. 리프 노드를 치환하면 `update`에서의 복잡한 인덱스 변환이 필요한데, 이것이 마지막 작업일 때는 성능에 큰 영향을 미치기 때문입니다.

### 탐색 (Searching)

탐색 절차는 B-트리 레이아웃보다 간단해집니다. `update`가 필요 없고 고정된 횟수만큼 반복하면 됩니다(마지막 반복만 별도로 처리).

```c++
int lower_bound(int _x) {
    unsigned k = 0; // 포인터 산술 최적화를 위해 k에 이미 B가 곱해졌다고 가정
    reg x = _mm256_set1_epi32(_x - 1);
    for (int h = H - 1; h > 0; h--) {
        unsigned i = permuted_rank(x, btree + offset(h) + k);
        k = k * (B + 1) + i * B;
    }
    unsigned i = direct_rank(x, btree + k);
    return btree[k + i];
}
```

B+ 레이아웃으로 전환한 보람이 있습니다. S+ 트리는 최적화된 S-트리보다 1.5~3배 빠릅니다.

![](../img/search-bplus.svg)

그래프 끝부분의 스파이크는 L1 TLB 용량 부족으로 인해 발생합니다. S+ 트리는 약 7%의 메모리 오버헤드 때문에 이 한계에 조금 더 빨리 도달합니다.

### `std::lower_bound`와의 비교

이진 탐색으로부터 먼 길을 왔습니다.

![](../img/search-all.svg)

상대적 속도 향상을 보면 차이가 명확합니다.

![](../img/search-relative.svg)

우리가 측정하고 있는 것은 실제 지연 시간이 아니라 *역처리량(reciprocal throughput)*입니다. 즉, 많은 쿼리를 실행하는 데 걸린 총 시간을 쿼리 수로 나눈 값입니다. 실제 지연 시간을 측정하면(이전 결과에 의존성이 있는 쿼리 실행) 속도 향상이 그만큼 인상적이지는 않습니다.

![](../img/search-relative-latency.svg)

S+ 트리의 성능 향상 중 많은 부분은 분기를 제거하고 메모리 요청을 최소화하여 인접한 쿼리들의 실행을 중첩시킬 수 있다는 점에서 나오기 때문입니다.

### 수정 및 추가 최적화 (Modifications and Further Optimizations)

쿼리 중 메모리 접근 횟수를 최소화하기 위해 블록 크기를 늘릴 수 있습니다. 32개 요소 노드(두 개의 캐시 라인과 네 개의 AVX2 레지스터에 걸쳐 있음)에서 로컬 하한을 찾기 위해 [비슷한 트릭](https://github.com/sslotin/amh-code/blob/a74495a2c19dddc697f94221629c38fee09fa5ee/binsearch/bplus32.cc#L94)을 사용할 수 있습니다.

또한 캐시 계층 구조에서 각 트리 레이어가 저장되는 위치를 제어하여 캐시를 더 효율적으로 사용할 수 있습니다.

제가 구현한 두 가지 버전의 최적화(블록 크기 32 및 비동기 읽기)는 처리량을 개선하지는 못했습니다.

![](../img/search-bplus-other.svg)

…하지만 지연 시간은 더 낮아졌습니다.

![](../img/search-latency-bplus.svg)

아직 구현하지 못했지만 유망하다고 생각하는 아이디어들은 다음과 같습니다.

- 블록 크기를 가변적으로 만듭니다. 하나의 32개 요소 레이어를 두는 것이 두 개의 별도 레이어를 두는 것보다 속도 저하가 적기 때문입니다.
- 노드를 자손 노드들과 함께 그룹화하여 메모리상에서 가깝게 배치합니다(hierarchical blocking).
- 특정 레이어에 선택적으로 프리페칭을 사용합니다.

그 외의 사소한 최적화로는 마지막 레이어의 노드도 치환하기, 레이어 저장 순서 바꾸기, 어셈블리로 재작성하기, `packs` 대신 `blending` 사용하기, `tzcnt` 대신 `popcount` 사용하기 등이 있습니다.

현재 구현은 AVX2에 특화되어 있으며 다른 플랫폼에 적용하려면 상당한 변경이 필요할 수 있습니다.

### 동적 트리로서 (As a Dynamic Tree)

`std::set`과 같은 포인터 기반 트리와의 비교는 더욱 유리합니다. S+ 트리는 최대 30배까지 더 빠릅니다.

![](../img/search-set-relative.svg)

이 가설을 검증하기 위해 각 노드에 자식의 위치를 가리키는 17개의 인덱스 배열을 추가하고 이를 따라가도록 해보았습니다. 널리 사용되는 B-트리 구현인 Abseil의 B-트리보다 S+ 트리가 훨씬 빠른 성능을 보였습니다.

![](../img/search-set-relative-all.svg)

물론 동적 탐색 트리를 구현하는 것은 더 복잡한 문제입니다. 업데이트 작업도 구현해야 하며 그 과정에서 분기 계수를 희생해야 할 수도 있습니다. 하지만 `std::set`보다 10~20배, `absl::btree_set`보다 3~5배 빠른 구현이 가능해 보이며, 이는 우리가 [다음에 시도해 볼 일](../b-tree) 중 하나입니다.

### 감사의 말 (Acknowledgements)

Cory Nelson의 [StackOverflow 답변](https://stackoverflow.com/questions/20616605/using-simd-avx-sse-for-tree-traversal)에서 16개 요소 치환 탐색 트릭을 가져왔습니다.
