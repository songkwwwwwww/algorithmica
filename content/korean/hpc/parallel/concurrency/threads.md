---
title: 스레드 (Threads)
weight: 2
---

스레드는 경량 프로세스(lightweight processes)입니다. 스레드는 다른 스레드와 코드 섹션, 데이터 섹션, 그리고 열린 파일이나 시그널과 같은 OS 리소스를 공유합니다. 하지만 프로세스와 마찬가지로, 스레드는 자신만의 프로그램 카운터(PC), 레지스터 세트, 그리고 스택 공간을 가집니다.

```cpp
#include <iostream>
using namespace std;

void print(int a[], int sz)
{
  for (int i = 0; i < sz; i++) cout << a[i] << " ";
  cout << endl;
}
 
void merge(int a[], const int low, const int mid, const int high)
{
  int *temp = new int[high-low+1];
        
  int left = low;
  int right = mid+1;
  int current = 0;
  // 두 배열을 temp[]로 병합합니다.
  while(left <= mid && right <= high) {
    if(a[left] <= a[right]) {
      temp[current] = a[left];
      left++;
    }
    else { // 오른쪽 원소가 왼쪽보다 작을 경우
      temp[current] = a[right];  
      right++;
    }
    current++;
  }

  // 배열을 완성합니다.

        // 극단적인 예시 a = 1, 2, 3 || 4, 5, 6
        // temp 배열이 이미 1, 2, 3으로 채워졌습니다.
        // 따라서 배열 a의 오른쪽 부분이 temp를 채우는 데 사용됩니다.
  if(left > mid) { 
    for(int i=right; i <= high;i++) {
      temp[current] = a[i];
      current++;
    }
  }
        // 극단적인 예시 a = 6, 5, 4 || 3, 2, 1
        // temp 배열이 이미 1, 2, 3으로 채워졌습니다.
        // 따라서 배열 a의 왼쪽 부분이 temp를 채우는 데 사용됩니다.
  else {  
    for(int i=left; i <= mid; i++) {
      temp[current] = a[i];
      current++;
    }
  }
  // 원래 배열로 복사합니다.
  for(int i=0; i<=high-low;i++) {
                a[i+low] = temp[i];
  }
  delete[] temp;
}
 
void merge_sort(int a[], const int low, const int high)
{
  if(low >= high) return;
  int mid = (low+high)/2;
  merge_sort(a, low, mid);  // 왼쪽 절반
  merge_sort(a, mid+1, high);  // 오른쪽 절반
  merge(a, low, mid, high);  // 병합합니다.
}
 
int main()
{        
  int a[] = {38, 27, 43, 3, 9, 82, 10};
  int arraySize = sizeof(a)/sizeof(int);

  print(a, arraySize);

  merge_sort(a, 0, (arraySize-1) );   

  print(a, arraySize);  
  return 0;
}
```

스레드는 여전히 운영체제에 의해 관리됩니다. 단지 서로 메모리를 공유할 수 있는 더 가벼운 구조일 뿐입니다.

## 스레드 vs 프로세스

언어 수준의 스레드, 운영체제 스레드, 그리고 하드웨어 스레드를 구분하는 것이 중요합니다.

```python
import logging
import threading
import time

def thread_function(name):
    logging.info("Thread %s: starting", name)
    time.sleep(2)
    logging.info("Thread %s: finishing", name)

if __name__ == "__main__":
    format = "%(asctime)s: %(message)s"
    logging.basicConfig(format=format, level=logging.INFO,
                        datefmt="%H:%M:%S")

    logging.info("Main    : before creating thread")
    x = threading.Thread(target=thread_function, args=(1,))
    logging.info("Main    : before running thread")
    x.start()
    logging.info("Main    : wait for the thread to finish")
    # x.join()
    logging.info("Main    : all done")
```

일부 언어는 하드웨어 스레드를 지원하지 않습니다. 예를 들어, 파이썬에는 전역 인터프리터 락(Global Interpreter Lock, GIL)이라는 것이 있어 두 스레드가 동시에 실행되는 것을 방지합니다. 이는 동시성 처리를 단순화하기 위해 만들어졌습니다. 파이썬(및 기타 단일 스레드 언어)에서 이를 해결하는 방법은 대신 프로세스를 생성하는 것입니다.

```python
# 프로세스를 사용한 예시
```
