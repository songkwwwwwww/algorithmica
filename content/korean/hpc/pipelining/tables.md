---
title: 명령어 테이블
weight: 3
---

<!-- This poses some additional challenges in coordinating how to execute the instructions — and also in which order. -->

실행 단계를 인터리빙하는 것은 디지털 전자 공학의 일반적인 아이디어이며, 메인 CPU 파이프라인뿐만 아니라 개별 명령어 및 [메모리](/hpc/cpu-cache/mlp) 수준에도 적용됩니다. 대부분의 실행 유닛은 자체적인 작은 파이프라인을 가지고 있으며, 이전 명령어 이후 단 1~2 사이클 만에 다른 명령어를 받을 수 있습니다.

이러한 맥락에서 명령어에 대해 두 가지 다른 "[비용](/hpc/complexity)"을 사용하는 것이 합리적입니다:

- *지연 시간(Latency)*: 명령어의 결과를 받기 위해 몇 사이클이 필요한지.
- *처리량(Throughput)*: 평균적으로 사이클당 몇 개의 명령어를 실행할 수 있는지.

<!-- alternative throughput definitions, maybe in scheduling? -->

[명령어 테이블(instruction tables)](https://www.agner.org/optimize/instruction_tables.pdf)이라고 불리는 특별한 문서에서 특정 아키텍처에 대한 지연 시간 및 처리량 수치를 얻을 수 있습니다. 다음은 제 Zen 2 아키텍처에 대한 샘플 값입니다 (차이가 있는 경우 모두 32비트 피연산자에 대해 지정됨):

| 명령어 | 지연 시간 | 역 처리량(RThroughput) |
|-------------|---------|:------------|
| `jmp`       | -       | 2           |
| `mov r, r`  | -       | 1/4         |
| `mov r, m`  | 4       | 1/2         |
| `mov m, r`  | 3       | 1           |
| `add`       | 1       | 1/3         |
| `cmp`       | 1       | 1/4         |
| `popcnt`    | 1       | 1/4         |
| `mul`       | 3       | 1           |
| `div`       | 13-28   | 13-28       |

몇 가지 설명:

- 우리 마음은 "많을수록" "나쁘다"는 비용 모델에 익숙해져 있기 때문에, 사람들은 주로 처리량 대신 처리량의 *역수(reciprocals)*를 사용합니다.
- 특정 명령어가 특히 빈번하게 사용된다면, 처리량을 높이기 위해 해당 실행 유닛을 복제할 수 있습니다. 가능하면 하나 이상으로 늘릴 수도 있지만, [디코드 너비(decode width)](/hpc/architecture/layout)보다 높을 수는 없습니다.
- 일부 명령어는 지연 시간이 0입니다. 이는 이러한 명령어가 스케줄러를 제어하는 데 사용되며 실행 단계에 도달하지 않음을 의미합니다. [CPU 프런트엔드](/hpc/architecture/layout)가 여전히 이를 처리해야 하므로 역 처리량은 0이 아닙니다.
- 대부분의 명령어는 파이프라이닝되어 있으며, 역 처리량이 $n$이라면 보통 해당 실행 유닛이 $n$ 사이클 후에 다른 명령어를 받을 수 있음을 의미합니다 (1 미만인 경우, 다음 사이클에 다른 명령어를 받을 수 있는 실행 유닛이 여러 개 있음을 의미함). 한 가지 눈에 띄는 예외는 [정수 나눗셈](/hpc/arithmetic/division)입니다. 이는 파이프라이닝이 매우 부실하거나 전혀 되지 않습니다.
- 일부 명령어는 크기뿐만 아니라 피연산자의 값에 따라 가변적인 지연 시간을 갖습니다. 메모리 연산(`add`와 같이 융합된 연산 포함)의 경우, 지연 시간은 보통 최선의 경우(L1 캐시 히트)에 대해 지정됩니다.

더 중요하고 세부적인 내용이 많지만, 일단 이 멘탈 모델이면 충분할 것입니다.

<!--

This mental model covers 80% of your needs.

Some instruction tables also list execution ports (or sometimes "pipes"). This is mostly relevant for SIMD.

This is a bit of an advanced and not well understood topic. Documentation is very obscure. people have to reverse engineer it. There are reasons to believe that folks at Intel don't know that themselves. The most comprehensive one is probably, uops.info.

There are tools like llvm-mca, but they aren't perfect either.

-->
