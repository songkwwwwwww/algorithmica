---
title: 확장 유클리드 알고리즘 (Extended Euclidean Algorithm)
weight: 3
---

[페르마의 소정리](../modular/#fermats-theorem)를 사용하면 [이진 거듭제곱(binary exponentiation)](../exponentiation/)을 통해 $O(\log n)$ 연산으로 모듈러 곱셈 역원을 계산할 수 있지만, 이는 모듈러스(modulus)가 소수일 때만 작동합니다. 이를 일반화한 [오일러의 정리(Euler's theorem)](https://en.wikipedia.org/wiki/Euler%27s_theorem)에 따르면, $m$과 $a$가 서로소일 때 다음이 성립합니다:

$$
a^{\phi(m)} \equiv 1 \pmod m
$$

여기서 $\phi(m)$은 $m$보다 작고 $m$과 서로소인 양의 정수의 개수로 정의되는 [오일러 피 함수(Euler's totient function)](https://en.wikipedia.org/wiki/Euler%27s_totient_function)입니다. $m$이 소수인 특수한 경우, $m-1$개의 모든 나머지가 서로소이므로 $\phi(m) = m - 1$이 되어 페르마의 소정리가 도출됩니다.

이를 통해 $\phi(m)$을 안다면 $a$의 역원을 $a^{\phi(m) - 1}$로 계산할 수 있습니다. 하지만 $\phi(m)$을 계산하는 것은 그리 빠르지 않습니다. 보통 이를 위해 $m$의 [인수분해](/hpc/algorithms/factorization/)가 필요하기 때문입니다. [유클리드 알고리즘](/hpc/algorithms/gcd/)을 수정하여 작동하는 더 일반적인 방법이 있습니다.

### 알고리즘

*확장 유클리드 알고리즘*은 $g = \gcd(a, b)$를 찾는 것 외에도 다음을 만족하는 정수 $x$와 $y$를 찾습니다:

$$
a \cdot x + b \cdot y = g
$$

여기서 $b$를 $m$으로, $g$를 1로 바꾸면 모듈러 역원을 찾는 문제가 됩니다:

$$
a^{-1} \cdot a + k \cdot m = 1
$$

참고로 $a$와 $m$이 서로소가 아니라면 해가 존재하지 않습니다. $a$와 $m$의 어떠한 정수 조합도 그들의 최대공약수의 배수가 아닌 값을 생성할 수 없기 때문입니다.

이 알고리즘 또한 재귀적입니다. $\gcd(b, a \bmod b)$에 대한 계수 $x'$와 $y'$를 계산하고 원래 숫자 쌍에 대한 해를 복원합니다. $(b, a \bmod b)$ 쌍에 대한 해 $(x', y')$가 있다면:

$$
b \cdot x' + (a \bmod b) \cdot y' = g
$$

초기 입력에 대한 해를 얻기 위해 $(a \bmod b)$를 $(a - \lfloor \frac{a}{b} \rfloor \cdot b)$로 다시 쓰고 위 방정식에 대입할 수 있습니다:

$$
b \cdot x' + (a - \Big \lfloor \frac{a}{b} \Big \rfloor \cdot b) \cdot y' = g
$$

이제 $a$와 $b$를 기준으로 항을 정리하면 다음과 같습니다:

$$
a \cdot \underbrace{y'}_x + b \cdot \underbrace{(x' - \Big \lfloor \frac{a}{b} \Big \rfloor \cdot y')}_y = g
$$

이를 초기 식과 비교하면, 초기 $x$와 $y$에 대해 $a$와 $b$의 계수를 그대로 사용할 수 있음을 알 수 있습니다.

### 구현

이 알고리즘을 재귀 함수로 구현합니다. 출력이 하나가 아니라 세 개의 정수이므로 계수를 참조(reference)로 전달합니다:

```c++
int gcd(int a, int b, int &x, int &y) {
    if (a == 0) {
        x = 0;
        y = 1;
        return b;
    }
    int x1, y1;
    int d = gcd(b % a, a, x1, y1);
    x = y1 - (b / a) * x1;
    y = x1;
    return d;
}
```

역원을 계산하기 위해 $a$와 $m$을 전달하고 알고리즘이 찾은 $x$ 계수를 반환하면 됩니다. 두 개의 양수를 전달하므로 계수 중 하나는 양수이고 다른 하나는 음수가 됩니다(반복 횟수가 홀수인지 짝수인지에 따라 다름). 따라서 $x$가 음수인지 확인하고 $m$을 더해 올바른 나머지를 얻어야 합니다.

```c++
int inverse(int a) {
    int x, y;
    gcd(a, M, x, y);
    if (x < 0)
        x += M;
    return x;
}
```

이 방식은 약 160ns가 소요되며, [이진 거듭제곱](../exponentiation)으로 역원을 구하는 것보다 10ns 더 빠릅니다. 이를 더욱 최적화하기 위해 반복문 형태로 바꿀 수 있으며, 이 경우 135ns가 소요됩니다:

```c++
int inverse(int a) {
    int b = M, x = 1, y = 0;
    while (a != 1) {
        y -= b / a * x;
        b %= a;
        swap(a, b);
        swap(x, y);
    }
    return x < 0 ? x + M : x;
}
```

이진 거듭제곱과 달리 실행 시간은 $a$의 값에 따라 달라집니다. 예를 들어 이 특정 $m$($10^9 + 7$) 값에 대해 최악의 입력은 564400443이며, 이 경우 알고리즘은 37번의 반복을 수행하고 250ns가 소요됩니다.

**연습 문제.** 동일한 기술을 [이진 GCD(binary GCD)](/hpc/algorithms/gcd/#binary-gcd)에 적용해 보십시오(최적화 실력이 매우 뛰어나지 않는 한 성능 향상은 크지 않을 것입니다).
