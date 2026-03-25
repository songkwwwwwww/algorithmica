---
title: 함수와 재귀
weight: 3
---

어셈블리에서 "함수를 호출"하려면, 함수의 시작 부분으로 [점프](../loops)한 다음 다시 돌아와야 합니다. 하지만 이때 두 가지 중요한 문제가 발생합니다.

1. 호출자(caller)가 피호출자(callee)와 동일한 레지스터에 데이터를 저장하면 어떻게 될까요?
2. "돌아갈" 곳은 어디일까요?

이 두 가지 문제는 함수를 호출하기 전에 함수에서 반환하는 데 필요한 모든 정보를 기록할 수 있는 메모리의 전용 공간을 둠으로써 해결할 수 있습니다. 이 공간을 *스택(stack)*이라고 부릅니다.

### 스택

하드웨어 스택은 소프트웨어 스택과 동일한 방식으로 작동하며, 유사하게 두 개의 포인터로 구현됩니다.

- *베이스 포인터(base pointer)*는 스택의 시작을 표시하며 관례적으로 `rbp`에 저장됩니다.
- *스택 포인터(stack pointer)*는 스택의 마지막 요소를 표시하며 관례적으로 `rsp`에 저장됩니다.

함수를 호출해야 할 때, 모든 지역 변수를 스택에 푸시하고(레지스터가 부족할 때와 같은 다른 상황에서도 가능), 현재 명령어 포인터를 푸시한 다음 함수의 시작 부분으로 점프합니다. 함수를 종료할 때는 스택 맨 위에 저장된 포인터를 확인하여 그곳으로 점프한 다음, 스택에 저장된 모든 변수를 다시 레지스터로 신중하게 읽어 들입니다.

이 모든 과정을 일반적인 메모리 연산과 점프를 통해 구현할 수 있지만, 매우 빈번하게 사용되기 때문에 이를 위한 4가지 특수 명령어가 존재합니다.

- `push`는 스택 포인터 위치에 데이터를 쓰고 포인터를 감소시킵니다.
- `pop`은 스택 포인터 위치에서 데이터를 읽고 포인터를 증가시킵니다.
- `call`은 다음 명령어의 주소를 스택 맨 위에 놓고 레이블로 점프합니다.
- `ret`은 스택 맨 위에서 반환 주소를 읽어 그곳으로 점프합니다.

이것들이 실제 하드웨어 명령어가 아니었다면 "구문적 설탕(syntactic sugar)"이라고 불렀을 것입니다. 이들은 단지 다음과 같은 두 명령어 조합의 융합된 형태일 뿐입니다.

```nasm
; "push rax"
sub rsp, 8
mov QWORD PTR [rsp], rax

; "pop rax"
mov rax, QWORD PTR [rsp]
add rsp, 8

; "call func"
push rip ; <- 명령어 포인터 (이런 식으로 직접 접근하는 것은 대개 허용되지 않음)
jmp func

; "ret"
pop  rcx ; <- 사용하지 않는 아무 레지스터나 선택
jmp rcx
```

`rbp`와 `rsp` 사이의 메모리 영역을 *스택 프레임(stack frame)*이라고 하며, 함수의 지역 변수가 일반적으로 저장되는 곳입니다. 이는 프로그램 시작 시 미리 할당되며, 스택에 용량(Linux에서는 기본적으로 8MB)보다 많은 데이터를 푸시하면 *스택 오버플로우(stack overflow)* 오류가 발생합니다. 현대의 운영 체제는 해당 주소 공간에 실제로 읽거나 쓰기 전까지는 메모리 페이지를 할당하지 않기 때문에, 매우 큰 스택 크기를 자유롭게 지정할 수 있습니다. 이는 사용할 수 있는 스택 메모리의 한계치처럼 작동하며, 모든 프로그램이 반드시 사용해야 하는 고정된 양은 아닙니다.

### 호출 규약

컴파일러와 운영 체제를 개발하는 사람들은 함수를 작성하고 호출하는 방법에 대한 [규약(conventions)](https://wiki.osdev.org/Calling_Conventions)을 만들었습니다. 이러한 규약은 컴파일을 별도의 단위로 분리하고, 이미 컴파일된 라이브러리를 재사용하며, 심지어 서로 다른 프로그래밍 언어로 작성된 라이브러리를 사용할 수 있게 하는 등 [소프트웨어 공학의 경이로운 성과](/hpc/compilation/stages/)들을 가능하게 합니다.

C 언어의 다음 예제를 살펴보겠습니다.

```c
int square(int x) {
    return x * x;
}

int distance(int x, int y) {
    return square(x) + square(y);
}
```

관례에 따라 함수는 인자를 `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` 순으로 받고(인자가 더 많으면 나머지는 스택에 저장), 반환 값을 `rax`에 넣은 다음 반환해야 합니다. 따라서 인자가 하나인 단순한 함수인 `square`는 다음과 같이 구현될 수 있습니다.

```nasm
square:             ; x = edi, ret = eax
    imul edi, edi
    mov  eax, edi
    ret
```

`distance`에서 이 함수를 호출할 때마다 지역 변수를 보존하는 번거로움을 거쳐야 합니다.

```nasm
distance:           ; x = rdi/edi, y = rsi/esi, ret = rax/eax
    push rdi
    push rsi
    call square     ; eax = square(x)
    pop  rsi
    pop  rdi

    mov  ebx, eax   ; x^2 저장
    mov  rdi, rsi   ; 새로운 x=y 이동

    push rdi
    push rsi
    call square     ; eax = square(x=y)
    pop  rsi
    pop  rdi

    add  eax, ebx   ; x^2 + y^2
    ret
```

이외에도 많은 미묘한 차이점들이 있지만, 이 책은 성능에 관한 책이므로 여기서는 자세히 다루지 않겠습니다. 사실 함수 호출을 처리하는 가장 좋은 방법은 애초에 호출을 피하는 것입니다.

### 인라이닝

데이터를 스택으로 주고받는 것은 이와 같은 작은 함수들에게 눈에 띄는 오버헤드를 발생시킵니다. 이렇게 해야 하는 이유는 일반적으로 피호출자가 호출자의 지역 변수가 저장된 레지스터를 수정할지 여부를 알 수 없기 때문입니다. 하지만 `square`의 코드에 접근할 수 있다면, 수정되지 않을 것으로 알고 있는 레지스터에 데이터를 저장함으로써 이 문제를 해결할 수 있습니다.

```nasm
distance:
    call square
    mov  ebx, eax
    mov  edi, esi
    call square
    add  eax, ebx
    ret
```

이것이 더 낫긴 하지만, 여전히 암시적으로 스택 메모리에 접근하고 있습니다. 각 함수 호출마다 명령어 포인터를 푸시하고 팝해야 하기 때문입니다. 이와 같이 간단한 경우, 피호출자의 코드를 호출자에게 직접 끼워 넣고 레지스터 충돌을 해결함으로써 함수 호출을 *인라이닝(inline)* 할 수 있습니다. 위 예제의 경우 다음과 같습니다.

```nasm
distance:
    imul edi, edi       ; edi = x^2
    imul esi, esi       ; esi = y^2
    add  edi, esi
    mov  eax, edi       ; "add eax, edi, esi"와 같은 명령어가 없으므로 별도의 mov가 필요함
    ret
```

이는 최적화 컴파일러가 이 코드 조각에서 생성하는 결과와 상당히 유사합니다. 다만 컴파일러는 결과 기계어 시퀀스를 몇 바이트 더 작게 만들기 위해 [lea 트릭](../assembly)을 사용합니다.

```nasm
distance:
    imul edi, edi       ; edi = x^2
    imul esi, esi       ; esi = y^2
    lea  eax, [rdi+rsi] ; eax = x^2 + y^2
    ret
```

이러한 상황에서 함수 인라이닝은 분명히 유익하며, 컴파일러는 대부분 이를 [자동으로](/hpc/compilation/situational) 수행합니다. 하지만 그렇지 않은 경우도 있는데, 이에 대해서는 [잠시 후에](../layout) 이야기하겠습니다.

### 꼬리 호출 제거

피호출자가 다른 함수 호출을 하지 않거나, 적어도 그 호출들이 재귀적이지 않을 때 인라이닝은 간단합니다. 이제 더 복잡한 예제로 넘어가 보겠습니다. 팩토리얼을 재귀적으로 계산하는 경우를 생각해 봅시다.

```cpp
int factorial(int n) {
    if (n == 0)
        return 1;
    return factorial(n - 1) * n;
}
```

이에 해당하는 어셈블리는 다음과 같습니다.

```nasm
; n = edi, ret = eax
factorial:
    test edi, edi   ; 값이 0인지 테스트
    jne  nonzero    ; ("cmp rax, 0"의 기계어는 1바이트 더 긺)
    mov  eax, 1     ; 1 반환
    ret
nonzero:
    push edi        ; 나중에 곱셈에 사용할 n 저장
    sub  edi, 1
    call factorial  ; f(n - 1) 호출
    pop  edi
    imul eax, edi
    ret
```

함수가 재귀적이더라도 구조를 재편성하여 "호출이 없는" 상태로 만드는 것이 가능한 경우가 많습니다. 이는 함수가 *꼬리 재귀(tail recursive)*인 경우, 즉 재귀 호출 직후에 바로 반환하는 경우에 해당합니다. 호출 후에 수행할 작업이 없으므로 스택에 아무것도 저장할 필요가 없으며, 재귀 호출을 시작 부분으로 점프하는 것으로 안전하게 대체할 수 있습니다. 결과적으로 함수를 루프로 바꾸는 것입니다.

`factorial` 함수를 꼬리 재귀로 만들기 위해 "현재까지의 곱" 인자를 전달할 수 있습니다.

```cpp
int factorial(int n, int p = 1) {
    if (n == 0)
        return p;
    return factorial(n - 1, p * n);
}
```

그러면 이 함수는 다음과 같이 루프로 쉽게 변환될 수 있습니다.

```nasm
; n > 0 가정
factorial:
    mov  eax, 1
loop:
    imul eax, edi
    sub  edi, 1
    jne  loop
    ret
```

재귀가 느려질 수 있는 주요 이유는 스택에 데이터를 쓰고 읽어야 하기 때문인 반면, 반복적(iterative)인 알고리즘과 꼬리 재귀 알고리즘은 그렇지 않기 때문입니다. 이 개념은 루프가 없고 오직 함수만 사용할 수 있는 함수형 프로그래밍에서 매우 중요합니다. 꼬리 호출 제거(tail call elimination)가 없다면 함수형 프로그램은 실행하는 데 훨씬 더 많은 시간과 메모리가 필요할 것입니다.
