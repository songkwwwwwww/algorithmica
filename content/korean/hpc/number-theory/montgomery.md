---
title: 몽고메리 곱셈 (Montgomery Multiplication)
weight: 4
---

놀랍지 않게도, [모듈러 산술](../modular)에서 계산의 상당 부분은 모듈로(modulo) 연산에 소비됩니다. 이 연산은 [일반 정수 나눗셈](/hpc/arithmetic/division/)만큼 느리며, 피연산자의 크기에 따라 보통 15~20 사이클이 소요됩니다.

이러한 번거로움을 해결하는 가장 좋은 방법은 모듈로 연산을 완전히 피하거나, 지연시키거나, [조건부 실행(predication)](/hpc/pipelining/branchless)으로 대체하는 것입니다. 예를 들어 모듈러 합계를 계산할 때 다음과 같이 할 수 있습니다.

```cpp
const int M = 1e9 + 7;

// 입력: [0, M) 범위의 n개 정수 배열
// 출력: M에 대한 모듈러 합
int slow_sum(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++)
        s = (s + a[i]) % M;
    return s;
}

int fast_sum(int *a, int n) {
    int s = 0;
    for (int i = 0; i < n; i++) {
        s += a[i]; // s < 2 * M
        s = (s >= M ? s - M : s); // cmov로 대체됨
    }
    return s;
}

int faster_sum(int *a, int n) {
    long long s = 0; // 오버플로 처리를 위한 64비트 정수
    for (int i = 0; i < n; i++)
        s += a[i]; // 벡터화됨
    return s % M;
}
```

하지만 가끔은 모듈러 곱셈이 연쇄적으로 일어나는 경우도 있는데, 이때는 상수 모듈로와 약간의 사전 계산이 필요한 [정수 나눗셈 트릭](../hpc/arithmetic/division/) 외에는 나눗셈의 나머지를 계산하는 과정을 피해갈 좋은 방법이 없습니다.

하지만 모듈러 산술을 위해 특별히 설계된 또 다른 기법이 있는데, 바로 *몽고메리 곱셈(Montgomery multiplication)*입니다.

### 몽고메리 공간 (Montgomery Space)

몽고메리 곱셈은 먼저 곱해지는 수들을 모듈러 곱셈을 저렴하게 수행할 수 있는 *몽고메리 공간*으로 변환하고, 실제 값이 필요할 때 다시 되돌리는 방식으로 작동합니다. 일반적인 정수 나눗셈 방법과 달리, 몽고메리 곱셈은 단 한 번의 모듈러 리덕션(reduction)을 수행하는 데는 효율적이지 않으며, 모듈러 연산이 연쇄적으로 일어날 때만 가치가 있습니다.

이 공간은 모듈로 $n$과 $n$과 서로소인 양의 정수 $r \ge n$에 의해 정의됩니다. 이 알고리즘은 $r$에 대한 모듈로 및 나눗셈을 포함하므로, 실제로는 $r$을 $2^{32}$ 또는 $2^{64}$로 선택하여 이러한 연산을 각각 우측 시프트(right-shift)와 비트 AND 연산으로 수행할 수 있도록 합니다.

**정의.** 몽고메리 공간에서 숫자 $x$의 *대표값(representative)* $\bar x$는 다음과 같이 정의됩니다.

$$
\bar{x} = x \cdot r \bmod n
$$

이 변환을 계산하려면 곱셈과 모듈로 연산이 필요한데, 이는 우리가 처음에 최적화하려고 했던 값비싼 연산입니다. 따라서 몽고메리 공간으로 숫자를 변환하고 다시 되돌리는 오버헤드가 그럴만한 가치가 있을 때만 이 방법을 사용하며, 일반적인 모듈러 곱셈에는 사용하지 않습니다.

몽고메리 공간 내부에서 덧셈, 뺄셈, 그리고 같음 여부 확인은 평소와 같이 수행됩니다.

$$
x \cdot r + y \cdot r \equiv (x + y) \cdot r \bmod n
$$

하지만 곱셈의 경우는 다릅니다. 몽고메리 공간에서의 곱셈을 $*$로, "일반적인" 곱셈을 $\cdot$으로 표시하면 결과는 다음과 같아야 합니다.

$$
\bar{x} * \bar{y} = \overline{x \cdot y} = (x \cdot y) \cdot r \bmod n
$$

하지만 몽고메리 공간에서의 일반적인 곱셈 결과는 다음과 같습니다.

$$
\bar{x} \cdot \bar{y} = (x \cdot y) \cdot r \cdot r \bmod n
$$

따라서 몽고메리 공간에서의 곱셈은 다음과 같이 정의됩니다.

$$
\bar{x} * \bar{y} = \bar{x} \cdot \bar{y} \cdot r^{-1} \bmod n
$$

이는 몽고메리 공간에서 두 수를 평범하게 곱한 후, 그 결과에 $r^{-1}$을 곱하고 모듈로 연산을 수행하여 *리덕션(reduce)*해야 함을 의미합니다. 그리고 이 특정 연산을 수행하는 효율적인 방법이 존재합니다.

### 몽고메리 리덕션 (Montgomery reduction)

$r=2^{32}$이고, 모듈로 $n$은 32비트이며, 리덕션해야 할 숫자 $x$가 64비트(두 32비트 숫자의 곱)라고 가정해 봅시다. 우리의 목표는 $y = x \cdot r^{-1} \bmod n$을 계산하는 것입니다.

$r$이 $n$과 서로소이므로, $[0, n)$ 범위에 다음을 만족하는 두 숫자 $r^{-1}$과 $n^\prime$이 존재한다는 것을 알 수 있습니다.

$$
r \cdot r^{-1} + n \cdot n^\prime = 1
$$

그리고 $r^{-1}$과 $n^\prime$은 모두 [확장 유클리드 알고리즘](../euclid-extended) 등을 사용하여 계산할 수 있습니다.

이 항등식을 사용하여 $r \cdot r^{-1}$을 $(1 - n \cdot n^\prime)$으로 표현하고 $x \cdot r^{-1}$을 다음과 같이 쓸 수 있습니다.

$$
\begin{aligned}
x \cdot r^{-1} &= x \cdot r \cdot r^{-1} / r
\\             &= x \cdot (1 - n \cdot n^{\prime}) / r
\\             &= (x - x \cdot n \cdot n^{\prime}    ) / r
\\             &\equiv (x - x \cdot n \cdot n^{\prime} + k \cdot r \cdot n) / r &\pmod n &\;\;\text{(임의의 정수 $k$에 대해)}
\\             &\equiv (x - (x \cdot n^{\prime} - k \cdot r) \cdot n) / r &\pmod n
\end{aligned}
$$

이제 $k$를 $\lfloor x \cdot n^\prime / r \rfloor$($x \cdot n^\prime$ 곱의 상위 64비트)로 선택하면 소거될 것이며, $(k \cdot r - x \cdot n^{\prime})$은 단순히 $x \cdot n^{\prime} \bmod r$($x \cdot n^\prime$의 하위 32비트)과 같아집니다. 이는 다음을 의미합니다.

$$
x \cdot r^{-1} \equiv (x - x \cdot n^{\prime} \bmod r \cdot n) / r
$$

알고리즘 자체는 단순히 이 공식을 평가하여 $q = x \cdot n^{\prime} \bmod r$과 $m = q \cdot n$을 계산하기 위한 두 번의 곱셈을 수행한 다음, $x$에서 이를 빼고 결과를 우측 시프트하여 $r$로 나눕니다.

마지막으로 처리해야 할 일은 결과가 $[0, n)$ 범위에 있지 않을 수 있다는 점입니다. 하지만 다음과 같으므로

$$
x < n \cdot n < r \cdot n \implies x / r < n
$$

그리고

$$
m = q \cdot n < r \cdot n \implies m / r < n
$$

다음이 보장됩니다.

$$
-n < (x - m) / r < n
$$

따라서 결과가 음수인지 확인하고 그런 경우 $n$을 더해주기만 하면 되며, 다음과 같은 알고리즘이 도출됩니다.

```c++
typedef __uint32_t u32;
typedef __uint64_t u64;

const u32 n = 1e9 + 7, nr = inverse(n, 1ull << 32);

u32 reduce(u64 x) {
    u32 q = u32(x) * nr;      // q = x * n' mod r
    u64 m = (u64) q * n;      // m = q * n
    u32 y = (x - m) >> 32;    // y = (x - m) / r
    return x < m ? y + n : y; // y < 0이면 n을 더해 [0, n) 범위에 있게 함
}
```

이 마지막 확인 작업은 비교적 저렴하지만 여전히 크리티컬 패스(critical path)에 있습니다. 결과가 $[0, n)$ 대신 $[0, 2 \cdot n - 2]$ 범위에 있어도 괜찮다면, 이를 제거하고 결과에 조건 없이 $n$을 더할 수 있습니다.

```c++
u32 reduce(u64 x) {
    u32 q = u32(x) * nr;
    u64 m = (u64) q * n;
    u32 y = (x - m) >> 32;
    return y + n
}
```

또한 계산 그래프에서 `>> 32` 연산을 한 단계 앞당겨 $(x - m) / r$ 대신 $\lfloor x / r \rfloor - \lfloor m / r \rfloor$를 계산할 수도 있습니다. 이는 어차피 $x$와 $m$의 하위 32비트가 다음과 같이 동일하기 때문에 올바른 계산입니다.

$$
m = x \cdot n^\prime \cdot n \equiv x \pmod r
$$

하지만 왜 굳이 하나 대신 두 번의 우측 시프트를 수행하는 것을 선택할까요? 이는 `((u64) q * n) >> 32`의 경우 32x32 곱셈을 수행하고 결과의 상위 32비트를 취해야 하는데(x86 `mul` 명령은 [이미 별도의 레지스터에 이를 기록하므로](../hpc/arithmetic/integer/#128-bit-integers) 비용이 들지 않음), 다른 우측 시프트인 `x >> 32`는 크리티컬 패스에 있지 않기 때문에 이점이 있습니다.

```c++
u32 reduce(u64 x) {
    u32 q = u32(x) * nr;
    u32 m = ((u64) q * n) >> 32;
    return (x >> 32) + n - m;
}
```

다른 모듈러 리덕션 방법과 비교했을 때 몽고메리 곱셈의 주요 장점 중 하나는 매우 큰 데이터 타입이 필요하지 않다는 것입니다. 결과의 하위 및 상위 $r$ 비트를 추출하는 $r \times r$ 곱셈만 있으면 되며, 이는 대부분의 하드웨어에서 [특별한 지원](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=7395,7392,7269,4868,7269,7269,1820,1835,6385,5051,4909,4918,5051,7269,6423,7410,150,2138,1829,1944,3009,1029,7077,519,5183,4462,4490,1944,5055,5012,5055&techs=AVX,AVX2&text=mul)을 받기 때문에 [SIMD](../hpc/simd/) 및 더 큰 데이터 타입으로도 쉽게 일반화할 수 있습니다.

```c++
typedef __uint128_t u128;

u64 reduce(u128 x) const {
    u64 q = u64(x) * nr;
    u64 m = ((u128) q * n) >> 64;
    return (x >> 64) + n - m;
}
```

일반적인 정수 나눗셈 트릭으로는 128비트를 64비트로 나누는 모듈로 연산이 불가능하다는 점에 유의하십시오. 컴파일러는 이를 지원하기 위해 느린 [긴 산술 라이브러리 함수(long arithmetic library function)](https://github.com/llvm-mirror/compiler-rt/blob/69445f095c22aac2388f939bedebf224a6efcdaf/lib/builtins/udivmodti4.c#L22)를 호출하도록 [되돌아갑니다(falls back)](https://godbolt.org/z/fbEE4v4qr).

### 더 빠른 역원 및 변환 (Faster Inverse and Transform)

몽고메리 곱셈 자체는 빠르지만, 약간의 사전 계산이 필요합니다.

- $n^\prime$을 계산하기 위해 법 $r$에 대한 $n$의 역원을 구함,
- 숫자를 몽고메리 공간으로 변환,
- 숫자를 몽고메리 공간에서 되돌림.

마지막 연산은 우리가 방금 구현한 `reduce` 프로시저로 이미 효율적으로 수행되지만, 처음 두 연산은 약간 최적화될 수 있습니다.

**역원 계산** $n^\prime = n^{-1} \bmod r$은 $r$이 2의 거듭제곱이라는 점을 이용하고 다음 항등식을 사용하면 확장 유클리드 알고리즘보다 더 빠르게 수행될 수 있습니다.

$$
a \cdot x \equiv 1 \bmod 2^k
\implies
a \cdot x \cdot (2 - a \cdot x)
\equiv
1 \bmod 2^{2k}
$$

증명:

$$
\begin{aligned}
a \cdot x \cdot (2 - a \cdot x)
   &= 2 \cdot a \cdot x - (a \cdot x)^2
\\ &= 2 \cdot (1 + m \cdot 2^k) - (1 + m \cdot 2^k)^2
\\ &= 2 + 2 \cdot m \cdot 2^k - 1 - 2 \cdot m \cdot 2^k - m^2 \cdot 2^{2k}
\\ &= 1 - m^2 \cdot 2^{2k}
\\ &\equiv 1 \bmod 2^{2k}.
\end{aligned}
$$

$2^1$에 대한 $a$의 역원인 $x = 1$로 시작하여 이 항등식을 정확히 $\log_2 r$번 적용할 수 있으며, 매번 역원의 비트 수를 두 배로 늘립니다. 이는 어느 정도 [뉴턴 방법(Newton's method)](../hpc/arithmetic/newton/)을 연상시킵니다.

숫자를 몽고메리 공간으로 **변환**하는 것은 숫자에 $r$을 곱하고 [통상적인 방법](../hpc/arithmetic/division/)으로 모듈로를 계산하여 수행할 수 있지만, 다음 관계를 이용할 수도 있습니다.

$$
\bar{x} = x \cdot r \bmod n = x * r^2
$$

공간으로 숫자를 변환하는 것은 단순히 $r^2$을 곱하는 것과 같습니다. 따라서 $r^2 \bmod n$을 미리 계산해 두고 대신 곱셈과 리덕션을 수행할 수 있습니다. 하지만 숫자와 $r=2^k$를 곱하는 것은 좌측 시프트로 구현할 수 있는 반면 $r^2 \bmod n$과의 곱셈은 불가능하기 때문에 이것이 실제로 더 빠를 수도 있고 아닐 수도 있습니다.

### 전체 구현 (Complete Implementation)

모든 것을 단일 `constexpr` 구조체로 묶는 것이 편리합니다.

```c++
struct Montgomery {
    u32 n, nr;
    
    constexpr Montgomery(u32 n) : n(n), nr(1) {
        // log(2^32) = 5
        for (int i = 0; i < 5; i++)
            nr *= 2 - n * nr;
    }

    u32 reduce(u64 x) const {
        u32 q = u32(x) * nr;
        u32 m = ((u64) q * n) >> 32;
        return (x >> 32) + n - m;
        // [0, 2 * n - 2] 범위의 숫자를 반환함
        // (제대로 된 모듈로 결과가 필요하다면 "x < n ? x : x - n" 형태의 확인을 추가하십시오)
    }

    u32 multiply(u32 x, u32 y) const {
        return reduce((u64) x * y);
    }

    u32 transform(u32 x) const {
        return (u64(x) << 32) % n;
        // multiply(x, r^2 mod n)으로도 구현 가능
    }
};
```

성능을 테스트하기 위해 몽고메리 곱셈을 [이진 거듭제곱(binary exponentiation)](../hpc/number-theory/exponentiation/)에 적용해 볼 수 있습니다.

```c++
constexpr Montgomery space(M);

int inverse(int _a) {
    u64 a = space.transform(_a);
    u64 r = space.transform(1);
    
    #pragma GCC unroll(30)
    for (int l = 0; l < 30; l++) {
        if ( (M - 2) >> l & 1 )
            r = space.multiply(r, a);
        a = space.multiply(a, a);
    }

    return space.reduce(r);
}
```

컴파일러가 생성한 빠른 모듈로 트릭을 사용하는 일반적인 이진 거듭제곱은 `inverse` 호출당 약 170ns가 소요되는 반면, 이 구현은 약 166ns가 소요되며 `transform`과 `reduce`를 생략하면 158ns까지 내려갑니다(`inverse`가 더 큰 모듈러 계산의 서브 프로시저로 사용되는 경우 합리적인 사용 사례입니다). 이는 작은 개선이지만, 몽고메리 곱셈은 SIMD 응용 프로그램 및 더 큰 데이터 타입에서 훨씬 더 유리해집니다.

**연습 문제.** 효율적인 *모듈러* [행렬 곱셈](/hpc/algorithms/matmul)을 구현해 보십시오.
