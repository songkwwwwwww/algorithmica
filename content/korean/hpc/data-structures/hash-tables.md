---
title: 해시 테이블 (Hash Tables)
weight: 8
draft: true
---


## 해시 테이블

![](https://upload.wikimedia.org/wikipedia/commons/thumb/7/7d/Hash_table_3_1_1_0_1_0_0_SP.svg/2560px-Hash_table_3_1_1_0_1_0_0_SP.svg.png =500x)

----

### 체이닝 (Chaining)

![](https://upload.wikimedia.org/wikipedia/commons/d/d0/Hash_table_5_0_1_1_1_1_1_LL.svg =500x)

많은 수의 연결 리스트 또는 가변 배열

----

### 오픈 어드레싱 (Open Addressing)

![](https://upload.wikimedia.org/wikipedia/commons/b/bf/Hash_table_5_0_1_1_1_1_0_SP.svg =500x)

고정된 수의 셀과 $i$번째 단계에서 확인할 위치를 결정하는 해시 함수 $f_i(x)$

----

순환 배열(cyclic array)을 이용한 구현:

```cpp
struct hashmap {
    const int size = (1<<24);
    int a[size] = {-1}, b[size];

    static inline int h(int x) { return (x^179)*7; }

    void add(int x, int y) {
        int k = h(x) % size;
        while (a[k] != -1 && a[k] != x)
            k = (k + 1) % size;
        a[k] = x, b[k] = y; 
    }

    int get(int x) {
        for (int k = h(x) % size; a[k] != -1; k = (k + 1) % size)
            if (a[k] == x)
                return b[k];
        return -1;
    }
};
```

점근적 복잡도(asymptotic complexity)는 같지만, 실제 속도에서는 2-3배 차이가 납니다.

----

![](https://upload.wikimedia.org/wikipedia/commons/1/1c/Hash_table_average_insertion_time.png =500x)

유일한 단점은 재해싱(rehashing)을 더 자주 해야 한다는 것입니다.
