---
title: 상황별 최적화 (Situational Optimizations)
weight: 3
---

<!--

Generally, you always want to specify the exact platform you are running and turn on `-O3`, but other optimizations, like the ones discussed [in the previous section](../assembly), are far more situational and require some input from the programmer.

-->

`-O2` 및 `-O3`에서 활성화되는 대부분의 컴파일러 최적화는 성능을 향상시키거나 적어도 심각하게 저하시키지 않음이 보장됩니다. `-O3`에 포함되지 않은 것들은 엄격하게 표준을 준수하지 않거나, 상황에 따라 크게 달라져서 사용이 유익한지 결정하기 위해 프로그래머의 추가적인 입력이 필요한 것들입니다.

우리가 이 책에서 이전에 다루었던 가장 자주 사용되는 것들에 대해 논의해 봅시다.

### 루프 언롤링 (Loop Unrolling)

[루프 언롤링](/hpc/architecture/loops#loop-unrolling)은 컴파일 타임에 알려진 작은 고정 횟수만큼 반복되는 루프가 아니라면 기본적으로 비활성화되어 있습니다. 고정 횟수 반복의 경우 점프가 전혀 없는 반복된 명령어 시퀀스로 대체됩니다. `-funroll-loops` 플래그를 사용하면 전역적으로 활성화할 수 있으며, 이는 반복 횟수를 컴파일 타임에 결정할 수 있거나 루프 진입 시 알 수 있는 모든 루프를 언롤링합니다.

프라그마를 사용하여 특정 루프를 타겟팅할 수도 있습니다:

```c++
#pragma GCC unroll 4
for (int i = 0; i < n; i++) {
    // ...
}
```

루프 언롤링은 바이너리 크기를 키우며, 실행 속도가 빨라질 수도 있고 그렇지 않을 수도 있습니다. 광적으로 사용하지 마십시오.

### 함수 인라이닝 (Function Inlining)

[인라이닝](/hpc/architecture/functions#inlining)은 컴파일러가 결정하도록 맡기는 것이 가장 좋지만, `inline` 키워드로 영향을 줄 수 있습니다:

```c++
inline int square(int x) {
    return x * x;
}
```

하지만 컴파일러가 잠재적인 성능 향상이 그만한 가치가 없다고 판단하면 이 힌트는 무시될 수 있습니다. `always_inline` 속성을 추가하여 인라이닝을 강제할 수 있습니다:

```c++
#define FORCE_INLINE inline __attribute__((always_inline))
```

또한 인라이닝된 함수의 크기(명령어 수 기준)에 대해 특정 임계값을 설정할 수 있는 `-finline-limit=n` 옵션도 있습니다. Clang의 상응하는 옵션은 `-inline-threshold`입니다.

### 분기 가능성 (Likeliness of Branches)

[분기 가능성](/hpc/architecture/layout#unequal-branches)은 `if` 및 `switch`에서 `[[likely]]` 및 `[[unlikely]]` 속성을 통해 힌트를 줄 수 있습니다:

```c++
int factorial(int n) {
    if (n > 1) [[likely]]
        return n * factorial(n - 1);
    else [[unlikely]]
        return 1;
}
```

이것은 C++20에서 처음 등장한 새로운 기능입니다. 그전에는 조건식을 감싸는 데 유사하게 사용되는 컴파일러 전용 인트린직(intrinsics)이 있었습니다. 이전 GCC에서의 동일한 예시입니다:

```c++
int factorial(int n) {
    if (__builtin_expect(n > 1, 1))
        return n * factorial(n - 1);
    else
        return 1;
}
```

<!--
What it usually does is it swaps the branches so that the more likely one goes immediately after jump (recall that "don't jump" branch is taken by default). The performance gain is usually rather small, because for most hot spots hardware branch prediction works just fine.
-->

나중에 더 관련성이 높아지면 다루겠지만, 이와 같이 컴파일러를 올바른 방향으로 안내해야 하는 다른 경우들이 많이 있습니다.

### 프로파일 가이드 최적화 (Profile-Guided Optimization)

소스 코드에 이 모든 메타데이터를 추가하는 것은 지루한 일입니다. 사람들은 이미 그런 일 없이도 C++ 작성을 싫어합니다.

또한 특정 최적화가 유익한지 여부가 항상 명확한 것은 아닙니다. 분기 재정렬, 함수 인라이닝 또는 루프 언롤링에 대한 결정을 내리려면 다음과 같은 질문에 대한 답이 필요합니다:

- 이 분기가 얼마나 자주 실행되는가?
- 이 함수가 얼마나 자주 호출되는가?
- 이 루프의 평균 반복 횟수는 얼마인가?

다행히도 이러한 실제 정보를 자동으로 제공하는 방법이 있습니다.

*프로파일 가이드 최적화*(PGO, 발음하기 더 쉽고 재미있어서 "포고"라고도 불림)는 정적 분석만으로는 달성할 수 없는 성능 향상을 위해 [프로파일링 데이터](/hpc/profiling)를 사용하는 기법입니다. 요약하자면, 프로그램의 관심 지점에 타이머와 카운터를 추가하고, 실제 데이터로 컴파일 및 실행한 다음, 테스트 실행에서 얻은 추가 정보를 제공하여 다시 컴파일하는 과정을 포함합니다.

현대 컴파일러에서는 이 전체 과정이 자동화되어 있습니다. 예를 들어, `-fprofile-generate` 플래그를 사용하면 GCC가 프로파일링 코드로 프로그램을 계측하도록 합니다:

```
g++ -fprofile-generate [기타 플래그] source.cc -o binary
```

실제 사용 사례를 최대한 대표하는 입력으로 프로그램을 실행하면 테스트 실행에 대한 로그 데이터가 포함된 여러 `*.gcda` 파일이 생성됩니다. 그 후 `-fprofile-use` 플래그를 추가하여 프로그램을 다시 빌드할 수 있습니다:

```
g++ -fprofile-use [기타 플래그] source.cc -o binary
```

이는 일반적으로 대규모 코드베이스에서 성능을 10-20% 향상시키며, 이러한 이유로 성능이 중요한 프로젝트의 빌드 프로세스에 흔히 포함됩니다. 이는 견고한 벤치마킹 코드에 투자해야 할 또 다른 이유이기도 합니다.

<!--

We will study how profiling works more deeply in the [next chapter](../../profiling).

-->
