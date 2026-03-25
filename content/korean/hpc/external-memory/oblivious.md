---
title: 캐시 비인지적 알고리즘 (Cache-Oblivious Algorithms)
weight: 7
---

[외부 메모리 모델](../model)의 맥락에서 두 가지 유형의 효율적인 알고리즘이 있습니다:

- *캐시 인지적(Cache-aware)* 알고리즘: *알려진* $B$와 $M$에 대해 효율적입니다.
- *캐시 비인지적(Cache-oblivious)* 알고리즘: *어떠한* $B$와 $M$에 대해서도 효율적입니다.

예를 들어, [외부 병합 정렬](../sorting)은 캐시 인지적이지만 캐시 비인지적 알고리즘은 아닙니다. $k$-way 병합 정렬을 수행하기 위해 적절한 $k$를 찾으려면 시스템의 메모리 특성, 즉 가용 메모리와 블록 크기의 비율을 알아야 하기 때문입니다.

캐시 비인지적 알고리즘이 흥미로운 이유는 특정 계층을 위해 특별히 튜닝되지 않았음에도 불구하고 캐시 계층 구조의 모든 메모리 레벨에서 자동으로 최적이 되기 때문입니다. 이 문서에서는 행렬 계산에서의 몇 가지 응용 사례를 살펴봅니다.

## 행렬 전치 (Matrix Transposition)

크기가 $N \times N$인 정사각형 행렬 $A$가 있고 이를 전치해야 한다고 가정해 봅시다. 정의에 따른 나이브한 접근 방식은 다음과 같습니다:

```cpp
for (int i = 0; i < n; i++)
    for (int j = 0; j < i; j++)
        swap(a[j * N + i], a[i * N + j]);
```

여기서는 메모리 연산을 더 명시적으로 보여주기 위해 2차원 배열 대신 메모리 영역의 시작 부분을 가리키는 단일 포인터를 사용했습니다.

이 코드의 I/O 복잡도는 $O(N^2)$입니다. 쓰기 작업이 순차적이지 않기 때문입니다. 반복 변수를 서로 바꾸더라도 상황은 반대가 될 뿐 결과는 동일합니다.

### 알고리즘

*캐시 비인지적* 알고리즘은 다음과 같은 블록 행렬 항등식에 기반합니다:

$$
\begin{pmatrix}
A & B \\
C & D
\end{pmatrix}^T=
\begin{pmatrix}
A^T & C^T \\
B^T & D^T
\end{pmatrix}
$$

이를 통해 분할 정복(divide-and-conquer) 접근 방식을 사용하여 문제를 재귀적으로 해결할 수 있습니다:

1. 입력 행렬을 4개의 작은 행렬로 나눕니다.
2. 각 행렬을 재귀적으로 전치합니다.
3. 모서리 결과 행렬을 서로 바꾸어 결과를 합칩니다.

행렬에서 분할 정복을 구현하는 것은 배열보다 약간 더 복잡하지만 기본 아이디어는 동일합니다. 하위 행렬을 명시적으로 복사하는 대신 하위 행렬에 대한 "뷰(view)"를 사용하고, 데이터가 L1 캐시에 들어갈 정도로 작아지면 나이브한 방식으로 전환합니다(또는 미리 알 수 없다면 $32 \times 32$와 같이 작은 크기를 선택합니다). 또한 $n$이 홀수여서 행렬을 4개의 동일한 하위 행렬로 나눌 수 없는 경우를 신중하게 처리해야 합니다.

```cpp
void transpose(int *a, int n, int N) {
    if (n <= 32) {
        for (int i = 0; i < n; i++)
            for (int j = 0; j < i; j++)
                swap(a[i * N + j], a[j * N + i]);
    } else {
        int k = n / 2;

        transpose(a, k, N);
        transpose(a + k, k, N);
        transpose(a + k * N, k, N);
        transpose(a + k * N + k, k, N);
        
        for (int i = 0; i < k; i++)
            for (int j = 0; j < k; j++)
                swap(a[i * N + (j + k)], a[(i + k) * N + j]);
        
        if (n & 1)
            for (int i = 0; i < n - 1; i++)
                swap(a[i * N + n - 1], a[(n - 1) * N + i]);
    }
}
```

이 알고리즘의 I/O 복잡도는 $O(\frac{N^2}{B})$입니다. 각 병합 단계에서 메모리 블록의 약 절반만 건드리면 되므로 단계마다 문제가 작아지기 때문입니다.

이 코드를 정사각형이 아닌 일반적인 행렬의 경우로 확장하는 것은 독자의 연습 문제로 남겨둡니다.

## 행렬 곱셈 (Matrix Multiplication)

다음으로 조금 더 복잡한 행렬 곱셈을 고려해 봅시다.

$$
C_{ij} = \sum_k A_{ik} B_{kj}
$$

나이브한 알고리즘은 정의를 그대로 코드로 옮긴 것입니다:

```cpp
// c[][]를 0으로 초기화하는 것을 잊지 마세요
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        for (int k = 0; k < n; k++)
            c[i * n + j] += a[i * n + k] * b[k * n + j];
```

각 스칼라 곱셈마다 별도의 블록 읽기가 필요하므로 총 $O(N^3)$개의 블록에 액세스해야 합니다.

잘 알려진 최적화 중 하나는 먼저 $B$를 전치하는 것입니다:

```cpp
for (int i = 0; i < n; i++)
    for (int j = 0; j < i; j++)
        swap(b[j][i], b[i][j])
// ^ 또는 이전에 만든 더 빠른 전치 알고리즘을 사용하세요

for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        for (int k = 0; k < n; k++)
            c[i * n + j] += a[i * n + k] * b[j * n + k]; // <- 인덱스에 주의하세요
```

전치를 나이브하게 하든 이전에 개발한 캐시 비인지적 방법으로 하든, 행렬 중 하나를 전치하여 곱셈을 수행하면 모든 메모리 액세스가 순차적이 되므로 $O(N^3/B + N^2)$의 성능을 보입니다.

더 이상 개선할 수 없을 것 같지만, 실제로는 가능합니다.

### 알고리즘

캐시 비인지적 행렬 곱셈은 본질적으로 전치와 동일한 트릭을 사용합니다. 데이터가 가장 낮은 캐시에 들어갈 때까지($N^2 \leq M$) 데이터를 나눕니다. 행렬 곱셈의 경우 다음 공식을 사용하는 것과 같습니다:

$$
\begin{pmatrix}
A_{11} & A_{12} \\
A_{21} & A_{22} \\
\end{pmatrix} \begin{pmatrix}
B_{11} & B_{12} \\
B_{21} & B_{22} \\
\end{pmatrix} = \begin{pmatrix}
A_{11} B_{11} + A_{12} B_{21} & A_{11} B_{12} + A_{12} B_{22}\\
A_{21} B_{11} + A_{22} B_{21} & A_{21} B_{12} + A_{22} B_{22}\\
\end{pmatrix}
$$

하지만 총 8번의 재귀적 행렬 곱셈이 필요하기 때문에 구현하기는 조금 더 어렵습니다:

```cpp
void matmul(const float *a, const float *b, float *c, int n, int N) {
    if (n <= 32) {
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                for (int k = 0; k < n; k++)
                    c[i * N + j] += a[i * N + k] * b[k * N + j];
    } else {
        int k = n / 2;

        // c11 = a11 b11 + a12 b21
        matmul(a,     b,         c, k, N);
        matmul(a + k, b + k * N, c, k, N);
        
        // c12 = a11 b12 + a12 b22
        matmul(a,     b + k,         c + k, k, N);
        matmul(a + k, b + k * N + k, c + k, k, N);
        
        // c21 = a21 b11 + a22 b21
        matmul(a + k * N,     b,         c + k * N, k, N);
        matmul(a + k * N + k, b + k * N, c + k * N, k, N);
        
        // c22 = a21 b12 + a22 b22
        mul(a + k * N,     b + k,         c + k * N + k, k, N);
        mul(a + k * N + k, b + k * N + k, c + k * N + k, k, N);

        if (n & 1) {
            for (int i = 0; i < n; i++)
                for (int j = 0; j < n; j++)
                    for (int k = (i < n - 1 && j < n - 1) ? n - 1 : 0; k < n; k++)
                        c[i * N + j] += a[i * N + k] * b[k * N + j];
        }
    }
}
```

여기에는 다른 많은 요인이 작용하므로 벤치마크는 생략하고 외부 메모리 모델에서의 이론적 성능 분석만 수행하겠습니다.

### 분석

알고리즘의 산술 복잡도는 동일하게 유지됩니다. 점화식

$$
T(N) = 8 \cdot T(N/2) + \Theta(N^2)
$$

은 $T(N) = \Theta(N^3)$으로 풀리기 때문입니다.

아직 "정복"한 것이 없는 것 같지만, I/O 복잡도를 생각해 봅시다:

$$
T(N) = \begin{cases}
O(\frac{N^2}{B}) & N \leq \sqrt M & \text{(데이터를 읽기만 하면 됨)} \\
8 \cdot T(N/2) + O(\frac{N^2}{B}) & \text{그 외}
\end{cases}
$$

이 점화식은 $O((\frac{N}{\sqrt M})^3)$의 기저 사례(base case)에 의해 지배되므로 총 복잡도는 다음과 같습니다.

$$
T(N) = O\left(\frac{(\sqrt{M})^2}{B} \cdot \left(\frac{N}{\sqrt M}\right)^3\right) = O\left(\frac{N^3}{B\sqrt{M}}\right)
$$

이는 단순히 $O(\frac{N^3}{B})$인 것보다 훨씬 더 낫습니다.

### 스트라센 알고리즘 (Strassen Algorithm)

카라츠바(Karatsuba) 알고리즘과 유사하게, 행렬 곱셈도 크기가 $\frac{n}{2}$인 7개의 행렬 곱셈을 포함하는 방식으로 분해할 수 있으며, 마스터 정리에 따르면 이러한 분할 정복 알고리즘은 $O(n^{\log_2 7}) \approx O(n^{2.81})$ 시간 내에 작동하며 외부 메모리 모델에서도 유사한 점근적 성능을 보입니다.

스트라센 알고리즘으로 알려진 이 기술은 마찬가지로 각 행렬을 4개로 나눕니다:

$$
\begin{pmatrix}
C_{11} & C_{12} \\
C_{21} & C_{22} \\
\end{pmatrix}
=\begin{pmatrix}
A_{11} & A_{12} \\
A_{21} & A_{22} \\
\end{pmatrix}
\begin{pmatrix}
B_{11} & B_{12} \\
B_{21} & B_{22} \\
\end{pmatrix}
$$

그런 다음 $\frac{N}{2} \times \frac{N}{2}$ 행렬의 중간 곱을 계산하고 이를 결합하여 행렬 $C$를 얻습니다:

$$
\begin{aligned}
   M_1 &= (A_{11} + A_{22})(B_{11} + B_{22})   & C_{11} &= M_1 + M_4 - M_5 + M_7
\\ M_2 &= (A_{21} + A_{22}) B_{11}             & C_{12} &= M_3 + M_5
\\ M_3 &= A_{11} (B_{21} - B_{22})             & C_{21} &= M_2 + M_4
\\ M_4 &= A_{22} (B_{21} - B_{11})             & C_{22} &= M_1 - M_2 + M_3 + M_6
\\ M_5 &= (A_{11} + A_{12}) B_{22}
\\ M_6 &= (A_{21} - A_{11}) (B_{11} + B_{12})
\\ M_7 &= (A_{12} - A_{22}) (B_{21} + B_{22})
\end{aligned}
$$

원한다면 단순 대입을 통해 이 공식들을 검증할 수 있습니다.

내가 아는 바로는 스트라센 알고리즘을 사용하는 주류 최적화 선형 대수 라이브러리는 없지만, 2000 이상의 큰 행렬에 대해 효율적인 [일부 프로토타입 구현](https://arxiv.org/pdf/1605.01078.pdf)이 존재합니다.

이 기술은 더 많은 하위 행렬 곱을 고려하여 점근적 성능을 더욱 낮추기 위해 여러 번 확장되었습니다. 2020년 기준 세계 기록은 $O(n^{2.3728596})$입니다. 행렬을 $O(n^2)$ 또는 최소한 $O(n^2 \log^k n)$ 시간 내에 곱할 수 있는지 여부는 여전히 미해결 문제입니다.

## 더 읽어보기

탄탄한 이론적 관점을 원한다면 Erik Demaine의 [Cache-Oblivious Algorithms and Data Structures](https://erikdemaine.org/papers/BRICS2002/paper.pdf)를 읽어보시기 바랍니다.
