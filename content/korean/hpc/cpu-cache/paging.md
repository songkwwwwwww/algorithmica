---
title: 메모리 페이징 (Memory Paging)
weight: 12
---

[다시 한 번](../associativity) 스트라이드 증가 루프를 살펴보겠습니다:

```cpp
const int N = (1 << 13);
int a[D * N];

for (int i = 0; i < D * N; i += D)
    a[i] += 1;
```

스트라이드 $D$를 변경하고 총 반복 횟수 $N$이 일정하도록 배열 크기를 비례해서 늘립니다. 총 메모리 액세스 횟수도 일정하므로, 모든 $D \geq 16$에 대해 정확히 $N$개의 캐시 라인 — 정확히는 $64 \cdot N = 2^6 \cdot 2^{13} = 2^{19}$ 바이트를 가져와야 합니다. 이는 스텝 크기와 관계없이 L2 캐시에 딱 들어맞으며, 처리량 그래프는 평평하게 보여야 합니다.

이번에는 $D$ 값을 1024까지 더 넓게 고려해 보겠습니다. 256 근처부터 그래프는 확실히 평평하지 않습니다:

![](../img/strides.svg)

이 변칙 역시 캐시 시스템 때문이지만, 표준 L1-L3 데이터 캐시와는 관련이 없습니다. [가상 메모리(virtual memory)](/hpc/external-memory/virtual)가 원인이며, 특히 가상 메모리 페이지의 물리 주소를 검색하는 역할을 하는 캐시인 *TLB(translation lookaside buffer)*가 그 주범입니다.

[제 CPU](https://en.wikichip.org/wiki/amd/microarchitectures/zen_2)에는 두 단계의 TLB가 있습니다:

- L1 TLB는 64개의 항목을 가지며, 페이지 크기가 4K라면 L2 TLB로 가지 않고도 $64 \times 4K = 512K$의 활성 메모리를 처리할 수 있습니다.
- L2 TLB는 2048개의 항목을 가지며, 페이지 테이블로 가지 않고도 $2048 \times 4K = 8M$의 메모리를 처리할 수 있습니다.

$D$가 256이 될 때 얼마나 많은 메모리가 할당될까요? 예상하셨겠지만 $8K \times 256 \times 4B = 8M$이며, 이는 정확히 L2 TLB가 감당할 수 있는 한계치입니다. $D$가 그보다 커지면 일부 요청이 메인 페이지 테이블로 리다이렉트되기 시작합니다. 메인 페이지 테이블은 지연 시간이 길고 처리량이 매우 제한적이어서 전체 계산의 병목 현상이 됩니다.

### 페이지 크기 변경 (Changing Page Size)

8MB의 속도 저하 없는 메모리는 매우 빡빡한 제한처럼 보입니다. 이를 해결하기 위해 하드웨어의 특성을 바꿀 수는 없지만, 페이지 크기를 늘려 TLB 용량에 가해지는 압박을 줄일 수는 *있습니다*.

현대 운영 체제는 전역적으로나 개별 할당에 대해 페이지 크기를 설정할 수 있게 해줍니다. CPU는 정해진 페이지 크기 세트만 지원합니다 — 제 CPU의 경우 4K 또는 2M 페이지를 사용할 수 있습니다. 또 다른 전형적인 페이지 크기는 1G인데, 이는 보통 수백 기가바이트의 RAM을 갖춘 서버급 하드웨어에서나 의미가 있습니다. 기본 4K를 넘는 크기는 리눅스에서는 *거대 페이지(huge pages)*, 윈도우에서는 *대형 페이지(large pages)*라고 불립니다.

리눅스에는 거대 페이지 할당을 관리하는 특수 시스템 파일이 있습니다. 모든 할당에 대해 커널이 거대 페이지를 제공하도록 설정하는 방법은 다음과 같습니다:

```bash
$ echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

거대 페이지를 이렇게 전역적으로 활성화하는 것이 항상 좋은 아이디어는 아닙니다. 메모리 세분성(granularity)이 떨어지고 프로세스가 소비하는 최소 메모리가 늘어나기 때문입니다 — 일부 환경에서는 여유 메모리 메가바이트 수보다 프로세스 수가 더 많을 수도 있습니다. 그래서 `always`와 `never` 외에 해당 파일에는 세 번째 옵션이 있습니다:

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```

`madvise`는 프로그램이 커널에 거대 페이지 사용 여부를 조언(advise)할 수 있게 해주는 특수 시스템 콜로, 필요할 때만 거대 페이지를 할당하는 데 사용할 수 있습니다. 이 옵션이 활성화되어 있다면 C++에서 다음과 같이 사용할 수 있습니다:

```c++
#include <sys/mman.h>

void *ptr = std::aligned_alloc(page_size, array_size);
madvise(ptr, array_size, MADV_HUGEPAGE);
```

메모리 영역이 해당 정렬을 가지고 있을 때만 거대 페이지를 사용한 할당을 요청할 수 있습니다.

윈도우에도 유사한 기능이 있습니다. 윈도우의 메모리 API는 이 두 기능을 하나로 결합합니다:

```c++
#include "memoryapi.h"

void *ptr = VirtualAlloc(NULL, array_size,
                         MEM_RESERVE | MEM_COMMIT | MEM_LARGE_PAGES, PAGE_READWRITE);
```

두 경우 모두 `array_size`는 `page_size`의 배수여야 합니다.

### 거대 페이지의 영향 (Impact of Huge Pages)

거대 페이지를 할당하는 두 방식 모두 즉시 성능 곡선을 평평하게 만듭니다:

![](../img/strides-hugepages.svg)

거대 페이지를 활성화하면 L2 캐시에 들어가지 않는 배열에 대해 [지연 시간(latency)](../latency)이 최대 10-15% 향상됩니다:

![](../img/permutation-hugepages.svg)

일반적으로 희소 읽기(sparse reads)가 발생하는 경우 거대 페이지를 활성화하는 것이 좋습니다. 성능이 약간 향상되며 ([거의](../aos-soa)) 저하되지 않기 때문입니다.

그렇긴 하지만, 하드웨어나 컴퓨팅 환경의 제약으로 인해 항상 사용 가능한 것은 아니므로 거대 페이지에만 의존해서는 안 됩니다. 데이터 액세스를 공간적으로 그룹화하는 것이 유익한 [많은](../cache-lines) [다른](../prefetching) [이유들](../aos-soa)이 있으며, 이는 자동으로 페이징 문제를 해결해 줍니다.

<!--


virtually located, physically tagged

Actually, TLB misses may stall memory reads for the same reason. The TLB cache is called "lookaside" because the lookup can happen independently from normal data cache lookups. L1 and L2 caches on the other side are private to the core, and so they can store virtual addresses and be queried concurrently with TLB — after fetching a cache line, its tag is used to restore the physical address, which is then checked against the concurrently fetched TLB entry. This trick does not work for shared memory however, because their bandwidth is limited, and dispatching read queries there for no reason is not a good idea in general. So we can observe a similar effect in L3 and RAM reads when the page does not fit L1 TLB and L2 TLB respectively.

For sparse reads, it often makes sense to increase page size, which improves the latency.

Typical size of a page is 4KB, but it can be up to 1G or so for large databases, but enabling it by default is not a good idea as scenarios when we have a VPS with 256M or RAM and more than 256 processes are not uncommon.

Typical page sizes are 4K, 2M and 1G (e.g., allowing for 256K, 128M, 64G memory regions to be stored in a 64-entry L1 TLB respectively).


- There are other types of cache inside CPUs that are used for things other than data. The most important for us are *instruction cache* (I-cache), which is used to speed up the fetching of machine code from memory, and *translation lookaside buffer* (TLB), which is used to store physical locations of virtual memory pages, which is instrumental to the efficiency of virtual memory.

You can fetch this information for your architecture with `cpuid` command.

-->
