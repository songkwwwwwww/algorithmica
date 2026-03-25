---
title: 벤치마킹 (Benchmarking)
weight: 6
---

대부분의 좋은 소프트웨어 공학 관행은 어떤 방식으로든 *개발 주기(development cycles)*를 단축하는 문제와 관련이 있습니다: 소프트웨어를 더 빠르게 컴파일하고(빌드 시스템), 버그를 최대한 빨리 발견하며(정적 분석, 지속적 통합), 새 버전이 준비되는 즉시 출시하고(지속적 배포), 사용자 피드백에 지체 없이 반응하는 것(애자일 개발)이 그 예입니다.

성능 엔지니어링도 다르지 않습니다. 제대로 수행한다면 다음과 같은 주기를 따라야 합니다:

1. 프로그램을 실행하고 지표를 수집합니다.
2. 병목 지점(bottleneck)이 어디인지 파악합니다.
3. 병목을 제거하고 1단계로 돌아갑니다.

이 섹션에서는 벤치마킹에 대해 알아보고, 이 주기를 단축하고 더 빠르게 반복할 수 있도록 돕는 몇 가지 실무 기술들을 논의할 것입니다. 대부분의 조언은 이 책을 집필하면서 얻은 것이므로, 이 책의 [코드 저장소](https://github.com/sslotin/ahm-code)에서 설명된 설정의 실제 예시들을 많이 찾아볼 수 있습니다.

### C++ 내부에서의 벤치마킹

벤치마킹 코드를 작성하는 방법에는 여러 가지가 있습니다. 가장 대중적인 방법은 비교하려는 동일 언어 구현체들을 한 파일에 넣고, `main` 함수에서 이들을 개별적으로 호출하여 동일한 소스 파일에서 원하는 모든 지표를 계산하는 것입니다.

이 방법의 단점은 많은 상용구 코드(boilerplate code)를 작성해야 하고 각 구현마다 이를 복제해야 한다는 것이지만, 메타프로그래밍을 통해 어느 정도 상쇄할 수 있습니다. 예를 들어, 여러 개의 [gcd](/hpc/algorithms/gcd) 구현을 벤치마킹할 때 다음과 같은 고차 함수(higher-order function)를 사용하면 벤치마킹 코드를 상당히 줄일 수 있습니다.

```c++
const int N = 1e6, T = 1e9 / N;
int a[N], b[N];

void timeit(int (*f)(int, int)) {
    clock_t start = clock();

    int checksum = 0;

    for (int t = 0; t < T; t++)
        for (int i = 0; i < n; i++)
            checksum ^= f(a[i], b[i]);
    
    float seconds = float(clock() - start) / CLOCKS_PER_SEC;

    printf("checksum: %d\n", checksum);
    printf("%.2f ns per call\n", 1e9 * seconds / N / T);
}

int main() {
    for (int i = 0; i < N; i++)
        a[i] = rand(), b[i] = rand();
    
    timeit(std::gcd);
    timeit(my_gcd);
    timeit(my_another_gcd);
    // ...

    return 0;
}
```

이 방법은 오버헤드가 매우 낮아 더 많은 실험을 수행하고 그로부터 [더 정확한 결과](../noise)를 얻을 수 있게 해줍니다. 여전히 반복적인 작업을 수행해야 하지만, 프레임워크를 통해 상당 부분 자동화할 수 있습니다. C++에서는 [Google benchmark library](https://github.com/google/benchmark)가 가장 대중적인 선택지입니다. 일부 프로그래밍 언어에는 벤치마킹을 위한 편리한 내장 도구들이 있습니다. [Python의 timeit 함수](https://docs.python.org/3/library/timeit.html)와 [Julia의 @benchmark 매크로](https://github.com/JuliaCI/BenchmarkTools.jl)가 대표적입니다.

C와 C++는 실행 속도 면에서 *효율적*이지만, 특히 분석 측면에서 가장 *생산적*인 언어는 아닙니다. 알고리즘이 입력 크기와 같은 매개변수에 의존하고 각 구현에서 하나 이상의 데이터 포인트를 수집해야 하는 경우, 벤치마킹 코드를 외부 환경과 통합하고 다른 도구를 사용하여 결과를 분석하는 것이 좋습니다.

### 구현 분리하기

모듈성과 재사용성을 높이는 한 가지 방법은 모든 테스트 및 분석 코드를 알고리즘의 실제 구현과 분리하고, 서로 다른 버전들이 별도의 파일에 구현되더라도 동일한 인터페이스를 갖도록 만드는 것입니다.

C/C++에서는 함수 인터페이스를 포함하는 단일 헤더 파일(예: `gcd.hh`)을 만들고 `main`에 모든 벤치마킹 코드를 작성하여 이를 수행할 수 있습니다.

```c++
int gcd(int a, int b); // 구현될 예정

// 데이터 구조의 경우, 설정(setup) 함수도 만들어야 합니다
// (모든 버전에 대해 동일한 전처리 단계가 충분하지 않은 경우)

int main() {
    const int N = 1e6, T = 1e9 / N;
    int a[N], b[N];
    // 주의: 지역 배열은 스택에 할당되므로 스택 오버플로우를 일으킬 수 있습니다
    // 큰 배열의 경우 "new"로 할당하거나 전역 배열을 생성하세요

    for (int i = 0; i < N; i++)
        a[i] = rand(), b[i] = rand();

    int checksum = 0;

    clock_t start = clock();

    for (int t = 0; t < T; t++)
        for (int i = 0; i < n; i++)
            checksum += gcd(a[i], b[i]);
    
    float seconds = float(clock() - start) / CLOCKS_PER_SEC;

    printf("%d\n", checksum);
    printf("%.2f ns per call\n", 1e9 * seconds / N / T);
    
    return 0;
}
```

그런 다음 각 알고리즘 버전마다 여러 구현 파일(예: `v1.cc`, `v2.cc` 등, 또는 의미 있는 이름)을 만들고 모두 해당 단일 헤더 파일을 포함하도록 합니다.

```c++
#include "gcd.hh"

int gcd(int a, int b) {
    if (b == 0)
        return a;
    else
        return gcd(b, a % b);
}
```

이렇게 하는 전체적인 목적은 소스 코드 파일을 건드리지 않고도 명령줄에서 특정 알고리즘 버전을 벤치마킹할 수 있도록 하는 것입니다. 이를 위해 명령줄 인자에서 파싱하는 등의 방법으로 매개변수를 노출할 수도 있습니다.

```c++
int main(int argc, char* argv[]) {
    int N = (argc > 1 ? atoi(argv[1]) : 1e6);
    const int T = 1e9 / N;

    // ...
}
```

또 다른 방법은 C 스타일의 전역 정의(global define)를 사용하고 컴파일 중에 `-D N=...` 플래그로 전달하는 것입니다.

```c++
#ifndef N
#define N 1000000
#endif

const int T = 1e9 / N;
```

이 방식을 사용하면 컴파일 타임 상수를 활용할 수 있어 일부 알고리즘의 성능에 매우 유리할 수 있습니다. 다만 매개변수를 변경할 때마다 프로그램을 다시 빌드해야 하므로, 다양한 매개변수 값에 걸쳐 지표를 수집하는 데 필요한 시간이 상당히 늘어날 수 있습니다.

### Makefile

<!-- TODO -->

소스 파일을 분리하면 [Make](https://en.wikipedia.org/wiki/Make_(software))와 같은 캐싱 빌드 시스템을 사용하여 컴파일 속도를 높일 수 있습니다.

저는 보통 프로젝트마다 다음과 같은 버전의 Makefile을 사용합니다.

```c++
compile = g++ -std=c++17 -O3 -march=native -Wall

%: %.cc gcd.hh
	$(compile) $< -o $@ 

%.s: %.cc gcd.hh
	$(compile) -S -fverbose-asm $< -o $@

%.run: %
	@./$<

.PHONY: %.run
```

이제 `make example`로 `example.cc`를 컴파일하고, `make example.run`으로 자동으로 실행할 수 있습니다.

또한 Makefile에 통계 계산을 위한 스크립트를 추가하거나, 프로파일링을 자동화하기 위해 `perf stat` 호출과 통합할 수도 있습니다.

### Jupyter Notebook

상위 수준의 분석 속도를 높이기 위해, 모든 스크립트를 넣고 그래프를 그리는 Jupyter 노트북을 만들 수 있습니다.

스칼라 결과를 반환하는 벤치마킹용 래퍼(wrapper)를 추가하면 편리합니다.

```python
def bench(source, n=2**20):
    !make -s {source}
    if _exit_code != 0:
        raise Exception("Compilation failed")
    res = !./{source} {n} {q}
    duration = float(res[0].split()[0])
    return duration
```

그런 다음 이를 사용하여 깔끔한 분석 코드를 작성할 수 있습니다.

```python
ns = list(int(1.17**k) for k in range(30, 60))
baseline = [bench('std_lower_bound', n=n) for n in ns]
results = [bench('my_binary_search', n=n) for n in ns]

# 다양한 배열 크기에 대한 상대적 속도 향상 그래프 그리기
import matplotlib.pyplot as plt

plt.plot(ns, [x / y for x, y in zip(baseline, results)])
plt.show()
```

일단 구축되면, 이 워크플로우는 반복 주기를 훨씬 빠르게 만들고 알고리즘 자체를 최적화하는 데 집중할 수 있게 해줍니다.
