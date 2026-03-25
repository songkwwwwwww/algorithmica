---
title: 분기 없는 프로그래밍
weight: 3
---

[이전 섹션](../branching)에서 확인했듯이, CPU가 효과적으로 예측할 수 없는 분기는 분기 예측 오류 발생 시 새로운 명령어를 가져오기 위해 긴 파이프라인 스톨(stall)을 유발할 수 있으므로 비용이 많이 듭니다. 이 섹션에서는 우선 분기를 제거하는 방법에 대해 논의합니다.

### 조건부 실행 (Predication)

이전에 시작한 것과 동일한 사례 연구를 계속하겠습니다. 랜덤 숫자 배열을 만들고 50보다 작은 모든 요소를 합산합니다:

```c++
for (int i = 0; i < N; i++)
    a[i] = rand() % 100;

volatile int s;

for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];
```

우리의 목표는 `if` 문으로 인해 발생하는 분기를 제거하는 것입니다. 다음과 같이 제거를 시도할 수 있습니다:

```c++
for (int i = 0; i < N; i++)
    s += (a[i] < 50) * a[i];
```

이제 루프는 원래의 약 14 사이클 대신 요소당 약 7 사이클이 소요됩니다. 또한 `50`을 다른 임계값으로 변경해도 성능이 일정하게 유지되므로 분기 확률에 의존하지 않습니다.

하지만 잠깐만요... 여전히 분기가 있어야 하지 않을까요? `(a[i] < 50)`이 어셈블리로 어떻게 매핑될까요?

어셈블리에는 불리언(Boolean) 타입도 없고, 비교 결과에 따라 1 또는 0을 반환하는 명령어도 없지만, 다음과 같이 간접적으로 계산할 수 있습니다: `(a[i] - 50) >> 31`. 이 트릭은 [정수의 이진 표현](/hpc/arithmetic/integer)에 의존합니다. 구체적으로, 식 `a[i] - 50`이 음수이면(`a[i] < 50`임을 의미), 결과의 최상위 비트가 1로 설정되며, 이를 우측 시프트(right-shift)를 사용하여 추출할 수 있다는 사실을 이용합니다.

```nasm
mov  ebx, eax   ; t = x
sub  ebx, 50    ; t -= 50
sar  ebx, 31    ; t >>= 31
imul  eax, ebx   ; x *= t
```

이 전체 시퀀스를 구현하는 또 다른 더 복잡한 방법은 이 부호 비트를 마스크로 변환한 다음 곱셈 대신 비트 연산 `and`를 사용하는 것입니다: `((a[i] - 50) >> 31 - 1) & a[i]`. `imul`은 다른 명령어와 달리 3 사이클이 소요된다는 점을 고려하면, 이 방식은 전체 시퀀스를 1 사이클 더 빠르게 만듭니다:

```nasm
mov  ebx, eax   ; t = x
sub  ebx, 50    ; t -= 50
sar  ebx, 31    ; t >>= 31
; imul  eax, ebx ; x *= t
sub  ebx, 1     ; t -= 1 (t = 0일 경우 언더플로우 발생)
and  eax, ebx   ; x &= t
```

이 최적화는 컴파일러의 관점에서 기술적으로 정확하지 않다는 점에 유의하세요. 표현 가능한 가장 낮은 50개의 정수($[-2^{31}, -2^{31} + 49]$ 범위)에 대해서는 언더플로우로 인해 결과가 틀릴 수 있습니다. 우리는 모든 숫자가 0에서 100 사이라는 것을 알고 있으므로 이런 일이 발생하지 않겠지만, 컴파일러는 이를 모릅니다.

하지만 컴파일러는 실제로 다른 방식을 선택합니다. 이러한 산술 트릭을 사용하는 대신, (점프와 동일한 방식으로 플래그 레지스터를 사용하여 계산되고 확인되는) 조건에 따라 값을 할당하는 특수 명령어 `cmov`("conditional move")를 사용합니다:

```nasm
mov     ebx, 0      ; cmov는 즉시값(immediate values)을 지원하지 않으므로 0 레지스터가 필요함
cmp     eax, 50
cmovge  eax, ebx    ; eax = (eax >= 50 ? eax : ebx=0)
```

따라서 위의 코드는 실제로는 다음과 같이 삼항 연산자를 사용하는 것에 더 가깝습니다:

```c++
for (int i = 0; i < N; i++)
    s += (a[i] < 50 ? a[i] : 0);
```

두 변형 모두 컴파일러에 의해 최적화되어 다음과 같은 어셈블리를 생성합니다:

```nasm
    mov     eax, 0
    mov     ecx, -4000000
loop:
    mov     esi, dword ptr [rdx + a + 4000000]  ; a[i] 로드
    cmp     esi, 50
    cmovge  esi, eax                            ; esi = (esi >= 50 ? esi : eax=0)
    add     dword ptr [rsp + 12], esi           ; s += esi
    add     rdx, 4
    jnz     loop                                ; "rdx가 0이 아닐 동안 반복"
```

이 일반적인 기술을 *조건부 실행(predication)*이라고 하며, 대략 다음의 대수적 트릭과 동일합니다:

$$
x = c \cdot a + (1 - c) \cdot b
$$

이러한 방식으로 분기를 제거할 수 있지만, 이는 *두* 분기 모두를 평가하는 비용과 `cmov` 자체의 비용을 수반합니다. ">=" 분기를 평가하는 데 비용이 들지 않기 때문에, 성능은 분기 버전의 ["항상 예" 케이스](../branching/#branch-prediction)와 정확히 동일합니다.

### 조건부 실행이 유익한 경우

조건부 실행을 사용하면 [제어 해저드(control hazard)](../hazards)를 제거할 수 있지만 데이터 해저드(data hazard)가 발생합니다. 여전히 파이프라인 스톨은 존재하지만 더 저렴한 것입니다. `cmov`가 해결될 때까지 기다리기만 하면 되며, 예측 오류 시 전체 파이프라인을 플러시(flush)할 필요가 없기 때문입니다.

그러나 분기 코드를 그대로 두는 것이 더 효율적인 상황도 많습니다. 단 한 쪽의 분기만 실행하는 대신 *두* 분기를 모두 계산하는 비용이 잠재적인 분기 예측 오류로 인한 페널티보다 클 때가 그렇습니다.

우리의 예에서 분기 코드는 분기를 약 75% 이상의 확률로 예측할 수 있을 때 승리합니다.

![](../img/branchy-vs-branchless.svg)

이 75% 임계값은 컴파일러가 `cmov`를 사용할지 여부를 결정하는 휴리스틱으로 흔히 사용됩니다. 불행히도 이 확률은 대개 컴파일 타임에 알 수 없으므로 다음 중 한 가지 방법으로 제공되어야 합니다:

- [프로파일 기반 최적화(PGO, profile-guided optimization)](/hpc/compilation/situational/#profile-guided-optimization)를 사용하여 조건부 실행 사용 여부를 스스로 결정하게 할 수 있습니다.
- [가능성 속성(likeliness attributes)](../branching#hinting-likeliness-of-branches) 및 [컴파일러 전용 인트린직(intrinsics)](/hpc/compilation/situational)을 사용하여 분기 가능성을 힌트로 줄 수 있습니다: GCC의 `__builtin_expect_with_probability` 및 Clang의 `__builtin_unpredictable`.
- 삼항 연산자나 다양한 산술 트릭을 사용하여 분기 코드를 다시 작성할 수 있습니다. 이는 프로그래머와 컴파일러 사이의 암묵적인 약속 역할을 합니다. 즉, 프로그래머가 코드를 이런 식으로 작성했다면 아마도 분기가 없기를 의도했을 가능성이 높습니다.

"올바른 방법"은 분기 힌트를 사용하는 것이지만, 불행히도 이에 대한 지원이 부족합니다. 현재 [이러한 힌트들은 컴파일러 백엔드가 `cmov`가 더 유익한지 결정할 때쯤이면 손실되는 것 같습니다](https://bugs.llvm.org/show_bug.cgi?id=40027). 이를 가능하게 하려는 [일부 진전](https://discourse.llvm.org/t/rfc-cmov-vs-branch-optimization/6040)이 있지만, 현재 컴파일러가 분기 없는 코드를 생성하도록 강제하는 좋은 방법은 없으므로 때로는 어셈블리로 작은 스니펫을 작성하는 것이 최선의 희망일 수 있습니다.

<!--

Because this is very architecture-specific.

in the absence of branch likeliness hints

While any program that uses a ternary operator is equivalent to a program that uses an `if` statement

The codes seem equivalent. My guess is that the compiler doesn't know that `s + a[i]` does not cause integer overflow.

(The compiler can't optimize it because it's technically [not allowed to](/hpc/compilation/contracts): despite `y - x` being valid, `x - y` could over/underflow, causing undefined behavior. Although fully correct, I guess the compiler just doesn't date executing it.)

Branchless computing tricks like this one are especially important in all sorts of parallel algorithms.

The `cmov` variant doesn't care about probabilities of branches. It only wins if the branch probability if 75% chance, which usually is the heuristic threshold set in compilers.

This is a legal optimization, but I guess an implicit contract has evolved between application programmers and compiler engineers that if you write a ternary operator, then you kind of telling that it is likely going to be an unpredictable branch.

The general technique is called *branchless* or *branch-free* programming. Predication is the main tool of it, but there are more complicated ways.

-->

<!--

Let's do a few more examples as an exercise.

```c++
int max(int a, int b) {
    return (a > b) * a + (a <= b) * b;
}
```

```c++
int max(int a, int b) {
    return (a > b ? a : b);
}
```


```c++
int abs(int a, int b) {
    return max(diff, -diff);
}
```

```c++
int abs(int a, int b) {
    int diff = a - b;
    return (diff < 0 ? -diff : diff);
}
```

```c++
int abs(int a) {
    return (a > 0 ? a : -a);
}
```

```c++
int abs(int a) {
    int mask = a >> 31;
    a ^= mask;
    a -= mask;
    return a;
}
```

-->

### 더 큰 예제들

**문자열.** 단순화하자면, `std::string`은 힙 어딘가에 할당된 널 종료 `char` 배열(C-문자열이라고도 함)에 대한 포인터와 문자열 크기를 포함하는 하나의 정수로 구성됩니다.

문자열의 흔한 값은 빈 문자열이며, 이는 기본값이기도 합니다. 이를 어떻게든 처리해야 하며, 관용적인 접근 방식은 포인터에 `nullptr`을, 문자열 크기에 `0`을 할당한 다음, 문자열과 관련된 모든 프로시저의 시작 부분에서 포인터가 널인지 또는 크기가 0인지 확인하는 것입니다.

그러나 이는 별도의 분기를 필요로 하며, (대부분의 문자열이 비어 있거나 비어 있지 않은 경우가 아니라면) 비용이 많이 듭니다. 이러한 검사와 분기를 제거하기 위해 어딘가에 할당된 단일 0 바이트인 "제로 C-문자열"을 할당한 다음, 모든 빈 문자열이 그곳을 가리키도록 할 수 있습니다. 이제 빈 문자열에 대한 모든 문자열 연산은 이 쓸모없는 0 바이트를 읽어야 하지만, 이는 분기 예측 오류보다는 훨씬 저렴합니다.

**이진 검색.** 표준 이진 검색은 [분기 없이 구현될 수 있으며](/hpc/data-structures/binary-search), (캐시에 들어가는) 작은 배열에서는 분기가 있는 `std::lower_bound`보다 약 4배 더 빠르게 작동합니다:

```c++
int lower_bound(int x) {
    int *base = t, len = n;
    while (len > 1) {
        int half = len / 2;
        base += (base[half - 1] < x) * half; // "cmov"로 대체됨
        len -= half;
    }
    return *base;
}
```

더 복잡하다는 점 외에도, 잠재적으로 더 많은 비교를 수행하고(상수 $\lceil \log_2 n \rceil$ 회 수행) 미래의 메모리 읽기를 투기적으로 수행할 수 없다(프리페칭 역할을 하므로 매우 큰 배열에서는 손해를 봄)는 사소한 단점이 있습니다.

일반적으로 데이터 구조는 연산이 일정한 수의 반복을 수행하도록 암시적 또는 명시적으로 *패딩(padding)*을 추가함으로써 분기 없이 만들어집니다. 더 복잡한 예제는 [해당 기사](/hpc/data-structures/binary-search)를 참조하세요.

<!--

The only downside of the branchless implementation is that it potentially does more memory reads: 

There are typically two ways to achieve this:

And in general, data structures can be "padded" to be made constant size or height.

That there are no substantial reasons why compilers can't do this on their own, but unfortunately this is just how it is right now.

-->

**데이터 병렬 프로그래밍.** 분기 없는 프로그래밍은 [SIMD](/hpc/simd) 애플리케이션에서 매우 중요합니다. SIMD에는 애초에 분기 기능이 없기 때문입니다.

배열 합계 예제에서 누산기에서 `volatile` 타입 한정자를 제거하면 컴파일러가 루프를 [벡터화](/hpc/simd/auto-vectorization)할 수 있습니다:

```c++
/* volatile */ int s = 0;

for (int i = 0; i < N; i++)
    if (a[i] < 50)
        s += a[i];
```

이제 요소당 약 0.3 사이클로 작동하며, 이는 주로 [메모리 병목 현상](/hpc/cpu-cache/bandwidth) 때문입니다.

컴파일러는 일반적으로 분기나 반복 간의 의존성이 없는 루프를 벡터화할 수 있습니다. 또한 [리덕션(reductions)](/hpc/simd/reduction)이나 else 없는 단순한 if 문을 포함하는 특정 루프들도 가능합니다. 그보다 복잡한 것을 벡터화하는 것은 매우 난해한 문제이며, [마스킹(masking)](/hpc/simd/masking) 및 [레지스터 내 순열(in-register permutations)](/hpc/simd/shuffling)과 같은 다양한 기술이 필요할 수 있습니다.

<!--

**Binary exponentiation.** However, when it is constant

When we can iterate in small batches, [autovectorization](/hpc/simd/autovectorization) speeds it up 13x.

-->
