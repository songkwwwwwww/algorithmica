---
title: 요약
weight: 99
ignoreIndexing: true
draft: true
---

이제 우리가 배운 것들을 요약할 준비가 되었습니다.

루프 언롤링(Loop unrolling).

최적화 체크리스트(가장 쉬운 것부터 어려운 것 순):

0. 최적화 플래그를 켭니다 (`-march=native`, `-O3`, `-ffast-math`, `-funroll-loops`)
1. 알고리즘이 메모리 중심(memory-bound)인지 계산 중심(compute-bound)인지 판단합니다.
2. 블로킹(Blocking): 데이터를 캐시에 들어가는 크기로 쪼개어 처리합니다.
3. 메모리 접근 패턴: 모든 읽기 작업을 선형화(linearize)하도록 노력합니다.
4. 프리페칭(Prefetching): 접근 패턴을 예측하기 어렵거나 포트가 비어있다면 프리페칭 단계를 추가합니다.
5. 분기(Branching): 분기문을 제거합니다.
6. 루프 언롤링(Loop Unrolling): `#pragma GCC unroll n` 등을 사용합니다.
7. 데이터 의존성(Data Dependencies): 명령어 테이블이나 llvm-mca를 확인합니다.
8. SIMD: 루프를 단순화합니다.
9. 산술(Arithmetic): 사용 가능한 가장 작은 타입을 사용하고 `-ffast-math`를 켭니다.
10. 시간 지역성(Temporal locality)을 고려합니다.
11. 할당(Allocations)을 줄입니다.
12. 프로파일링을 수행합니다.
