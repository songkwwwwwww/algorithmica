---
title: 현대 하드웨어를 위한 알고리즘
menuTitle: HPC
weight: 5
#authors:
#- Sergey Slotin
#created: "2021년 2월"
#date: 2021-09-16
noToc: true
---

이것은 [Sergey Slotin](http://sereja.me/)이 집필 중인 고성능 컴퓨팅(HPC) 서적인 "현대 하드웨어를 위한 알고리즘(Algorithms for Modern Hardware)"입니다.

이 책은 성능 엔지니어와 실무 알고리즘 연구자부터 고급 알고리즘 코스를 막 마치고 시간 복잡도를 $O(n \log n)$에서 $O(n \log \log n)$으로 줄이는 것보다 더 실질적인 프로그램 가속 방법을 배우고 싶은 컴퓨터 공학 학부생까지 모두를 대상으로 합니다.

모든 도서 자료는 [GitHub에 호스팅](https://github.com/algorithmica-org/algorithmica)되어 있으며, 코드는 [별도의 저장소](https://github.com/sslotin/scmm-code)에 있습니다. 이 프로젝트는 공동 작업 프로젝트는 아니지만, 모든 기여와 피드백은 대환영입니다.

### FAQ

**버그/오타 수정.** 어느 페이지에서든 오류를 발견하면 선호도 순서대로 다음 중 하나를 수행해 주세요.

- 모든 페이지의 우측 상단에 있는 연필 아이콘을 클릭하여 즉시 수정하거나([Prose](https://prose.io/) 에디터 사용), 더 전통적인 방식인 GitHub에서 페이지를 직접 수정합니다(소스 링크도 우측 상단에 있습니다).
- [GitHub에 이슈](https://github.com/algorithmica-org/algorithmica/issues)를 생성합니다.
- 저에게 직접 [말해 주세요](http://sereja.me/).

또는 이 글이 논의되고 있는 다른 웹사이트에 댓글을 남겨 주세요. 저는 제가 태그된 [HackerNews](https://news.ycombinator.com/from?site=algorithmica.org), [CodeForces](https://codeforces.com/profile/sslotin), [Twitter](https://twitter.com/sergey_slotin) 스레드 대부분을 읽습니다.

**출시일.** 이 책은 여러 파트로 나뉘어 있으며, 긴 휴식 기간을 사이에 두고 순차적으로 완성할 계획입니다. 제1부 성능 엔지니어링은 2022년 3월 기준으로 약 75% 완성되었으며, 이번 여름까지 95% 이상 완성되기를 희망합니다.

이와 같은 오픈 소스 도서의 "출시"는 본질적으로 다음을 의미합니다.

- 모든 필수 섹션을 마무리하고 모든 할 일(TODO)을 채우는 것,
- 목차를 거의 확정하는 것(케이스 스터디 제외),
- 마지막으로 대대적인 교열을 수행하는 것(전문 편집자의 도움을 받기를 희망합니다. 저는 아직 영어에서 쉼표가 어떻게 작동하는지 잘 모릅니다),
- 일러스트레이션을 그리는 것(현재 표시된 것 중 많은 것들을 빌려왔습니다),
- 인쇄 최적화된 PDF를 만들고 이를 배포하는 가장 좋은 방법을 찾는 것.

그 이후에는 주로 오류를 수정하고 기술 변화나 새로운 알고리즘 발전을 반영하는 사소한 편집만 진행할 예정입니다. 전자책/인쇄판은 "원하는 만큼 지불(pay what you want)"하는 방식으로 판매될 가능성이 높으며, 어떤 경우에도 웹 버전은 항상 온라인에서 완전히 무료로 제공될 것입니다.

**사전 주문 / 금전적 지원.** 저의 불행한 국적과 출생지 때문에 현재로서는 불가능합니다. 국제 제재를 준수하면서 [전쟁](https://en.wikipedia.org/wiki/2022_Russian_invasion_of_Ukraine)을 후원하지 않고, 탈세로 감옥에 가지 않는 방법을 찾을 때까지는요.

그러니 신경 쓰지 마세요. 이 책을 지원하고 싶다면 그냥 공유해 주시고 오타 수정을 도와주세요. 그것만으로도 충분합니다.

**번역.** 이 웹사이트에는 번역을 생성하고 관리하는 별도의 기능이 있습니다. 이미 이 책을 이탈리아어와 중국어로 번역하고 싶어 하는 훌륭한 분들로부터 연락을 받았습니다. (그리고 저도 제 모국어인 러시아어로 일부 번역할 예정입니다.)

하지만 책이 아직 진화 중이기 때문에 적어도 파트 1이 끝날 때까지는 번역을 시작하지 않는 것이 좋을 것 같습니다. 그렇더라도 어떤 글이든 번역하여 블로그에 게시하는 것은 언제든지 환영합니다. 링크를 보내주시면 나중에 중앙 집중식 번역이 시작될 때 통합할 수 있습니다.

**러시아어 버전 "번역".** [ru.algorithmica.org/cs/](https://ru.algorithmica.org/cs/)에 게시된 글들은 고급 성능 엔지니어링에 관한 것이 아니라 주로 고전적인 컴퓨터 과학 알고리즘에 관한 것입니다. 점근적 복잡도 이상의 가속 방법에 대해서는 논의하지 않습니다. 그곳의 정보 대부분은 고유한 것이 아니며 이미 인터넷의 다른 곳에 영어로 존재합니다. 예를 들어, 비슷한 취지의 [cp-algorithms.com](https://cp-algorithms.com/) 등이 있습니다.

**대학에서의 성능 엔지니어링 교육.** 이 책을 쓰는 제 목표 중 하나는 대학에서 컴퓨터 공학(정확히 말하면 알고리즘 설계)이 가르쳐지는 방식을 바꾸는 것입니다. 이에 대해 자세히 설명해 보겠습니다.

대부분의 컴퓨터 공학 교육과정이 기반으로 하는 두 권의 매우 영향력 있는 교과서가 있습니다. 둘 다 의심할 여지 없이 훌륭하지만, [하나는](https://en.wikipedia.org/wiki/The_Art_of_Computer_Programming) 50년 전의 것이고, [다른 하나는](https://en.wikipedia.org/wiki/Introduction_to_Algorithms) 30년 전의 것이며, 그 이후로 [컴퓨터는 많이 변했습니다](/hpc/complexity/hardware). 점근적 복잡도가 이제 유일한 결정 요인이 아닙니다. 현대의 실무 알고리즘 설계에서는 은하계 규모의 입력에서 이론적으로 적은 연산을 수행하는 방식보다, 하드웨어에서 사용 가능한 다양한 유형의 병렬성을 더 잘 활용하는 방식을 선택합니다.

그럼에도 불구하고 대부분의 대학 컴퓨터 공학 커리큘럼은 이러한 변화를 완전히 무시하고 있습니다. MIT의 "[Performance Engineering of Software Systems](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-172-performance-engineering-of-software-systems-fall-2018/)", 알토 대학의 "[Programming Parallel Computers](https://ppc.cs.aalto.fi/)", 그리고 데니스 바흐발로프(Denis Bakhvalov)의 "[Performance Ninja](https://github.com/dendibakh/perf-ninja)"와 같은 훌륭한 비학술적 코스들이 이를 바로잡으려 노력하고 있지만, 대부분의 컴퓨터 공학 졸업생들은 여전히 현대 하드웨어를 1990년대의 것처럼 취급합니다.

제가 정말로 달성하고 싶은 것은 성능 엔지니어링이 알고리즘 입문 직후에 가르쳐지는 것입니다. 이 주제에 대한 최초의 포괄적인 교과서를 쓰는 것이 그 큰 부분이며, 제가 여름까지 이를 마무리하려고 서두르는 이유도 다음 학년도에 대학들이 이를 채택할 수 있게 하기 위해서입니다. 하지만 새로운 코스를 만들려면 그 이상이 필요합니다. 균형 잡힌 커리큘럼, 코스 인프라, 강의 슬라이드, 실습 과제 등이 필요합니다. 그래서 책을 마친 후 한동안은 성능 엔지니어링을 *가르치기* 위한 코스 자료와 도구들을 작업할 예정입니다. 이를 현실로 만들고 싶어 하는 다른 분들과의 협력도 기대하고 있습니다.

### 파트 I: 성능 엔지니어링

첫 번째 파트에서는 컴퓨터 아키텍처의 기초와 단일 스레드 알고리즘의 최적화를 다룹니다.

캐싱, SIMD, 파이프라이닝과 같은 주요 CPU 최적화 주제를 살펴보고 C++ 예제를 제공하며, 이어서 대규모 케이스 스터디를 통해 STL 알고리즘이나 데이터 구조보다 상당한 속도 향상을 달성하는 방법을 보여줍니다.

예정된 목차:

```
0. 서문 (Preface)
1. 복잡도 모델 (Complexity Models)
 1.1. 현대 하드웨어 (Modern Hardware)
 1.2. 프로그래밍 언어 (Programming Languages)
 1.3. 계산 모델 (Models of Computation)
 1.4. 최적화 시점 (When to Optimize)
2. 컴퓨터 아키텍처 (Computer Architecture)
 1.1. 명령어 집합 아키텍처 (Instruction Set Architectures)
 1.2. 어셈블리어 (Assembly Language)
 1.3. 루프와 조건문 (Loops and Conditionals)
 1.4. 함수와 재귀 (Functions and Recursion)
 1.5. 간접 분기 (Indirect Branching)
 1.6. 기계어 코드 레이아웃 (Machine Code Layout)
 1.7. 시스템 호출 (System Calls)
 1.8. 가상화 (Virtualization)
3. 명령어 수준 병렬성 (Instruction-Level Parallelism)
 3.1. 파이프라인 해저드 (Pipeline Hazards)
 3.2. 분기 비용 (The Cost of Branching)
 3.3. 분기 없는 프로그래밍 (Branchless Programming)
 3.4. 명령어 테이블 (Instruction Tables)
 3.5. 명령어 스케줄링 (Instruction Scheduling)
 3.6. 처리량 컴퓨팅 (Throughput Computing)
 3.7. 이론적 성능 한계 (Theoretical Performance Limits)
4. 컴파일 (Compilation)
 4.1. 컴파일 단계 (Stages of Compilation)
 4.2. 플래그와 타겟 (Flags and Targets)
 4.3. 상황별 최적화 (Situational Optimizations)
 4.4. 계약 프로그래밍 (Contract Programming)
 4.5. 비-제로 비용 추상화 (Non-Zero-Cost Abstractions)
 4.6. 컴파일 타임 계산 (Compile-Time Computation)
 4.7. 산술 최적화 (Arithmetic Optimizations)
 4.8. 컴파일러가 할 수 있는 것과 할 수 없는 것 (What Compilers Can and Can't Do)
5. 프로파일링 (Profiling)
 5.1. 인스트루먼테이션 (Instrumentation)
 5.2. 통계적 프로파일링 (Statistical Profiling)
 5.3. 프로그램 시뮬레이션 (Program Simulation)
 5.4. 기계어 코드 분석기 (Machine Code Analyzers)
 5.5. 벤치마킹 (Benchmarking)
 5.6. 정확한 결과 얻기 (Getting Accurate Results)
6. 산술 (Arithmetic)
 6.1. 부동 소수점 수 (Floating-Point Numbers)
 6.2. 구간 산술 (Interval Arithmetic)
 6.3. 뉴턴 방법 (Newton's Method)
 6.4. 빠른 역 제곱근 (Fast Inverse Square Root)
 6.5. 정수 (Integers)
 6.6. 정수 나눗셈 (Integer Division)
 6.7. 비트 조작 (Bit Manipulation)
(6.8. 데이터 압축 (Data Compression))
7. 정수론 (Number Theory)
 7.1. 모듈러 역수 (Modular Inverse)
 7.2. 몽고메리 곱셈 (Montgomery Multiplication)
(7.3. 유한체 (Finite Fields))
(7.4. 오류 정정 (Error Correction))
 7.5. 암호학 (Cryptography)
 7.6. 해싱 (Hashing)
 7.7. 난수 생성 (Random Number Generation)
8. 외부 메모리 (External Memory)
 8.1. 메모리 계층 구조 (Memory Hierarchy)
 8.2. 가상 메모리 (Virtual Memory)
 8.3. 외부 메모리 모델 (External Memory Model)
 8.4. 외부 정렬 (External Sorting)
 8.5. 리스트 랭킹 (List Ranking)
 8.6. 교체 정책 (Eviction Policies)
 8.7. 캐시 무관 알고리즘 (Cache-Oblivious Algorithms)
 8.8. 공간 및 시간 지역성 (Spacial and Temporal Locality)
(8.9. B-트리 (B-Trees))
(8.10. 아고점 알고리즘 (Sublinear Algorithms))
(9.13. 메모리 관리 (Memory Management))
9. RAM 및 CPU 캐시 (RAM & CPU Caches)
 9.1. 메모리 대역폭 (Memory Bandwidth)
 9.2. 메모리 지연 시간 (Memory Latency)
 9.3. 캐시 라인 (Cache Lines)
 9.4. 메모리 공유 (Memory Sharing)
 9.5. 메모리 수준 병렬성 (Memory-Level Parallelism)
 9.6. 프리페칭 (Prefetching)
 9.7. 정렬 및 패킹 (Alignment and Packing)
 9.8. 포인터 대안 (Pointer Alternatives)
 9.9. 캐시 연관성 (Cache Associativity)
 9.10. 메모리 페이징 (Memory Paging)
 9.11. AoS 및 SoA
10. SIMD 병렬성 (SIMD Parallelism)
 10.1. 인트린직 및 벡터 타입 (Intrinsics and Vector Types)
 10.2. 데이터 이동 (Moving Data)
 10.3. 리덕션 (Reductions)
 10.4. 마스킹 및 블렌딩 (Masking and Blending)
 10.5. 레지스터 내 셔플 (In-Register Shuffles)
 10.6. 자동 벡터화 및 SPMD (Auto-Vectorization and SPMD)
11. 알고리즘 케이스 스터디 (Algorithm Case Studies)
 11.1. 이진 GCD (Binary GCD)
(11.2. 소수 판별 및 체 (Prime Number Sieves))
 11.3. 정수 인수분해 (Integer Factorization)
 11.4. 로지스틱 회귀 (Logistic Regression)
 11.5. 큰 정수 및 카라츠바 알고리즘 (Big Integers & Karatsuba Algorithm)
 11.6. 빠른 푸리에 변환 (Fast Fourier Transform)
 11.7. 수론적 변환 (Number-Theoretic Transform)
 11.8. SIMD를 이용한 Argmin (Argmin with SIMD)
 11.9. SIMD를 이용한 접두사 합 (Prefix Sum with SIMD)
 11.10. 10진수 정수 읽기 (Reading Decimal Integers)
 11.11. 10진수 정수 쓰기 (Writing Decimal Integers)
(11.12. 실수 읽기 및 쓰기 (Reading and Writing Floats))
(11.13. 문자열 검색 (String Searching))
 11.14. 정렬 (Sorting)
 11.15. 행렬 곱셈 (Matrix Multiplication)
12. 데이터 구조 케이스 스터디 (Data Structure Case Studies)
 12.1. 이진 검색 (Binary Search)
 12.2. 정적 B-트리 (Static B-Trees)
(12.3. 검색 트리 (Search Trees))
 12.4. 세그먼트 트리 (Segment Trees)
(12.5. 트라이 (Tries))
(12.6. 구간 최소 쿼리 (Range Minimum Query))
 12.7. 해시 테이블 (Hash Tables)
(12.8. 비트맵 (Bitmaps))
(12.9. 확률적 필터 (Probabilistic Filters))
```

우리가 가속화할 멋진 작업들:

- 2배 빠른 GCD (`std::gcd` 대비)
- 8-15배 빠른 이진 검색 (`std::lower_bound` 대비)
- 5-10배 빠른 세그먼트 트리 (펜윅 트리 대비)
- 5배 빠른 해시 테이블 (`std::unordered_map` 대비)
- 2배 빠른 popcount (`popcnt`를 반복 호출하는 것 대비)
- 35배 빠른 일련의 정수 파싱 (`scanf` 대비)
- ?배 빠른 정렬 (`std::sort` 대비)
- 2배 빠른 합계 (`std::accumulate` 대비)
- 2-3배 빠른 접두사 합 (단순 구현 대비)
- 10배 빠른 argmin (단순 구현 대비)
- 10배 빠른 배열 검색 (`std::find` 대비)
- 15배 빠른 검색 트리 (`std::set` 대비)
- 100배 빠른 행렬 곱셈 ("for-for-for" 대비)
- 최적의 워드 크기 정수 인수분해 (60비트 정수당 ~0.4ms)
- 최적의 카라츠바 알고리즘
- 최적의 FFT

분량: 450-600 페이지  
출시 예정일: 2022년 3분기

### 파트 II: 병렬 알고리즘

동시성, 병렬성 모델, 컨텍스트 스위칭, 그린 스레드, 동시성 런타임, 캐시 일관성, 동기화 기본 요소, OpenMP, 리덕션, 스캔, 리스트 랭킹, 그래프 알고리즘, 락 프리(lock-free) 데이터 구조, 이기종 컴퓨팅, CUDA, 커널, 워프(warp), 블록, 행렬 곱셈, 정렬.

분량: 150-200 페이지  
출시 예정일: 2023-2024년?

### 파트 III: 분산 컴퓨팅

네트워킹, 메시지 패싱, 액터 모델, 통신 제약 알고리즘, 분산 기본 요소, all-reduce, MapReduce, 스트림 처리, 쿼리 계획, 스토리지, 샤딩, 압축, 분산 데이터베이스, 일관성, 신뢰성, 스케줄링, 워크플로우 엔진, 클라우드 컴퓨팅.

출시 예정일: ??? (완성될 가능성이 높음)

### 파트 IV: 소프트웨어 및 하드웨어

LLVM IR, 컴파일러 최적화 및 백엔드, 인터프리터, JIT 컴파일, Cython, JAX, Numba, Julia, OpenCL, DPC++, oneAPI, XLA, (기초) Verilog, FPGA, ASIC, TPU 및 기타 AI 가속기.

출시 예정일: ??? (완성될 가능성이 낮음)

### 감사의 말

이 책은 많은 분들이 작성한 블로그 포스트, 연구 논문, 컨퍼런스 발표 및 기타 작업물을 기반으로 합니다.

- [Agner Fog](https://agner.org/optimize/)
- [Daniel Lemire](https://lemire.me/en/#publications)
- [Andrei Alexandrescu](https://erdani.com/index.php/about/)
- [Chandler Carruth](https://twitter.com/chandlerc1024)
- [Wojciech Muła](http://0x80.pl/articles/index.html)
- [Malte Skarupke](https://probablydance.com/)
- [Travis Downs](https://travisdowns.github.io/)
- [Brendan Gregg](https://www.brendangregg.com/blog/index.html)
- [Andreas Abel](http://embedded.cs.uni-saarland.de/abel.php)
- [Jakob Kogler](https://cp-algorithms.com/)
- [Igor Ostrovsky](http://igoro.com/)
- [Steven Pigeon](https://hbfs.wordpress.com/)
- [Denis Bakhvalov](https://easyperf.net/notes/)
- [Paul Khuong](https://pvk.ca/)
- [Pat Morin](https://cglab.ca/~morin/)
- [Victor Eijkhout](https://www.tacc.utexas.edu/about/directory/victor-eijkhout)
- [Robert van de Geijn](https://www.cs.utexas.edu/~rvdg/)
- [Edmond Chow](https://www.cc.gatech.edu/~echow/)
- [Peter Cordes](https://stackoverflow.com/users/224132/peter-cordes)
- [Geoff Langdale](https://branchfree.org/)
- [Matt Kulukundis](https://twitter.com/JuvHarlequinKFM)
- [Georg Sauthoff](https://gms.tf/)
- [Danila Kutenin](https://danlark.org/author/kutdanila/)
- [Ivica Bogosavljević](https://johnysswlab.com/author/ibogi/)
- [Matt Pharr](https://pharr.org/matt/)
- [Jan Wassenberg](https://research.google/people/JanWassenberg/)
- [Marshall Lochbaum](https://mlochbaum.github.io/publications.html)
- [Pavel Zemtsov](https://pzemtsov.github.io/)
- [Gustavo Duarte](https://manybutfinite.com/)
- [Nyaan](https://nyaannyaan.github.io/library/)
- [Nayuki](https://www.nayuki.io/category/programming)
- [Konstantin](http://const.me/)
- [InstLatX64](https://twitter.com/InstLatX64)
- [ridiculous_fish](https://ridiculousfish.com/blog/)
- [Z boson](https://stackoverflow.com/users/2542702/z-boson)
- [Creel](https://www.youtube.com/c/WhatsACreel)

### 고지: 기술 선택에 대하여

이 책의 예제들은 C++, GCC, x86-64, CUDA, Spark를 사용하지만, 전달하고자 하는 기본 원칙은 이들에 국한되지 않습니다.

양심적으로 말하자면, 저는 이러한 선택들이 모두 만족스럽지는 않습니다. 단지 이 기술들이 현재 가장 널리 쓰이고 안정적이어서 독자들에게 더 도움이 되기 때문에 선택했을 뿐입니다. 가능하다면 각각 C / Rust / [Carbon?](https://github.com/carbon-language/carbon-lang), LLVM, arm, OpenCL, Dask를 선택했을 것입니다. 아마 나중에 일부 스택이 변경된 제2판이 나올 수도 있겠네요.
