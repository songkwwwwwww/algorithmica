---
title: 비트 조작 (Bit Manipulation)
weight: 7
draft: true
---

이 문서는 Sean Eron Anderson의 [Bit Twiddling Hacks](https://graphics.stanford.edu/~seander/bithacks.html)를 상당 부분 참고했습니다. 일부 방법은 추가되었고, 일부는 하드웨어에서 직접 해결되어 삭제되었습니다. 대부분은 이미 컴파일러에 의해 최적화되어 있습니다.

`cmov`를 사용할 수 있는 아키텍처에서는 많은 기법이 불필요해졌습니다.

이 문서는 또한 연습 문제로도 활용될 수 있습니다.

## 기본 연산 (Basic Operations)

`>>`

산술 시프트(arithmetic shift)는 음수의 경우 `1`을, 양수의 경우 `0`을 채워 넣습니다. (구현에 따라 다를 수 있습니다.)

C/C++에서 음수를 왼쪽이나 오른쪽으로 시프트하는 것은 정의되지 않은 동작(undefined behavior)을 유발합니다.

`<<`

"왼쪽으로 회전(rotate left)"을 수행하는 `rol` 명령어.

`__builtin_popcount` `popcnt`: x에서 1인 비트의 개수를 반환합니다.

`__builtin_parity`: x의 *패리티(parity)*를 반환합니다 (즉, 1인 비트의 개수를 2로 나눈 나머지).

이는 아마도 [오류 검출](https://en.wikipedia.org/wiki/Parity_bit)을 위한 것입니다.

`__builtin_clrsb`: x에서 선행하는 중복 부호 비트의 개수를 반환합니다. 즉, 최상위 비트와 동일한 값을 가진 바로 다음 비트들의 개수입니다. 0이나 다른 값들에 대한 특별한 경우는 없습니다.

`__builtin_ffs`: x의 최하위 1 비트의 인덱스에 1을 더한 값을 반환하며, x가 0이면 0을 반환합니다.

`__builtin_clz`: 최상위 비트 위치부터 시작하여 x에서 선행하는 0 비트의 개수를 반환합니다. x가 0이면 결과는 정의되지 않습니다.

`__builtin_ctz`: 최하위 비트 위치부터 시작하여 x에서 후행하는 0 비트의 개수를 반환합니다. x가 0이면 결과는 정의되지 않습니다.

`ctz`, `clz` -> `__lg`

## 레시피 (Recipes)

### 정수의 부호

`(x < 0)` 또는 `x >> 31`

### 두 정수가 같은 부호를 갖는지 확인

`x ^ z < 0`

### 정수의 절댓값

부호 비트 추출: `int mask = x >> 31`. 이는 음수의 경우 `1`, 양수의 경우 `0`이 됩니다.

초기 숫자와 XOR 연산: `x ^ mask` (부호에 따라 1을 더하거나 빼는 것과 같습니다).

단계 2의 결과에서 mask를 뺌: `(x ^ mask) - mask`

또는 `(v + mask) ^ mask`를 사용할 수 있는데, 이는 동일한 작업을 역순으로 수행합니다.

### 마지막 1 비트 가져오기

`x & -x`

### 정수의 마지막 1 비트 제거하기

`x & (x - 1)`

### 2의 거듭제곱인지 확인하기

`(x & (x - 1)) == 0`

0 또한 2의 거듭제곱으로 간주된다는 점에 주의하십시오.

### 비트 반전 (Reversing bits)

Clang에는 `__builtin_bitreverse{8,16,32,64}`가 있습니다.

```c++
int reverseBits(int x)
{
	unsigned int s = sizeof(x) * 8;
	T mask = ~T(0);
	while ((s >>= 1) > 0)
	{
		mask ^= mask << s;
		x = ((x >> s) & mask) | ((x << s) & ~mask);
	}
	return x;
}
```

### XOR를 이용한 숫자 교환 (Swapping)

한 번쯤 들어보셨을 기법입니다.

```c++
a ^= b;
b ^= a;
a ^= b;
```

내부적으로는 이렇게 수행되지 않습니다. 별도의 `xchng` 명령어가 존재합니다.

### 2의 거듭제곱으로 나머지 연산 (Modulus)

`m = (1 << k)`인 경우, `x % m`은 `x & (m - 1)`과 같습니다.

## 마스크 (Masks)

마스킹 연산입니다.

### 브루트 포스 (Brute forcing)

재귀를 사용할 수 있지만(당연히 느립니다), 브랜치리스(branchless) 방식으로도 가능합니다.

배낭 문제(Knapsack problem)는 $O(2^n)$의 브루트 포스 해법을 가집니다.

```c++
int ans = 0;
for (int mask = 0; mask < (1 << n); mask++) {
    int s = 0;
    for (int i = 0; i < n; i++)
        if (mask >> i & 1)
            s += a[i];
    if (s <= C)
        ans = max(ans, s);
}
```

### 모든 부분집합의 부분집합 (Subsets of all subsets)

```c++
for (int submask = mask; submask != 0; submask = (submask - 1) & mask) {
    // ...
}
```

총합은 $3^n$이 됩니다. 각 반복의 각 비트는 세 가지 상태 중 하나에 있을 수 있습니다: $m$에 없음, 아직 $s$에 없음, $s$와 $m$ 모두에 있음. 총 $n$개의 비트가 있으므로 최대 $3^n$개의 서로 다른 조합이 가능합니다.
