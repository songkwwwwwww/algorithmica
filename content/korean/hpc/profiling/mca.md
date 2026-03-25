---
title: 머신 코드 분석기 (Machine Code Analyzers)
weight: 4
---

*머신 코드 분석기 (machine code analyzer)*는 작은 어셈블리 코드 스니펫을 입력으로 받아 컴파일러가 사용할 수 있는 정보를 사용하여 특정 마이크로아키텍처에서의 실행을 [시뮬레이션](../simulation)하는 프로그램입니다. 이 도구는 전체 블록의 지연 시간(latency)과 처리량(throughput)뿐만 아니라, CPU 내부의 다양한 자원 활용도를 사이클 단위로 정확하게 출력합니다.

### `llvm-mca` 사용하기

다양한 머신 코드 분석기가 있지만, 저는 개인적으로 `clang`과 함께 패키지 관리자를 통해 설치할 수 있는 `llvm-mca`를 선호합니다. 또한 [UICA](https://uica.uops.info)라는 웹 기반 도구 시스템이나 [Compiler Explorer](https://godbolt.org/)에서 언어를 "Analysis"로 선택하여 접근할 수도 있습니다.

`llvm-mca`는 주어진 어셈블리 스니펫을 정해진 횟수만큼 반복 실행하고 각 인스트럭션의 자원 사용량에 대한 통계를 계산합니다. 이는 병목 지점이 어디인지 찾는 데 유용합니다.

간단한 예로 배열 합계를 살펴보겠습니다.

```asm
loop:
    addl (%rax), %edx
    addq $4, %rax
    cmpq %rcx, %rax
    jne  loop
```

다음은 Skylake 마이크로아키텍처에 대한 `llvm-mca` 분석 결과입니다.

```yaml
Iterations:        100
Instructions:      400
Total Cycles:      108
Total uOps:        500

Dispatch Width:    6
uOps Per Cycle:    4.63
IPC:               3.70
Block RThroughput: 0.8
```

먼저 루프와 하드웨어에 대한 일반적인 정보를 출력합니다.

- 루프를 100번 "실행"하여 총 108 사이클 동안 400개의 인스트럭션을 실행했습니다. 이는 평균적으로 $\frac{400}{108} \approx 3.7$ [사이클당 인스트럭션 (IPC)](/hpc/complexity/hardware)을 실행한 것과 같습니다.
- CPU는 이론적으로 사이클당 최대 6개의 인스트럭션을 실행할 수 있습니다 ([디스패치 폭 (dispatch width)](/hpc/architecture/layout)).
- 각 루프 주기는 이론적으로 평균 0.8 사이클 내에 실행될 수 있습니다 ([블록 역 처리량 (block reciprocal throughput)](/hpc/pipelining/tables)).
- 여기서 "uOps"는 CPU가 각 인스트럭션을 분할한 마이크로 연산(micro-operations)을 의미합니다(예: 로드-더하기가 결합된 인스트럭션은 두 개의 uOp으로 구성됨).

그 다음 각 개별 인스트럭션에 대한 정보를 제공합니다.

```yaml
Instruction Info:
[1]: uOps
[2]: Latency
[3]: RThroughput
[4]: MayLoad
[5]: MayStore
[6]: HasSideEffects (U)

[1]    [2]    [3]    [4]    [5]    [6]    Instructions:
 2      6     0.50    *                   addl  (%rax), %edx
 1      1     0.25                        addq  $4, %rax
 1      1     0.25                        cmpq  %rcx, %rax
 1      1     0.50                        jne   -11
```

이 정보들은 [인스트럭션 테이블](/hpc/pipelining/tables)에 있는 내용과 동일합니다.

- 각 인스트럭션이 몇 개의 uOp으로 분할되는지.
- 각 인스트럭션이 완료되는 데 걸리는 사이클 수 (지연 시간, latency).
- 여러 복사본이 동시에 실행될 수 있음을 고려할 때, 분할 상환된 의미에서 각 인스트럭션이 완료되는 데 걸리는 사이클 수 (역 처리량, reciprocal throughput).

마지막으로 가장 중요한 부분인 어떤 인스트럭션이 언제 어디서 실행되는지를 출력합니다.

```yaml
Resource pressure by instruction:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]    [8]    [9]    Instructions:
 -      -     0.01   0.98   0.50   0.50    -      -     0.01    -     addl (%rax), %edx
 -      -      -      -      -      -      -     0.01   0.99    -     addq $4, %rax
 -      -      -     0.01    -      -      -     0.99    -      -     cmpq %rcx, %rax
 -      -     0.99    -      -      -      -      -     0.01    -     jne  -11
```

실행 포트(execution ports)에 대한 경합이 [구조적 해저드 (structural hazards)](/hpc/pipelining/hazards)를 유발하므로, 처리량 중심의 루프에서는 포트가 병목 지점이 되는 경우가 많습니다. 이 차트는 그 원인을 진단하는 데 도움이 됩니다. 사이클 단위의 완벽한 간트 차트(Gantt chart) 같은 것을 제공하지는 않지만, 각 인스트럭션에 사용된 실행 포트의 집계 통계를 제공하여 어떤 포트가 과부하 상태인지 찾을 수 있게 해줍니다.

<!--

CPU는 매우 복잡한 장치이지만, 본질적으로는 특정 종류의 인스트럭션을 전문적으로 처리하는 여러 "포트"가 있습니다. 이러한 포트들이 병목 지점이 되는 경우가 많으며, 위의 차트는 그 이유를 진단하는 데 도움이 됩니다.

아직 이것이 어떻게 작동하는지 논의할 준비가 되지 않았지만, 마지막 장에서 자세히 다룰 예정입니다.

-->
