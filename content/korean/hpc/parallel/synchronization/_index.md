---
title: 동기화 기본 요소 (Synchronization Primitives)
weight: 2
---

다음 루프를 고려해 봅시다:

```cpp
int s = 0;

for (int i = 0; i < n; i++) {
    s += a[i];
}
```

이를 다음과 같이 병렬로 만들 수 있습니다:

```cpp
int s = 0;

#pragma omp parallel for
for (int i = 0; i < n; i++) {
    s += a[i];
}
```

이 스니펫은 다음 장에서 다룰 OpenMP를 사용합니다. 지금 당장 알아야 할 것은 이것이 여러 스레드를 생성하고 작업을 고르게 분산시킨다는 점입니다. C++ 스레드를 사용하여 동일한 기능을 하는 함수를 작성할 수 있습니다.

문제는...
