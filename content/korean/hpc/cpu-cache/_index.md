---
title: RAM 및 CPU 캐시 (RAM & CPU Caches)
weight: 9
---

[이전 장](../external-memory)에서 우리는 [외부 메모리 모델(external memory model)](../external-memory/model)을 사용하여 메모리 집약적 알고리즘의 성능을 추정하며 컴퓨터 메모리를 이론적인 관점에서 공부했습니다.

외부 메모리 모델은 HDD나 네트워크 저장소와 관련된 계산에서 어느 정도 정확합니다. 메모리 내 값에 대한 산술 연산 비용이 외부 I/O 연산에 비해 무시할 수 있을 정도로 작기 때문입니다. 하지만 이러한 연산 비용이 서로 비슷해지는 캐시 계층 구조의 하위 레벨에서는 이 모델이 너무 부정확합니다.

메모리 내 알고리즘에 대해 더 정밀한 최적화를 수행하려면 CPU 캐시 시스템의 다양한 세부 사항을 고려해야 합니다. 무미건조한 사양과 이론적 한계치가 담긴 지루한 인텔 문서들을 산더미처럼 공부하는 대신, 우리는 실제 코드에서 자주 발생하는 액세스 패턴을 닮은 수많은 작은 벤치마크 프로그램들을 실행하여 이러한 파라미터들을 실험적으로 추정해 볼 것입니다.

<!--

At this level, we can no longer simply ignore either all arithmetic or memory operations. To perform more fine-grained optimization of realistic programs, we need to know the cost of memory accesses on real systems and in real units — in cycles and nanoseconds — along with many other intricacies of the RAM and CPU cache system.

-->

### 실험 설정 (Experimental Setup)

이전과 마찬가지로 모든 실험은 "Zen 2" CPU인 Ryzen 7 4700U에서 실행할 것입니다. 주요 캐시 관련 사양은 다음과 같습니다:

- 8개의 물리 코어 (하이퍼스레딩 없음), 2GHz로 작동 (부스트 모드 시 4.1GHz — [비활성화함](/hpc/profiling/noise));
- 256K의 8-way 집합 연관(set associative) L1 데이터 캐시, 또는 코어당 32K;
- 4M의 8-way 집합 연관 L2 캐시, 또는 코어당 512K;
- 8M의 16-way 집합 연관 L3 캐시, 8개 코어 간에 [공유](sharing);
- 16GB (2x8G) DDR4 RAM @ 2667MHz.

리눅스에서 `dmidecode -t cache` 또는 `lshw -class memory`를 실행하거나 윈도우에서 [CPU-Z](https://en.wikipedia.org/wiki/CPU-Z)를 설치하여 여러분의 하드웨어와 비교해 볼 수 있습니다. [WikiChip](https://en.wikichip.org/wiki/amd/ryzen_7/4700u)과 [7-CPU](https://www.7-cpu.com/cpu/Zen2.html)에서도 CPU에 대한 추가 세부 정보를 찾을 수 있습니다. 모든 결론이 현존하는 모든 CPU 플랫폼에 일반화되는 것은 아닙니다.

<!--

Although the CPU can be clocked at 4.1GHz in boost mode, we will perform most experiments at 2GHz to reduce noise — so keep in mind that in realistic applications the numbers can be multiplied by 2.

-->

[컴파일러가 사용되지 않는 값을 최적화하여 제거하는 것을 방지](/hpc/profiling/noise/)하는 데 어려움이 있어, 이 글의 코드 스니펫들은 설명을 위해 약간 단순화되었습니다. 직접 재현해 보고 싶다면 [코드 저장소](https://github.com/sslotin/amh-code/tree/main/cpu-cache)를 확인하십시오.

### 감사의 말 (Acknowledgements)

이 장은 Igor Ostrovsky의 "[Gallery of Processor Cache Effects](http://igoro.com/archive/gallery-of-processor-cache-effects/)"와 Ulrich Drepper의 "[What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)"에서 영감을 받았습니다. 두 문서 모두 훌륭한 참고 자료가 될 것입니다.

<!--

### Recall: CPU Caches

If you jumped to this page straight from Google or just forgot what [we've been doing](../), here is a brief summary of how memory operations work in CPUs:

The last few points may be a bit hand-wavy, but don't worry: they will become clear as we go along with the experiments and demonstrate it all in action.

## Summary and Lessons Learned

Excluding TLB, our experiments suggest the following:

| Type | Size | Latency | Bandwidth |
|:-----|:-----|---------|-----------|
| L1   | 32K  | 2ns     | $\infty$  |
| L2   | 512K | 10ns    | 50G/s     |
| L3   | 4M   | 50ns    | 35G/s     |
| RAM  | GB   | 100ns   | 8G/s      |

There are more thorough [measurements for Zen 2](https://www.7-cpu.com/cpu/Zen2.html).

We can learn valuable lessons from our experiments. There are two types of memory-bound algorithms. Loops or data structures.

**Latency-constrained.** For the purpose of designing algorithms, a more important characteristic is the **bandwidth-latency product** which basically tells how many cache lines you can request while waiting for the first one without queueing up. It is around 5 or more on most systems. CPUs can detect simple patterns such as linear iteration forward or backward.

**Bandwidth-constrained.** We started the previous section with how it is not relevant which algorithm is used to determine cache eviction. In most practical cases, this is really the case.

But in some cases the specifics start to matter. In set-associative cache, there may be a problem when we are only working with data cells that all map to the same cache line. When is this the case? When we are considering memory locations that are all have the same remainder modulo some large power of two.

Unfortunately, this happens quite often, as we programmers love using powers of two for our algorithms and data structures.

Fortunately, this is easy to fix: just don't use powers of two. Not necessarily for the algorithm, but at least for the memory layout.

More fundamental [academic paper](https://www2.eecs.berkeley.edu/Pubs/TechRpts/1993/CSD-93-767.pdf) by Rafael Saavedra and Alan Smith.

-->
