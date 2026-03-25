---
title: 프로그래밍 언어 (Programming Languages)
aliases:
  - /hpc/analyzing-performance
weight: 2
---

이 책을 읽고 있다면, 컴퓨터 과학 여정의 어느 시점에서 코드의 효율성에 대해 처음으로 고민하기 시작한 순간이 있었을 것입니다.

저의 경우는 고등학교 때였습니다. 웹사이트를 만들고 *유용한* 프로그래밍을 하는 것만으로는 대학에 갈 수 없다는 것을 깨닫고, 알고리즘 프로그래밍 올림피아드라는 흥미진진한 세계에 발을 들였을 때였죠. 저는 고등학생치고는 괜찮은 프로그래머였지만, 그전까지 제 코드가 실행되는 데 시간이 얼마나 걸리는지 진지하게 궁금해한 적이 없었습니다. 하지만 갑자기 그것이 중요해졌습니다. 각 문제에는 엄격한 시간 제한이 있었기 때문입니다. 저는 연산 횟수를 세기 시작했습니다. 1초에 몇 번의 연산을 할 수 있을까요?

이 질문에 답할 만큼 컴퓨터 아키텍처에 대해 잘 알지는 못했습니다. 하지만 정확한 답이 필요한 것은 아니었습니다. 어림잡아 계산할 규칙(rule of thumb)이 필요했죠. 제 생각의 흐름은 이랬습니다: "2~3GHz는 매초 20억에서 30억 개의 명령어가 실행된다는 뜻이고, 배열 요소로 무언가를 하는 단순한 루프에서는 루프 카운터를 늘리고, 종료 조건을 확인하고, 배열 인덱싱을 하는 등의 작업도 필요하니까, 유용한 명령어 하나당 3~5개의 명령어가 더 필요하다고 치자." 그렇게 해서 $5 \cdot 10^8$이라는 추정치를 사용하게 되었습니다. 이 진술 중 어느 것도 사실은 아니지만, 알고리즘이 필요로 하는 연산 횟수를 세고 이 숫자로 나누는 것은 제 사례에서 훌륭한 어림셈이 되었습니다.

물론 진짜 답은 훨씬 더 복잡하며 여러분이 생각하는 "연산"의 종류에 따라 크게 달라집니다. [포인터 추적(pointer chasing)](/hpc/cpu-cache/latency)과 같은 작업은 $10^7$만큼 낮을 수 있고, [SIMD 가속](/hpc/simd) 선형 대수의 경우 $10^{11}$만큼 높을 수 있습니다. 이러한 현저한 차이를 보여주기 위해, 서로 다른 언어로 구현된 행렬 곱셈 사례 연구를 살펴보고 컴퓨터가 이를 어떻게 실행하는지 깊이 파고들어 보겠습니다.

<!--

Because of this logic, and also because of the [computation model](../) postulated in CS 101, many programmers have a misconception that computers can execute a certain number of "operations" per second, and that using different programming languages has some sort of [multiplier effect](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html) on that number:

- "you can execute about $5 \cdot 10^8$ operations per second on this machine,"
- "C is 2 times faster than Java,"
- "Python is 100x slower than C++."

-->

## 언어의 종류 (Types of Languages)

<!--

Processors can be thought of as *state machines*. They keep their *state* in several fixed-length *registers*, one of which, the instruction pointer, indicates a memory location of the next instruction to be read and executed. This instruction somehow modifies the registers and moves the instruction pointer to the next instruction to be executed, and so on.

These instructions — called *machine code* — are binary encoded, quirky and very difficult to work with, so no sane person writes them directly nowadays. Instead, we use higher-level programming languages and employ alternative means to feed instructions to the processor.

-->

가장 낮은 수준에서 컴퓨터는 CPU를 제어하는 데 사용되는 이진 인코딩된 *명령어*로 구성된 *기계어(machine code)*를 실행합니다. 기계어는 구체적이고 독특하며 다루는 데 많은 지적 노력이 필요하므로, 사람들이 컴퓨터를 만든 후 가장 먼저 한 일 중 하나는 프로그래밍 과정을 단순화하기 위해 컴퓨터의 작동 세부 사항을 추상화한 *프로그래밍 언어*를 만드는 것이었습니다.

프로그래밍 언어는 근본적으로 인터페이스일 뿐입니다. 그 언어로 작성된 모든 프로그램은 CPU에서 실행되기 위해 어느 시점에서 기계어로 변환되어야 하는 더 나은 고수준 표현일 뿐입니다. 그리고 이를 수행하는 몇 가지 서로 다른 방법이 있습니다:

- 프로그래머의 관점에서는 두 가지 유형의 언어가 있습니다: 실행 전에 미리 처리하는 *컴파일(compiled)* 언어와, 실행 중에 *인터프리터*라는 별도의 프로그램을 사용하여 실행되는 *인터프리터(interpreted)* 언어입니다.
- 컴퓨터의 관점에서도 두 가지 유형의 언어가 있습니다: 기계어를 직접 실행하는 *네이티브(native)* 언어와, 이를 수행하기 위해 일종의 *런타임(runtime)*에 의존하는 *매니지드(managed)* 언어입니다.

인터프리터에서 기계어를 실행하는 것은 말이 되지 않으므로, 총 세 가지 유형의 언어가 남습니다:

- 파이썬(Python), 자바스크립트(JavaScript) 또는 루비(Ruby)와 같은 인터프리터 언어.
- 자바(Java), C# 또는 얼랭(Erlang)과 같이 런타임이 있는 컴파일 언어 (그리고 Scala, F# 또는 Elixir와 같이 해당 VM에서 작동하는 언어들).
- C, Go 또는 Rust와 같이 네이티브 컴파일 언어.

컴퓨터 프로그램을 실행하는 "정답"은 없습니다. 각 접근 방식은 고유한 이점과 단점을 가집니다. 인터프리터와 가상 머신은 유연성을 제공하고 동적 타이핑, 런타임 코드 수정, 자동 메모리 관리와 같은 훌륭한 고수준 프로그래밍 기능을 가능하게 하지만, 이러한 기능에는 피할 수 없는 성능상의 절충(trade-offs)이 따르며 이제 그에 대해 이야기해 보겠습니다.

### 인터프리터 언어 (Interpreted languages)

다음은 순수 파이썬으로 구현된 $1024 \times 1024$ 행렬 곱셈 예제입니다:

```python
import time
import random

n = 1024

a = [[random.random()
      for row in range(n)]
      for col in range(n)]

b = [[random.random()
      for row in range(n)]
      for col in range(n)]

c = [[0
      for row in range(n)]
      for col in range(n)]

start = time.time()

for i in range(n):
    for j in range(n):
        for k in range(n):
            c[i][j] += a[i][k] * b[k][j]

duration = time.time() - start
print(duration)
```

이 코드는 630초가 걸립니다. 10분이 넘는 시간이죠!

이 숫자를 객관적으로 살펴봅시다. 이를 실행한 CPU의 클록 주파수는 1.4GHz입니다. 즉, 초당 $1.4 \cdot 10^9$ 사이클을 수행하며, 전체 계산에는 거의 $10^{15}$ 사이클이, 가장 안쪽 루프의 곱셈 한 번당 약 880 사이클이 소요된다는 뜻입니다.

파이썬이 프로그래머의 의도를 파악하기 위해 수행해야 하는 일들을 고려하면 이는 놀라운 일이 아닙니다:

- `c[i][j] += a[i][k] * b[k][j]` 식을 파싱합니다.
- `a`, `b`, `c`가 무엇인지 알아내려 하고 타입 정보가 포함된 특수 해시 테이블에서 그 이름을 조회합니다.
- `a`가 리스트임을 이해하고, 그 `[]` 연산자를 가져오고, `a[i]`에 대한 포인터를 검색하고, 그것도 리스트임을 파악하고, 다시 그 `[]` 연산자를 가져오고, `a[i][k]`에 대한 포인터를 얻은 다음 실제 요소를 가져옵니다.
- 그 타입을 조회하고, 그것이 `float`임을 알아내고, `*` 연산자를 구현하는 메서드를 가져옵니다.
- `b`와 `c`에 대해서도 동일한 작업을 수행하고 마침내 결과를 `c[i][j]`에 더하여 할당합니다.

물론 파이썬과 같이 널리 사용되는 언어의 인터프리터는 잘 최적화되어 있으며 동일한 코드를 반복 실행할 때 이러한 단계 중 일부를 건너뛸 수 있습니다. 하지만 언어 설계상 피할 수 없는 상당한 오버헤드가 여전히 존재합니다. 만약 우리가 이 모든 타입 체크와 포인터 추적을 제거한다면, 곱셈당 사이클 비율을 1에 가깝게 하거나 네이티브 곱셈의 "비용"이 얼마든 그에 가깝게 만들 수 있지 않을까요?

### 매니지드 언어 (Managed Languages)

동일한 행렬 곱셈 프로시저를 자바(Java)로 구현한 것입니다:

```java
import java.util.Random;

public class Matmul {
    static int n = 1024;
    static double[][] a = new double[n][n];
    static double[][] b = new double[n][n];
    static double[][] c = new double[n][n];

    public static void main(String[] args) {
        Random rand = new Random();

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                a[i][j] = rand.nextDouble();
                b[i][j] = rand.nextDouble();
                c[i][j] = 0;
            }
        }

        long start = System.nanoTime();

        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                for (int k = 0; k < n; k++)
                    c[i][j] += a[i][k] * b[k][j];
                
        double diff = (System.nanoTime() - start) * 1e-9;
        System.out.println(diff);
    }
}
```

이제 10초 만에 실행되며, 이는 곱셈당 약 13 CPU 사이클에 해당합니다. 파이썬보다 63배 빠릅니다. `b`의 요소를 메모리에서 비순차적으로 읽어야 한다는 점을 고려하면 실행 시간은 대략 예상한 대로입니다.

자바는 *컴파일* 언어이지만 *네이티브* 언어는 아닙니다. 프로그램은 먼저 *바이트코드*로 컴파일되고, 이는 가상 머신(JVM)에 의해 해석됩니다. 더 높은 성능을 달성하기 위해 가장 안쪽의 `for` 루프와 같이 자주 실행되는 코드 부분은 런타임 중에 기계어로 컴파일된 다음 오버헤드 거의 없이 실행됩니다. 이 기술을 *JIT 컴파일(just-in-time compilation)*이라고 합니다.

JIT 컴파일은 언어 자체의 기능이 아니라 구현 방식의 기능입니다. [PyPy](https://www.pypy.org/)라고 불리는 JIT 컴파일된 파이썬 버전도 있는데, 이는 코드 수정 없이 위 코드를 실행하는 데 약 12초가 걸립니다.

### 컴파일 언어 (Compiled Languages)

이제 C의 차례입니다:

```cpp
#include <stdlib.h>
#include <stdio.h>
#include <time.h>

#define n 1024
double a[n][n], b[n][n], c[n][n];

int main() {
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            a[i][j] = (double) rand() / RAND_MAX;
            b[i][j] = (double) rand() / RAND_MAX;
        }
    }

    clock_t start = clock();

    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            for (int k = 0; k < n; k++)
                c[i][j] += a[i][k] * b[k][j];

    float seconds = (float) (clock() - start) / CLOCKS_PER_SEC;
    printf("%.4f\n", seconds);
    
    return 0;
}
```

`gcc -O3`로 컴파일하면 9초가 걸립니다.

엄청난 개선처럼 보이지는 않습니다 — 자바나 PyPy보다 1~3초 앞선 것은 JIT 컴파일의 추가 시간 때문일 수 있습니다. 하지만 우리는 아직 훨씬 더 뛰어난 C 컴파일러 생태계를 활용하지 않았습니다. 만약 `-march=native`와 `-ffast-math` 플래그를 추가한다면, 시간은 갑자기 0.6초로 단축됩니다!

여기서 일어난 일은 우리가 [컴파일러에게 전달](/hpc/compilation/flags/)한 실행 중인 CPU의 정확한 모델(`-march=native`)과 [부동 소수점 계산](/hpc/arithmetic/float)을 재배치할 수 있는 자유(`-ffast-math`)를 준 것이고, 컴파일러는 이를 활용해 [벡터화(vectorization)](/hpc/simd)를 사용하여 이러한 속도 향상을 달성했습니다.

소스 코드를 크게 변경하지 않고도 동일한 성능을 달성하도록 PyPy와 자바의 JIT 컴파일러를 튜닝하는 것이 불가능한 것은 아니지만, 네이티브 코드로 직접 컴파일되는 언어에서 훨씬 더 쉽습니다.

### BLAS

마지막으로 전문가 수준으로 최적화된 구현이 무엇을 할 수 있는지 살펴봅시다. 널리 사용되는 최적화 선형 대수 라이브러리인 [OpenBLAS](https://www.openblas.net/)를 테스트해 보겠습니다. 가장 쉬운 방법은 다시 파이썬으로 돌아가 `numpy`에서 호출하는 것입니다.

```python
import time
import numpy as np

n = 1024

a = np.random.rand(n, n)
b = np.random.rand(n, n)

start = time.time()

c = np.dot(a, b)

duration = time.time() - start
print(duration)
```

이제 약 0.12초가 걸립니다. 자동 벡터화된 C 버전보다 약 5배 빠르고, 초기 파이썬 구현보다 약 5250배 빠릅니다!

일반적으로는 이렇게 극적인 개선을 보기는 힘듭니다. 지금 당장은 이것이 어떻게 달성되었는지 정확히 설명할 준비가 되지 않았습니다. OpenBLAS의 밀집 행렬 곱셈 구현은 일반적으로 각 아키텍처에 맞춰 별도로 작성된 [5000줄의 수동 어셈블리](https://github.com/xianyi/OpenBLAS/blob/develop/kernel/x86_64/dgemm_kernel_16x2_haswell.S)로 구성됩니다. 이후 장들에서 관련 기술들을 하나씩 설명하고, 다시 [이 예제로 돌아와서](/hpc/algorithms/matmul) 40줄 미만의 C 코드로 우리만의 BLAS 수준 구현을 개발해 보겠습니다.

### 요점 (Takeaway)

여기서 얻을 수 있는 핵심 교훈은 네이티브 저수준 언어를 사용한다고 해서 반드시 성능을 얻는 것은 아니지만, 성능에 대한 *제어권*을 얻게 된다는 것입니다.

"초당 N번의 연산"이라는 단순화와 마찬가지로, 많은 프로그래머들은 서로 다른 프로그래밍 언어를 사용하는 것이 그 숫자에 일종의 배수를 곱하는 효과가 있다는 오해를 가지고 있습니다. 그런 식으로 생각하고 [언어를 성능 관점에서 비교](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html)하는 것은 큰 의미가 없습니다. 프로그래밍 언어는 근본적으로 편리한 추상화를 대가로 성능에 대한 *일부* 제어권을 가져가는 도구일 뿐입니다. 실행 환경과 관계없이, 하드웨어가 제공하는 기회를 활용하는 것은 여전히 주로 프로그래머의 몫입니다.
