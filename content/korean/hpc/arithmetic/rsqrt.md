---
title: 고속 역 제곱근 (Fast Inverse Square Root)
weight: 4
---

부동 소수점 수의 역 제곱근 $\frac{1}{\sqrt x}$은 정규화된 벡터를 계산하는 데 사용되며, 이는 다시 컴퓨터 그래픽스(예: 조명을 시뮬레이션하기 위한 입사각과 반사각 결정)와 같은 다양한 시뮬레이션 시나리오에서 광범위하게 사용됩니다.

$$
\hat{v} = \frac{\vec v}{\sqrt {v_x^2 + v_y^2 + v_z^2}}
$$

먼저 제곱근을 계산한 다음 1을 그 값으로 나누어 역 제곱근을 직접 계산하는 방식은 매우 느립니다. 두 연산 모두 하드웨어로 구현되어 있음에도 불구하고 느리기 때문입니다.

하지만 부동 소수점 수가 메모리에 저장되는 방식을 활용하는 놀랍도록 훌륭한 근사 알고리즘이 있습니다. 사실 이 알고리즘은 너무나 훌륭해서 [하드웨어로 직접 구현](https://www.felixcloutier.com/x86/rsqrtps)되었을 정도입니다. 따라서 소프트웨어 엔지니어들에게 이 알고리즘 자체가 더 이상 실용적으로 중요하지는 않지만, 그 고유한 아름다움과 교육적 가치 때문에 살펴보려 합니다.

이 방법 자체만큼이나 흥미로운 것은 그 탄생 배경입니다. 이 알고리즘은 게임 스튜디오인 *id Software*가 1999년의 상징적인 게임인 *퀘이크 III 아레나(Quake III Arena)*에서 사용하여 유명해졌습니다. 하지만 실제로는 "누구한테 배운 사람한테 배운 사람" 식의 계보를 거슬러 올라가면 William Kahan(IEEE 754와 카한 합산 알고리즘을 만든 바로 그분)에 닿는 것으로 보입니다.

이 알고리즘은 2005년경 게임의 소스 코드가 공개되면서 게임 개발 커뮤니티에서 유명해졌습니다. 다음은 주석을 포함한 [관련 발췌문](https://github.com/id-Software/Quake-III-Arena/blob/master/code/game/q_math.c#L552)입니다.

```c++
float Q_rsqrt(float number) {
    long i;
    float x2, y;
    const float threehalfs = 1.5F;

    x2 = number * 0.5F;
    y  = number;
    i  = * ( long * ) &y;                       // 사악한 부동 소수점 비트 수준 해킹
    i  = 0x5f3759df - ( i >> 1 );               // 이게 도대체 뭐야? (what the fuck?) 
    y  = * ( float * ) &i;
    y  = y * ( threehalfs - ( x2 * y * y ) );   // 첫 번째 반복
//  y  = y * ( threehalfs - ( x2 * y * y ) );   // 두 번째 반복, 생략 가능

    return y;
}
```

이 코드가 무엇을 하는지 단계별로 살펴보겠지만, 그 전에 잠시 짚고 넘어갈 것이 있습니다.

### 근사 로그 (Approximate Logarithm)

컴퓨터(또는 저렴한 계산기)가 일상화되기 전에는 로그표를 사용하여 곱셈과 관련 연산을 수행했습니다. $a$와 $b$의 로그값을 찾아 더한 다음, 결과의 역로그(inverse logarithm)를 찾는 방식이었습니다.

$$
a \times b = 10^{\log a + \log b} = \log^{-1}(\log a + \log b)
$$

다음 항등식을 사용하여 $\frac{1}{\sqrt x}$를 계산할 때도 동일한 트릭을 사용할 수 있습니다.

$$
\log \frac{1}{\sqrt x} = - \frac{1}{2} \log x
$$

고속 역 제곱근은 이 항등식에 기반하며, 따라서 $x$의 로그값을 매우 빠르게 계산해야 합니다. 알고리즘의 핵심은 32비트 `float`을 정수로 재해석(reinterpreting)하기만 해도 로그값을 근사할 수 있다는 점입니다.

[부동 소수점](../float)은 부호 비트(양수인 경우 0), 지수 $e_x$, 가수 $m_x$를 순차적으로 저장하며, 이는 다음과 같은 값에 대응한다는 점을 기억하십시오.

$$
x = 2^{e_x} \cdot (1 + m_x)
$$

따라서 로그값은 다음과 같습니다.

$$
\log_2 x = e_x + \log_2 (1 + m_x)
$$

$m_x \in [0, 1)$이므로, 우변의 로그는 다음과 같이 근사될 수 있습니다.

$$
\log_2 (1 + m_x) \approx m_x
$$

이 근사는 구간의 양 끝점에서 정확하지만, 평균적인 경우를 고려하기 위해 작은 상수 $\sigma$만큼 이동시켜야 합니다. 따라서 다음과 같습니다.

$$
\log_2 x = e_x + \log_2 (1 + m_x) \approx e_x + m_x + \sigma
$$

이제 이 근사를 염두에 두고 $L=2^{23}$(`float`의 가수 비트 수)과 $B=127$(지수 바이어스)을 정의할 때, $x$의 비트 패턴을 정수 $I_x$로 재해석하면 다음과 같은 결과를 얻습니다.

$$
\begin{aligned}
I_x &= L \cdot (e_x + B + m_x)
\\  &= L \cdot (e_x + m_x + \sigma + B - \sigma )
\\  &\approx L \cdot \log_2 (x) + L \cdot (B - \sigma )
\end{aligned}
$$

(정수에 $L=2^{23}$을 곱하는 것은 왼쪽으로 23비트 시프트하는 것과 동일합니다.)

평균 제곱 오차(mean square error)를 최소화하도록 $\sigma$를 조정하면 놀랍도록 정확한 근사치가 나옵니다.

![부동 소수점 수 $x$를 정수(파란색)로 재해석한 것과 스케일링 및 이동된 로그(회색)의 비교](../img/approx.svg)

이제 근사식으로부터 로그를 표현하면 다음과 같습니다.

$$
\log_2 x \approx \frac{I_x}{L} - (B - \sigma)
$$

좋습니다. 이제 어디까지 했었죠? 아, 역 제곱근을 계산하려고 했었죠.

### 결과 근사 (Approximating the Result)

$\log_2 y = - \frac{1}{2} \log_2 x$ 항등식을 사용하여 $y = \frac{1}{\sqrt x}$를 계산하기 위해, 이를 근사 공식에 대입하면 다음과 같습니다.

$$
\frac{I_y}{L} - (B - \sigma)
\approx
- \frac{1}{2} ( \frac{I_x}{L} - (B - \sigma) )
$$

$I_y$에 대해 풀면 다음과 같습니다.

$$
I_y \approx \frac{3}{2} L (B - \sigma) - \frac{1}{2} I_x
$$

놀랍게도 로그를 계산할 필요조차 없습니다. 위의 공식은 단지 어떤 상수에서 $x$의 정수 재해석 결과의 절반을 뺀 것일 뿐입니다. 코드에서는 다음과 같이 작성되었습니다.

```cpp
i = * ( long * ) &y;
i = 0x5f3759df - ( i >> 1 );
```

첫 번째 줄에서 `y`를 정수로 재해석하고, 두 번째 줄에서 이를 공식에 대입합니다. 첫 번째 항인 마법의 숫자 $\frac{3}{2} L (B - \sigma) = \mathtt{0x5F3759DF}$가 쓰였고, 두 번째 항은 나눗셈 대신 이진 시프트를 사용하여 계산되었습니다.

### 뉴턴 방법으로 반복 (Iterating with Newton's Method)

그다음으로 이어지는 것은 $f(y) = \frac{1}{y^2} - x$와 매우 훌륭한 초기값을 사용한 몇 번의 뉴턴 방법 반복 코드입니다. 업데이트 규칙은 다음과 같습니다.

$$
f'(y) = - \frac{2}{y^3} \implies y_{i+1} = y_{i} (\frac{3}{2} - \frac{x}{2} y_i^2) = \frac{y_i (3 - x y_i^2)}{2}
$$

코드에서는 다음과 같이 작성되었습니다.

```cpp
x2 = number * 0.5F;
y  = y * ( threehalfs - ( x2 * y * y ) );
```

초기 근사값이 워낙 훌륭해서 게임 개발 목적에는 단 한 번의 반복으로도 충분했습니다. 첫 번째 반복만으로도 정답의 99.8% 이내에 수렴하며, 정확도를 더 높이기 위해 추가로 반복할 수도 있습니다. 현재 하드웨어에서 수행하는 방식이 바로 이것입니다. [x86 명령어](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#ig_expand=3037,3009,5135,4870,4870,4872,4875,833,879,874,849,848,6715,4845,6046,3853,288,6570,6527,6527,90,7307,6385,5993&text=rsqrt&techs=AVX,AVX2)는 몇 번의 반복을 수행하여 $1.5 \times 2^{-12}$ 이하의 상대 오차를 보장합니다.

### 더 읽어볼 거리 (Further Reading)

[고속 역 제곱근에 관한 위키피디아 문서](https://en.wikipedia.org/wiki/Fast_inverse_square_root#Floating-point_representation).
