---
title: 간접 분기
weight: 4
---

어셈블리 과정에서 모든 레이블은 주소(절대 주소 또는 상대 주소)로 변환된 다음 점프 명령어에 인코딩됩니다.

레지스터 내부에 저장된 상수가 아닌 값으로도 점프할 수 있는데, 이를 *계산된 점프(computed jump)*라고 합니다.

```nasm
jmp rax
```

이는 동적 언어 및 더 복잡한 제어 흐름 구현과 관련된 몇 가지 흥미로운 응용 분야를 가지고 있습니다.

### 다중 분기

`switch` 문이 무엇인지 잊으셨을 수도 있으니, 미국의 성적 시스템에서 GPA를 계산하는 작은 서브루틴을 예로 들어보겠습니다.

```cpp
switch (grade) {
    case 'A':
        return 4.0;
        break;
    case 'B':
        return 3.0;
        break;
    case 'C':
        return 2.0;
        break;
    case 'D':
        return 1.0;
        break;
    case 'E':
    case 'F':
        return 0.0;
        break;
    default:
        return NAN;
}
```

개인적으로 교육적인 맥락이 아닌 곳에서 마지막으로 switch를 사용한 게 언제인지 기억이 나지 않습니다. 일반적으로 switch 문은 "if, else if, else if, else if…" 등의 시퀀스와 동일하며, 이러한 이유로 많은 언어에는 switch 문이 아예 없기도 합니다. 그럼에도 불구하고 이러한 제어 흐름 구조는 파서, 인터프리터 및 기타 상태 머신을 구현하는 데 중요하며, 이는 종종 단일 `while (true)` 루프와 그 내부의 `switch (state)` 문으로 구성됩니다.

변수가 가질 수 있는 값의 범위를 제어할 수 있는 경우, 계산된 점프를 활용하여 다음과 같은 트릭을 사용할 수 있습니다. $n$개의 조건부 분기를 만드는 대신, 가능한 점프 위치에 대한 포인터/오프셋을 포함하는 *분기 테이블(branch table)*을 만들고, $[0, n)$ 범위의 값을 갖는 `state` 변수로 인덱싱하면 됩니다.

컴파일러는 값들이 조밀하게 모여 있을 때(반드시 엄격하게 순차적일 필요는 없지만, 테이블에 빈 필드를 둘 만한 가치가 있어야 함) 이 기술을 사용합니다. 또한 *계산된 goto(computed goto)*를 사용하여 명시적으로 구현할 수도 있습니다.

```cpp
void weather_in_russia(int season) {
    static const void* table[] = {&&winter, &&spring, &&summer, &&fall};
    goto *table[season];

    winter:
        printf("Freezing\n");
        return;
    spring:
        printf("Dirty\n");
        return;
    summer:
        printf("Dry\n");
        return;
    fall:
        printf("Windy\n");
        return;
}
```

Switch 기반 코드는 컴파일러가 최적화하기에 항상 간단한 것은 아니므로, 상태 머신의 맥락에서는 `goto` 문이 직접 사용되는 경우가 많습니다. `glibc`의 I/O 관련 부분에 많은 예가 있습니다.

### 동적 디스패치

간접 분기는 런타임 다형성을 구현하는 데에도 중요한 역할을 합니다.

가상 `.speak()` 메서드를 가진 추상 클래스 `Animal`과, 짖는 `Dog`와 야옹하는 `Cat`이라는 두 개의 구체적인 구현이 있는 전형적인 예시를 생각해 봅시다.

```cpp
struct Animal {
    virtual void speak() { printf("<abstract animal sound>\n");}
};

struct Dog : Animal {
    void speak() override { printf("Bark\n"); }
};

struct Cat : Animal {
    void speak() override { printf("Meow\n"); }
};
```

우리는 동물을 생성하고 그 타입을 미리 알지 못한 채 `.speak()` 메서드를 호출하고 싶으며, 이는 어떻게든 올바른 구현을 호출해야 합니다.

```c++
Dog sparkles;
Cat mittens;

Animal *catdog = (rand() & 1) ? (Animal*)&sparkles : (Animal*)&mittens;
catdog->speak();
```

이 동작을 구현하는 방법은 여러 가지가 있지만, C++는 *가상 메서드 테이블(virtual method table)*을 사용하여 이를 수행합니다.

`Animal`의 모든 구체적인 구현에 대해, 컴파일러는 모든 메서드(즉, 명령어 시퀀스)를 모든 클래스에 대해 정확히 동일한 길이가 되도록 패딩을 넣고(`ret` 뒤에 일부 [채우기 명령어](../layout) 삽입), 명령어 메모리의 어딘가에 순차적으로 씁니다. 그런 다음 구조체(즉, 모든 인스턴스)에 *런타임 타입 정보(run-time type information, RTTI)* 필드를 추가하는데, 이는 기본적으로 클래스의 가상 메서드의 올바른 구현을 가리키는 메모리 영역의 오프셋일 뿐입니다.

가상 메서드 호출 시, 해당 오프셋 필드를 구조체 인스턴스에서 가져와 일반적인 함수 호출을 수행하며, 이때 모든 파생 클래스의 모든 메서드와 기타 필드가 정확히 동일한 오프셋을 갖는다는 사실을 이용합니다.

물론 이는 약간의 오버헤드를 추가합니다.

- [분기 예측 실패](/hpc/pipelining)와 동일한 파이프라인 플러싱(pipeline flushing) 이유로 약 15 사이클 정도를 더 소비해야 할 수도 있습니다.
- 컴파일러가 함수 호출 자체를 인라이닝할 수 없을 가능성이 매우 높습니다.
- 클래스 크기가 몇 바이트 정도 증가합니다(이는 구현에 따라 다름).
- 바이너리 크기 자체가 약간 증가합니다.

이러한 이유로 성능이 중요한 애플리케이션에서는 일반적으로 런타임 다형성을 피합니다.
