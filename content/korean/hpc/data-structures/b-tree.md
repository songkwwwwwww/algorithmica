---
title: 탐색 트리 (Search Trees)
weight: 3
---

[이전 글](../s-tree)에서는 정렬된 배열에서 이진 탐색 속도를 높이기 위해 *정적* B-트리를 설계하고 구현했습니다. 그 글의 [마지막 섹션](../s-tree/#as-a-dynamic-tree)에서는 [SIMD](/hpc/simd)를 통한 성능 이점을 유지하면서 이를 다시 *동적*으로 만드는 방법에 대해 간략히 논의했으며, S+ 트리의 내부 노드에 명시적인 포인터를 추가하고 따라가는 방식으로 예측을 검증했습니다.

이 글에서는 해당 제안을 이어가며 정수 키를 위한 최소한의 기능을 갖춘 탐색 트리를 설계합니다. 이를 통해 `lower_bound` 쿼리에서는 `std::set` 대비 최대 18배, [`absl::btree`](https://abseil.io/blog/20190812-btree) 대비 최대 7배의 속도 향상을 [달성](#evaluation)했으며, `insert` 쿼리에서는 각각 최대 8배와 2배의 속도 향상을 얻었습니다. 또한 개선의 여지는 여전히 충분합니다.

이 구조의 메모리 오버헤드는 32비트 정수의 경우 약 30%이며, 최종 구현은 [150줄 미만의 C++ 코드](https://github.com/sslotin/amh-code/blob/main/b-tree/btree-final.cc)로 이루어져 있습니다. 이 구조는 다른 산술 타입이나 해시, 국가 코드, 주식 종목 코드와 같은 작고 고정된 길이의 문자열로 쉽게 일반화할 수 있습니다.

<!--

7-18x/3-8x speedup over `std::set` and 3-7x/1.5-2x

that we call *B− tree*

-->

## B− 트리 (B− Tree)

다른 사례 연구에서처럼 작은 점진적 개선을 하는 대신, 이 글에서는 *B− 트리*라고 이름 붙인 하나의 자료 구조를 구현할 것입니다. 이는 [B+ 트리](../s-tree/#b-tree-layout-1)를 기반으로 하지만 몇 가지 사소한 차이점이 있습니다.

- B− 트리의 노드는 내부 노드의 자식 포인터를 제외하고는 포인터나 메타데이터를 저장하지 않습니다 (B+ 트리의 리프 노드는 다음 리프 노드에 대한 포인터를 저장합니다). 이를 통해 리프 노드의 키를 캐시 라인에 완벽하게 배치할 수 있습니다.
- 키 $i$를 자식 $(i + 1)$의 *최소* 키 대신 자식 $i$의 서브트리에 있는 *최대* 키로 정의합니다. 이렇게 하면 리프 노드에 도달한 후 다른 노드를 가져올 필요가 없습니다 (B+ 트리에서는 리프 노드의 모든 키가 검색 키보다 작을 수 있어 첫 번째 요소를 가져오기 위해 다음 리프 노드로 이동해야 할 수도 있습니다).

또한 일반적인 크기보다 작은 $B=32$의 노드 크기를 사용합니다. [S+ 트리에 최적이었던](../s-tree/#modifications-and-further-optimizations) 16이 아닌 이유는 포인터를 가져오는 것과 관련된 추가 오버헤드가 있기 때문입니다. 트리 높이를 약 20% 줄이는 이점이 노드당 두 배의 요소를 처리하는 비용보다 크며, 평균적으로 $\frac{B}{2}$번의 삽입마다 비용이 많이 드는 노드 분할을 수행해야 하는 `insert` 쿼리의 실행 시간을 개선하기 때문이기도 합니다.

<!--

We will discuss other node sizes later.

This is needed simd to be efficient (we will discuss other node sizes later).

There is some overhead, so it makes sense to use more than one cache line.

Analogous to the B+ tree,

-->

### 메모리 레이아웃 (Memory Layout)

소프트웨어 공학 측면에서 최선의 접근 방식은 아닐 수 있지만, 리프와 내부 노드를 구분하지 않고 미리 할당된 큰 배열에 전체 트리를 저장할 것입니다.

```c++
const int R = 1e8;
alignas(64) int tree[R];
```

또한 구현을 단순화하기 위해 이 배열을 무한대 값으로 미리 채웁니다.

```c++
for (int i = 0; i < R; i++)
    tree[i] = INT_MAX;
```

(일반적으로 내부적으로 `new`를 사용하는 `std::set`이나 다른 구조와 비교하는 것은 기술적으로 공정하지 않을 수 있지만, 메모리 할당과 초기화가 여기서 병목 현상은 아니므로 평가에 큰 영향을 미치지는 않습니다.)

두 노드 유형 모두 키를 정렬된 순서대로 순차적으로 저장하며, 배열에서 첫 번째 키의 인덱스로 식별됩니다.

- 리프 노드는 최대 $(B - 1)$개의 키를 가지지만 무한대 값을 채워 $B$개의 요소로 패딩됩니다.
- 내부 노드는 최대 $(B - 2)$개의 키를 가지며 $B$개로 패딩되고, 최대 $(B - 1)$개의 자식 노드 인덱스를 가지며 역시 $B$개로 패딩됩니다.

이러한 설계 결정은 임의적인 것이 아닙니다.

- 패딩을 통해 리프 노드는 정확히 2개의 캐시 라인을 차지하고, 내부 노드는 정확히 4개의 캐시 라인을 차지합니다.
- 캐시 공간을 절약하고 SIMD로 더 빠르게 이동하기 위해 [포인터 대신 인덱스](/hpc/cpu-cache/pointers/)를 사용합니다. (앞으로는 "포인터"와 "인덱스"를 혼용해서 사용하겠습니다.)
- 키와 인덱스가 서로 다른 캐시 라인에 저장되더라도 키 바로 뒤에 인덱스를 저장합니다. [그럴 만한 이유](/hpc/cpu-cache/aos-soa/)가 있기 때문입니다.
- 노드 분할 시 임시 결과를 저장해야 하므로 리프 노드에서 1개, 내부 노드에서 $2+1=3$개의 배열 셀을 의도적으로 "낭비"합니다.

처음에는 루트로 비어 있는 리프 노드 하나만 가집니다.

```c++
const int B = 32;

int root = 0;   // 루트의 키가 시작되는 위치
int n_tree = B; // 할당된 배열 셀의 수
int H = 1;      // 현재 트리 높이
```

새 노드를 "할당"하려면 리프 노드인 경우 `n_tree`를 $B$만큼, 내부 노드인 경우 $2B$만큼 늘리면 됩니다.

새 노드는 가득 찬 노드를 분할할 때만 생성되므로 루트를 제외한 각 노드는 최소한 절반 이상 채워집니다. 이는 정수 요소당 4~8바이트가 필요함을 의미합니다 (내부 노드는 그 수치에 $\frac{1}{16}$ 정도 기여합니다). 삽입이 순차적인 경우 4바이트에 가깝고, 입력이 최악의 조건(adversarial)인 경우 8바이트에 가깝습니다. 쿼리가 균등하게 분포될 때 노드는 평균적으로 약 75% 채워지며, 요소당 약 5.2바이트로 예상됩니다.

B-트리는 포인터 기반 이진 트리에 비해 메모리 효율이 매우 높습니다. 예를 들어 `std::set`은 최소 3개의 포인터(왼쪽 자식, 오른쪽 자식, 부모)가 필요하며, 이것만으로도 $3 \times 8 = 24$바이트가 들고, [구조체 패딩](/hpc/cpu-cache/alignment/)으로 인해 키와 메타 정보를 저장하는 데 최소 8바이트가 더 듭니다.

### 탐색 (Searching)

작업의 90% 이상이 조회(lookup)인 경우가 매우 흔하며, 그렇지 않더라도 다른 모든 트리 작업은 일반적으로 키를 찾는 것부터 시작합니다. 따라서 먼저 탐색을 구현하고 최적화해 보겠습니다.

[S-트리](../s-tree/#optimization)를 구현할 때는 블렌딩/팩(blending/packs) 인스트럭션이 작동하는 방식의 복잡함 때문에 키를 치환된(permuted) 순서로 저장했습니다. *동적 트리* 문제의 경우 키를 치환된 순서로 저장하면 삽입 구현이 훨씬 어려워지므로 접근 방식을 바꾸겠습니다.

정렬된 배열에서 요소 `x`가 들어갈 위치를 찾는 또 다른 방법은 "`x`보다 작지 않은 첫 번째 요소의 인덱스"가 아니라 "`x`보다 작은 요소의 개수"로 생각하는 것입니다. 이 관찰을 통해 다음과 같은 아이디어를 얻을 수 있습니다. 키를 `x`와 비교하고, 벡터 마스크를 32비트 마스크로 합친 뒤(각 비트는 매핑이 일대일 대응이기만 하면 어떤 요소와도 대응될 수 있음), 그 마스크에 대해 `popcnt`를 호출하여 `x`보다 작은 요소의 개수를 반환하는 것입니다.

이 트릭을 사용하면 셔플링 없이도 로컬 탐색을 효율적으로 수행할 수 있습니다.

```c++
typedef __m256i reg;

reg cmp(reg x, int *node) {
    reg y = _mm256_load_si256((reg*) node);
    return _mm256_cmpgt_epi32(x, y);
}

// x보다 작은 키의 개수를 반환
unsigned rank32(reg x, int *node) {
    reg m1 = cmp(x, node);
    reg m2 = cmp(x, node + 8);
    reg m3 = cmp(x, node + 16);
    reg m4 = cmp(x, node + 24);

    // m1/m3에서 하위 16비트를, m2/m4에서 상위 16비트를 가져옴
    m1 = _mm256_blend_epi16(m1, m2, 0b01010101);
    m3 = _mm256_blend_epi16(m3, m4, 0b01010101);
    m1 = _mm256_packs_epi16(m1, m3); // blendv를 사용할 수도 있지만 packs가 더 간단함

    unsigned mask = _mm256_movemask_epi8(m1);
    return __builtin_popcount(mask);    
}
```

이 절차 때문에 "키 영역"을 무한대 값으로 패딩해야 합니다. 이로 인해 비어 있는 셀에 메타데이터를 저장할 수 없게 됩니다 (SIMD 레인을 로드할 때 이를 마스킹하는 데 몇 사이클을 더 쓸 의향이 없다면 말이죠).

이제 `lower_bound`를 구현하기 위해 S+ 트리에서 했던 것처럼 트리를 내려가되, 자식 번호를 계산한 후 포인터를 가져옵니다.

```c++
int lower_bound(int _x) {
    unsigned k = root;
    reg x = _mm256_set1_epi32(_x);
    
    for (int h = 0; h < H - 1; h++) {
        unsigned i = rank32(x, &tree[k]);
        k = tree[k + B + i];
    }

    unsigned i = rank32(x, &tree[k]);

    return tree[k + i];
}
```

탐색 구현은 쉽고 오버헤드도 크지 않습니다. 어려운 부분은 삽입(insertion) 구현입니다.

### 삽입 (Insertion)

한편으로는 삽입을 올바르게 구현하는 데 많은 코드가 필요하지만, 다른 한편으로는 그 코드의 대부분이 아주 가끔만 실행되므로 성능에 크게 신경 쓸 필요는 없습니다. 대개의 경우 리프 노드에 도달하여(이미 방법을 알아냈죠) 새 키를 삽입하고, 키의 일부 접미사를 오른쪽으로 한 칸씩 이동하기만 하면 됩니다. 때때로 노드를 분할하거나 조상을 업데이트해야 하지만 이는 상대적으로 드문 일이므로 가장 일반적인 실행 경로에 먼저 집중해 보겠습니다.

$(B - 1)$개의 정렬된 요소가 있는 배열에 키를 삽입하기 위해, 요소들을 벡터 레지스터에 로드한 다음 [미리 계산된](/hpc/compilation/precalc/) 마스크를 사용하여 오른쪽으로 한 칸 이동시켜 [마스크 저장(mask-store)](/hpc/simd/masking)할 수 있습니다. 마스크는 주어진 `i`에 대해 어떤 요소를 써야 하는지 알려줍니다.

```c++
struct Precalc {
    alignas(64) int mask[B][B];

    constexpr Precalc() : mask{} {
        for (int i = 0; i < B; i++)
            for (int j = i; j < B - 1; j++)
                // i부터 B - 2까지 모든 요소가 이동해야 함
                mask[i][j] = -1;
    }
};

constexpr Precalc P;

void insert(int *node, int i, int x) {
    // 다음 레인의 첫 번째 요소를 덮어쓰지 않도록 오른쪽에서 왼쪽으로 반복해야 함
    for (int j = B - 8; j >= 0; j -= 8) {
        // 키 로드
        reg t = _mm256_load_si256((reg*) &node[j]);
        // 해당 마스크 로드
        reg mask = _mm256_load_si256((reg*) &P.mask[i][j]);
        // 오른쪽으로 한 칸 이동하여 마스크 쓰기
        _mm256_maskstore_epi32(&node[j + 1], mask, t);
    }
    node[i] = x; // 마지막으로 요소 자체를 씀
}
```

이 [constexpr 마법](/hpc/compilation/precalc/)이 우리가 사용하는 유일한 C++ 기능입니다.

더 효율적인 다른 방법들이 있을 수 있지만 지금은 여기서 멈추겠습니다.

노드를 분할할 때는 키의 절반을 다른 노드로 이동해야 하므로 이를 수행하는 또 다른 기본 함수(primitive)를 작성해 보겠습니다.

```c++
// 노드의 두 번째 절반을 이동하고 무한대 값으로 채움
void move(int *from, int *to) {
    const reg infs = _mm256_set1_epi32(INT_MAX);
    for (int i = 0; i < B / 2; i += 8) {
        reg t = _mm256_load_si256((reg*) &from[B / 2 + i]);
        _mm256_store_si256((reg*) &to[i], t);
        _mm256_store_si256((reg*) &from[B / 2 + i], infs);
    }
}
```

이 두 벡터 함수가 구현되었으므로 이제 삽입을 아주 신중하게 구현할 수 있습니다.

```c++
void insert(int _x) {
    // 절차의 시작은 lower_bound와 동일하지만,
    // 조상을 업데이트해야 할 경우를 대비해 경로를 저장함
    unsigned sk[10], si[10]; // 각 반복에서의 k와 i
    //           ^------^ 트리 높이가 10을 넘지 않는다고 가정함
    //                    (16^10개 이상의 요소가 필요함)
    
    unsigned k = root;
    reg x = _mm256_set1_epi32(_x);

    for (int h = 0; h < H - 1; h++) {
        unsigned i = rank32(x, &tree[k]);

        // 선택적으로 키 i를 즉시 업데이트
        tree[k + i] = (_x > tree[k + i] ? _x : tree[k + i]);
        sk[h] = k, si[h] = i; // 그리고 경로 저장
        
        k = tree[k + B + i];
    }

    unsigned i = rank32(x, &tree[k]);

    // 삽입이 완료되기 전에 가득 찼는지 확인을 시작할 수 있음
    bool filled  = (tree[k + B - 2] != INT_MAX);

    insert(tree + k, i, _x);

    if (filled) {
        // 노드 분할이 필요하므로 새로운 리프 노드 생성
        move(tree + k, tree + n_tree);
        
        int v = tree[k + B / 2 - 1]; // 새로 삽입될 키
        int p = n_tree;              // 새로 생성된 노드에 대한 포인터
        
        n_tree += B;

        for (int h = H - 2; h >= 0; h--) {
            // 루트에 도달하거나 노드가 분할되지 않을 때까지 올라가며 반복
            k = sk[h], i = si[h];

            filled = (tree[k + B - 3] != INT_MAX);

            // 노드에 이미 올바른 키(오른쪽 것)와 올바른 포인터(왼쪽 것)가 있음
            insert(tree + k,     i,     v);
            insert(tree + k + B, i + 1, p);
            
            if (!filled)
                return; // 완료

            // 새로운 내부 노드 생성
            move(tree + k,     tree + n_tree);     // 키 이동
            move(tree + k + B, tree + n_tree + B); // 포인터 이동

            v = tree[k + B / 2 - 1];
            tree[k + B / 2 - 1] = INT_MAX;

            p = n_tree;
            n_tree += 2 * B;
        }

        // 여기까지 왔다면 루트에 도달했고 루트가 둘로 분할되었음을 의미하므로 새로운 루트가 필요함
        tree[n_tree] = v;

        tree[n_tree + B] = root;
        tree[n_tree + B + 1] = p;

        root = n_tree;
        n_tree += 2 * B;
        H++;
    }
}
```

비효율적인 부분이 많지만 다행히 `if (filled)` 본문은 약 $\frac{B}{2}$번의 삽입마다 아주 드물게 실행됩니다. 삽입 성능이 최우선 순위는 아니므로 이대로 두겠습니다.

## 평가 (Evaluation)

`insert`와 `lower_bound`만 구현했으므로 이 두 가지를 측정해 보겠습니다.

평가가 합리적인 시간 내에 이루어지기를 원하므로, 벤치마크는 다음 두 단계를 번갈아 수행하는 루프로 구성됩니다.

- 개별 `insert`를 사용하여 구조체 크기를 $1.17^k$에서 $1.17^{k+1}$로 늘리고 소요 시간을 측정합니다.
- $10^6$번의 무작위 `lower_bound` 쿼리를 수행하고 소요 시간을 측정합니다.

크기 $10^4$에서 시작하여 $10^7$에서 종료하며, 총 약 50개의 데이터 포인트를 얻습니다. 두 쿼리 유형 모두 $[0, 2^{30})$ 범위 내에서 균등하게 생성되며 각 단계 사이에 독립적입니다. 데이터 생성 프로세스에서 중복 키를 허용하므로 `std::multiset` 및 `absl::btree_multiset`[^absl]과 비교했습니다. 하지만 편의상 `std::set` 및 `absl::btree`라고 부르겠습니다. 또한 세 가지 실행 모두 시스템 수준에서 [휴즈 페이지(hugepages)](/hpc/cpu-cache/paging)를 활성화했습니다.

[^absl]: Abseil의 B-트리와만 비교하는 것이 충분히 설득력이 없다고 생각하신다면, [언제든지](https://github.com/sslotin/amh-code/tree/main/b-tree) 벤치마크에 선호하는 탐색 트리를 추가해 보세요.

<!--

Keys are uniform, but we should not rely on that fact (e.g., using interpolation search).

It is common that >90% of operations are lookups. Optimizing searches is important because every other operation starts with locating a key.

I apologize to everyone else, but this is sort of your fault for not using a public benchmark.

-->

B− 트리의 성능은 조회(lookup)에 관해서는 원래 예측했던 것과 일치합니다.

![](../img/btree-absolute.svg)

상대적인 속도 향상은 구조체 크기에 따라 다릅니다. STL 대비 7-18배/3-8배, Abseil 대비 3-7배/1.5-2배 향상되었습니다.

![](../img/btree-relative.svg)

삽입은 스칼라 코드를 사용하는 `absl::btree`보다 1.5-2배만 더 빠릅니다. 삽입이 *그렇게* 느린 이유에 대한 제 최선의 추측은 데이터 의존성 때문입니다. 트리 노드가 변경될 수 있으므로 CPU는 이전 쿼리가 완료되기 전에 다음 쿼리 처리를 시작할 수 없습니다 (두 쿼리의 [실제 지연 시간](../s-tree/#comparison-with-stdlower_bound)은 대략 같으며 `lower_bound` 역처리량(reciprocal throughput)의 약 3배입니다).

![](../img/btree-absl.svg)

구조체 크기가 작을 때 `lower_bound`의 [역처리량](../s-tree/#comparison-with-stdlower_bound)은 불연속적인 단계로 증가합니다. 방문할 노드가 루트 하나뿐일 때는 3.5ns에서 시작하여, 노드가 2개일 때는 6.5ns, 3개일 때는 12ns로 증가합니다. 그 후 L2 캐시에 도달하고(그래프에는 표시되지 않음) 더 완만하게 증가하기 시작하지만, 트리 높이가 높아질 때 여전히 눈에 띄는 스파이크가 발생합니다.

흥미롭게도 B− 트리는 키를 하나만 저장할 때조차 `absl::btree`보다 성능이 뛰어납니다. `absl::btree`는 [분기 예측 실패(branch misprediction)](/hpc/pipelining/branching/)로 인해 약 5ns 정도 지체되는 반면, B− 트리의 탐색은 완전히 분기가 없습니다(branchless).

### 가능한 최적화 (Possible Optimizations)

이전의 자료 구조 최적화 시도에서는 가능한 많은 변수를 컴파일 타임 상수로 만드는 것이 큰 도움이 되었습니다. 컴파일러는 이러한 상수를 머신 코드에 하드코딩하고, 산술 연산을 단순화하며, 모든 루프를 풀고(unroll), 그 외 여러 가지 유용한 작업을 수행할 수 있습니다.

트리 높이가 일정하다면 전혀 문제가 되지 않겠지만, 실제로는 그렇지 않습니다. 하지만 높이가 *대체로* 일정합니다. 높이는 거의 변하지 않으며, 실제로 벤치마크 제약 조건 하에서 최대 높이는 6에 불과했습니다.

우리가 할 수 있는 일은 몇 가지 서로 다른 컴파일 타임 상수 높이에 대해 `insert` 및 `lower_bound` 함수를 미리 컴파일하고, 트리가 성장함에 따라 이들 사이를 전환하는 것입니다. 관용적인 C++ 방식은 가상 함수를 사용하는 것이지만, 저는 다음과 같이 원시 함수 포인터를 사용하여 명시적으로 하는 것을 선호합니다.

```c++
void (*insert_ptr)(int);
int (*lower_bound_ptr)(int);

void insert(int x) {
    insert_ptr(x);
}

int lower_bound(int x) {
    return lower_bound_ptr(x);
}
```

이제 트리 높이를 파라미터로 갖는 템플릿 함수를 정의하고, `insert` 함수 내부의 트리 성장 블록에서 트리가 자람에 따라 포인터를 변경합니다.

```c++
template <int H>
void insert_impl(int _x) {
    // ...
}

template <int H>
void insert_impl(int _x) {
    // ...
    if (/* 트리가 성장함 */) {
        // ...
        insert_ptr = &insert_impl<H + 1>;
        lower_bound_ptr = &lower_bound_impl<H + 1>;
    }
}

template <>
void insert_impl<10>(int x) {
    std::cerr << "이 깊이에 도달해서는 안 됩니다" << std::endl;
    exit(1);
}
```

<!--
insert_ptr = &insert_impl<1>;
lower_bound_ptr = &lower_bound_impl<1>;
-->

이 방법으로 성능 향상을 얻지는 못했지만, 여전히 이 접근 방식에 큰 기대를 걸고 있습니다. 컴파일러가 `sk`와 `si`를 제거하고 임시 저장소를 완전히 없애며 모든 것을 한 번만 읽고 계산하도록 `insert` 절차를 크게 최적화할 수 있기(이론적으로는) 때문입니다.

노드 분할이 드물어지도록 더 큰 블록 크기를 사용함으로써 삽입을 최적화할 수도 있겠지만, 이는 탐색 속도가 느려지는 대가를 치러야 합니다. 레이어마다 서로 다른 노드 크기를 시도해 볼 수도 있습니다. 리프 노드는 내부 노드보다 커야 할 것입니다.

**또 다른 아이디어**는 삽입 시 여분의 키를 형제 노드로 옮겨 노드 분할을 가능한 한 늦추는 것입니다.

이러한 특정 변형 중 하나는 B* 트리로 알려져 있습니다. 현재 노드가 가득 차면 마지막 키를 다음 노드로 옮기고, 두 노드가 모두 가득 차면 두 노드를 공동으로 분할하여 2/3가 채워진 세 개의 노드를 생성합니다. 이는 메모리 오버헤드를 줄이고(노드는 평균적으로 5/6가 채워짐) 분기 계수(fanout factor)를 높여 높이를 줄임으로써 모든 작업에 도움이 됩니다.

이 기술은 예를 들어 3-to-4 분할까지 확장될 수 있지만, 더 일반화하면 `insert`가 느려지는 대가를 치르게 됩니다.

**그리고 또 다른 아이디어**는 (일부) 포인터를 없애는 것입니다. 예를 들어 큰 트리의 경우 루트로 $16 \cdot 17$개 정도의 요소를 갖는 작은 [S+ 트리](../s-tree)를 사용할 수 있으며, 루트가 변경되는 드문 경우마다 처음부터 다시 구축합니다. 불행하게도 이를 전체 트리로 확장할 수는 없습니다. 동적 구조를 완전히 암시적(implicit)으로 만들면서 쿼리당 $\Omega(\sqrt n)$ 미만의 작업을 수행하는 것은 불가능하다는 논문이 어디선가 있었던 것으로 기억합니다.

[스키프 리스트(skip list)](https://en.wikipedia.org/wiki/Skip_list)와 같은 트리가 아닌 자료 구조를 시도해 볼 수도 있습니다. 이를 벡터화하려는 [성공적인 시도](https://doublequan.github.io/)도 있었습니다. 비록 속도 향상이 그렇게 인상적이지는 않았지만 말이죠. 스킵 리스트가 특히 개선될 수 있다는 기대는 낮지만, 동시성 환경에서는 더 높은 전체 처리량을 달성할 수 있을 것입니다.

### 기타 작업 (Other Operations)

키를 *삭제*하기 위해, 동일한 마스크 저장 트릭을 사용하여 노드에서 키를 찾아 제거할 수 있습니다. 그 후 노드가 최소 절반 이상 채워져 있으면 완료입니다. 그렇지 않으면 다음 형제 노드에서 키를 빌려오려고 시도합니다. 형제 노드에 $\frac{B}{2}$개 이상의 키가 있으면 첫 번째 키를 가져오고 형제 노드의 키를 왼쪽으로 한 칸씩 이동시킵니다. 그렇지 않으면 현재 노드와 다음 노드 모두 키가 $\frac{B}{2}$개 미만이므로 두 노드를 병합할 수 있으며, 그 후 부모로 이동하여 거기서 반복적으로 키를 삭제합니다.

구현하고 싶은 또 다른 기능은 *순회(iteration)*입니다. `l`부터 `r`까지 각 키를 대량으로 로드하는 것은 데이터베이스의 `SELECT abc ORDER BY xyz`와 같은 쿼리에서 매우 흔한 패턴입니다. B+ 트리는 이러한 빠른 순회를 위해 일반적으로 데이터 레이어에 다음 노드에 대한 포인터를 저장합니다. B− 트리에서는 훨씬 작은 노드 크기를 사용하므로 이 방식을 사용하면 [포인터 추적(pointer chasing)](/hpc/cpu-cache/latency/) 문제가 발생할 수 있습니다. 부모 노드로 가서 $B$개의 포인터를 모두 읽는 것이 이 문제를 해결하므로 아마 더 빠를 것입니다. 따라서 조상의 스택(`insert`에서 사용한 `sk` 및 `si` 배열)이 반복자 역할을 할 수 있으며, 노드에 포인터를 별도로 저장하는 것보다 나을 수도 있습니다.

`std::set`이 제공하는 거의 모든 기능을 쉽게 구현할 수 있지만, 다른 B-트리와 마찬가지로 B− 트리가 `std::set`을 직접 대체하기는 매우 어렵습니다. 이는 포인터 안정성(pointer stability) 요구 사항 때문입니다. 요소에 대한 포인터는 해당 요소가 삭제되지 않는 한 유효해야 하는데, 노드를 수시로 분할하고 병합할 때는 이를 달성하기 어렵습니다. 이는 탐색 트리뿐만 아니라 대부분의 자료 구조에서 발생하는 주요 문제입니다. 포인터 안정성과 높은 성능을 동시에 얻는 것은 거의 불가능에 가깝습니다.

<!--
Maybe if the C++ standard adds something like `std::set_with_unstable_pointers`

We can't store junk in keys.
-->

## 감사의 말 (Acknowledgements)

Abseil에서의 B-트리 적용 가능성 및 사용에 대해 유의미한 토론을 해주신 Google의 [Danila Kutenin](https://danlark.org/)에게 감사를 표합니다.

<!-- One interesting use case is *rope*, also known as *cord*, which is used for wrapping strings in a tree to support mass operations. For example, editing a very large text file. Which is the topic. -->
