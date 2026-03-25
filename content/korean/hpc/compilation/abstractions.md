---
title: 비용이 발생하는 추상화 (Non-Zero-Cost Abstractions)
weight: 7
draft: true
---

일반적으로 추상화는 훌륭합니다. 잘 적용되면 코드의 양과 프로그래머의 정신적 부담을 줄여줍니다.

하지만 추상화는 종종 성능 측면에서 비용을 수반합니다. 공유 라이브러리를 사용할 때, 해당 함수를 적절히 호출하기 위해 데이터를 이동시키는 데 추가적인 사이클을 소비해야 합니다. 가상 메서드(virtual method)를 호출할 때, 다음에 어떤 코드가 실행될지 확실히 예측할 수 없으며 효과적으로 분기 예측 실패(branch mispredict)를 겪게 됩니다.

C++와 Rust 같은 언어들은 런타임 오버헤드가 없으며, 원칙적으로 컴파일러에 의해 완전히 제거될 수 있는 *제로 비용(zero-cost)* 추상화라는 개념을 강력하게 홍보합니다. 하지만 실제로는 제로 비용 추상화라는 것은 존재하지 않습니다 — 컴파일러 기술이 아직 그 단계에 도달하지 못했기 때문입니다.

**가상 함수 (Virtual functions).** 모든 형태의 런타임 다형성.

**범위 검사 (Bounds checking).** 컴파일러가 이를 제거하는 데 능숙하긴 합니다.

**일반적으로 복잡한 모든 코드.** 개인적으로 저를 괴롭히는 예시는 C++ 표준 라이브러리의 `std::min`입니다. 단순히 최솟값을 직접 구하는 것보다 반복적으로 성능이 떨어지는데, 그 이유는 단순히 `return (a < b ? a : b)`로 구현된 것이 아니라 범용성을 위해 가변 인자 초기화 리스트와 반복자를 사용하여 구현되었기 때문입니다.

```cpp
template<typename _Tp> GLIBCXX14_CONSTEXPR inline _Tp min(initializer_list<_Tp> __l) {
    return *std::min_element(__l.begin(), __l.end());
}
```

대개 작은 프로그램을 하드웨어에 더 가깝고 직관적으로 다시 작성하는 것은 그리 어렵지 않습니다. 추상화 계층을 제거하기 시작하면 컴파일러는 결국 굴복할 것입니다.

객체 지향 언어와 특히 함수형 언어에는 이와 같이 뚫기 힘든 추상화가 존재합니다. 이러한 이유로 사람들은 성능이 중요한 소프트웨어(인터프리터, 런타임, 데이터베이스)를 고수준 언어보다는 C에 가까운 스타일로 작성하는 것을 선호하곤 합니다.

수염이 덥수룩한 C/어셈블리 프로그래머들처럼 말이죠.

### 메모리 (Memory)

포인터 추적(Pointer chasing).

```c++
typedef vector< vector<int> > matrix;
matrix a(n, vector<int>(n, 0));

int val = a[i][j];
```

이것은 최대 두 배까지 느립니다: 먼저 포인터를 가져와야 하기 때문입니다.

```c++
int a = new int[n * n];
memset(a, 0, 4 * n* n);

int val = a[i * n + j];
```

정말로 추상화를 원한다면 래퍼(wrapper)를 작성할 수 있습니다:

```c++
template<typename T>
struct Matrix {
    int x, y, n, N;
    T* data;
    T* operator[](int i) { return data + (x + i) * N + y; }
};
```

예를 들어, [캐시 무관 전치(cache-oblivious transposition)](/hpc/external-memory/oblivious)는 다음과 같이 진행될 수 있습니다:

```c++
Matrix<T> subset(int _x, int _y, int _n) { return {_n, _x, _y, N, data}; }

Matrix<T> transpose() {
    if (n <= 32) {
        for (int i = 0; i < n; i++)
            for (int j = 0; j < i; j++)
                swap((*this)[j][i], (*this)[i][j]);
    } else {
        auto A = subset(x, y, n / 2).transpose();
        auto B = subset(x + n / 2, y, n / 2).transpose();
        auto C = subset(x, y + n / 2, n / 2).transpose();
        auto D = subset(x + n / 2, y + n / 2, n / 2).transpose();
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                swap(B[i][j], C[i][j]);
    }

    return *this;
}
```

저는 개인적으로 저수준 코드를 작성하는 것을 선호하는데, 최적화하기가 더 쉽기 때문입니다.

더 깔끔하다고요? 그렇게 생각하지 않습니다.
