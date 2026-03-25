---
title: 사전 계산 (Precomputation)
weight: 8
---

컴파일러가 특정 변수가 사용자 제공 데이터에 의존하지 않는다고 추론할 수 있으면, 컴파일 타임에 그 값을 계산하고 생성된 머신 코드에 내장하여 상수로 바꿀 수 있습니다.

이 최적화는 성능에 큰 도움이 되지만 C++ 표준의 일부는 아니므로 컴파일러가 *반드시* 해야 하는 것은 아닙니다. 컴파일 타임 계산이 구현하기 어렵거나 시간이 많이 걸리는 경우, 컴파일러는 그 기회를 포기할 수도 있습니다.

### 상수 표현식 (Constant Expressions)

더 확실한 해결책으로, 현대 C++에서는 함수를 `constexpr`로 표시할 수 있습니다. 상수를 전달하여 호출하면 그 값은 컴파일 타임에 계산됨이 보장됩니다:

```c++
constexpr int fibonacci(int n) {
    if (n <= 2)
        return 1;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

static_assert(fibonacci(10) == 55);
```

이러한 함수들은 다른 `constexpr` 함수만 호출할 수 있고 메모리 할당을 할 수 없는 것과 같은 몇 가지 제한 사항이 있지만, 그 외에는 "있는 그대로" 실행됩니다.

`constexpr` 함수는 런타임 중에 아무런 비용도 들지 않지만 컴파일 시간은 늘어난다는 점에 유의하십시오. 따라서 효율성에 조금이라도 신경을 써야 하며 NP-완전(NP-complete) 문제를 넣지 마십시오:

```c++
constexpr int fibonacci(int n) {
    int a = 1, b = 1;
    while (n--) {
        int c = a + b;
        a = b;
        b = c;
    }
    return b;
}
```

이전 C++ 표준에서는 함수 내부에서 어떠한 상태도 사용할 수 없고 재귀에 의존해야 하는 등 훨씬 더 많은 제한이 있어서 전체 과정이 C++ 프로그래밍이라기보다 Haskell 프로그래밍처럼 느껴지기도 했습니다. C++17부터는 명령형 스타일을 사용하여 정적 배열을 계산할 수도 있으며, 이는 룩업 테이블(lookup tables)을 사전 계산하는 데 유용합니다:

```c++
struct Precalc {
    int isqrt[1000];

    constexpr Precalc() : isqrt{} {
        for (int i = 0; i < 1000; i++)
            isqrt[i] = int(sqrt(i));
    }
};

constexpr Precalc P;

static_assert(P.isqrt[42] == 6);
```

상수가 아닌 값을 전달하며 `constexpr` 함수를 호출할 때, 컴파일러가 이를 컴파일 타임에 계산할지 여부는 선택 사항입니다:

```c++
for (int i = 0; i < 100; i++)
    cout << fibonacci(i) << endl;
```

이 예제에서 기술적으로는 고정된 횟수만큼 반복하고 컴파일 타임에 알려진 파라미터로 `fibonacci`를 호출하지만, 기술적으로는 컴파일 타임 상수가 아닙니다. 이 루프를 최적화할지 여부는 컴파일러의 몫이며, 무거운 계산의 경우 대개 하지 않는 쪽을 선택합니다.

<!--

### Code Generation

There are plenty of languages that support computing *data* during compile time, but none can produce efficient code at all times.

One huge example is generating lexers and parsers: which is usually done in.

For example, CUDA and OpenCL are mostly C, and have no support for metaprogramming.

At some point (and perhaps to this day), these languages had no way to unroll loops, so people would write a [jinja template](https://jinja.palletsprojects.com/en/3.0.x/), call the thing from Python, and then compile.

It is not uncommon to use a templating engine to generate code. For example, CUDA (a GPU programming language) has no loop unrolling

-->
