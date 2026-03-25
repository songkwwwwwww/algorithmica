---
title: 통계적 프로파일링 (Statistical Profiling)
weight: 2
---

[계측 (Instrumentation)](../instrumentation)은 특히 프로그램의 여러 작은 섹션에 관심이 있는 경우 꽤 번거로운 프로파일링 방식입니다. 도구를 통해 부분적으로 자동화할 수 있더라도, 계측 자체의 오버헤드 때문에 세밀한 통계를 수집하는 데는 도움이 되지 않습니다.

프로파일링에 대한 또 다른 덜 침습적인 접근 방식은 무작위 간격으로 프로그램 실행을 중단하고 인스트럭션 포인터(instruction pointer)가 어디에 있는지 확인하는 것입니다. 각 함수의 블록에서 포인터가 멈춘 횟수는 해당 함수를 실행하는 데 소비한 총 시간에 대략 비례합니다. 또한 [콜 스택(call stack)](/hpc/architecture/functions)을 조사하여 어떤 함수가 어떤 함수에 의해 호출되는지 확인하는 등 유용한 정보를 얻을 수도 있습니다.

원칙적으로는 `gdb`로 프로그램을 실행하고 무작위 간격으로 `ctrl+c`를 눌러 이를 수행할 수 있지만, 현대의 CPU와 운영체제는 이러한 유형의 프로파일링을 위한 전용 유틸리티를 제공합니다.

### 하드웨어 이벤트 (Hardware Events)

하드웨어 *성능 카운터(performance counters)*는 마이크로프로세서에 내장된 특수 레지스터로, 특정 하드웨어 관련 활동의 횟수를 저장할 수 있습니다. 이는 기본적으로 활성화 와이어가 연결된 바이너리 카운터에 불과하므로 마이크로칩에 추가하는 비용이 저렴합니다.

각 성능 카운터는 회로의 큰 부분 집합에 연결되어 있으며 분기 예측 실패(branch mispredict)나 캐시 미스(cache miss)와 같은 특정 하드웨어 이벤트가 발생할 때마다 증가하도록 구성할 수 있습니다. 프로그램 시작 시 카운터를 재설정하고, 프로그램을 실행한 후 마지막에 저장된 값을 출력하면 실행 중에 특정 이벤트가 발생한 정확한 횟수와 일치하게 됩니다.

또한 여러 이벤트를 다중화(multiplexing)하여 추적할 수도 있습니다. 즉, 프로그램을 균등한 간격으로 중단하고 카운터를 재구성하는 방식입니다. 이 경우 결과는 정확하지 않고 통계적인 근사치가 됩니다. 한 가지 미묘한 점은 샘플링 빈도를 단순히 높인다고 해서 정확도가 향상되지는 않는다는 것입니다. 빈도가 너무 높으면 성능에 큰 영향을 미쳐 분포가 왜곡될 수 있기 때문입니다. 따라서 여러 통계를 수집하려면 프로그램을 더 오랜 시간 동안 실행해야 합니다.

전반적으로 이벤트 기반 통계적 프로파일링은 성능 문제를 진단하는 데 가장 효과적이고 쉬운 방법입니다.

### perf를 이용한 프로파일링

위에서 설명한 이벤트 샘플링 기술에 의존하는 성능 분석 도구를 *통계적 프로파일러(statistical profilers)*라고 합니다. 많은 도구가 있지만 이 책에서 주로 사용할 도구는 리눅스 커널과 함께 제공되는 통계적 프로파일러인 [perf](https://perf.wiki.kernel.org/)입니다. 리눅스가 아닌 시스템에서는 인텔의 [VTune](https://software.intel.com/content/www/us/en/develop/tools/oneapi/components/vtune-profiler.html#gs.cuc0ks)을 사용할 수 있으며, 이는 우리의 목적에 거의 동일한 기능을 제공합니다. VTune은 무료로 제공되지만 독점 소프트웨어이며 90일마다 커뮤니티 라이선스를 갱신해야 하는 반면, perf는 자유 소프트웨어입니다.

Perf는 프로그램의 실시간 실행을 기반으로 보고서를 생성하는 명령줄 애플리케이션입니다. 소스 코드가 필요하지 않으며 여러 프로세스와 운영체제와의 상호작용을 포함하는 매우 광범위한 애플리케이션을 프로파일링할 수 있습니다.

설명을 돕기 위해 백만 개의 임의의 정수 배열을 만들고, 이를 정렬한 다음, 백만 번의 이진 탐색을 수행하는 작은 프로그램을 작성했습니다.

```c++
void setup() {
    for (int i = 0; i < n; i++)
        a[i] = rand();
    std::sort(a, a + n);
}

int query() {
    int checksum = 0;
    for (int i = 0; i < n; i++) {
        int idx = std::lower_bound(a, a + n, rand()) - a;
        checksum += idx;
    }
    return checksum;
}
```

컴파일 후(`g++ -O3 -march=native example.cc -o run`), `perf stat ./run`으로 실행하면 실행 중 기본 성능 이벤트의 횟수가 출력됩니다.

```yaml
 Performance counter stats for './run':

        646.07 msec task-clock:u               # 0.997 CPUs utilized          
             0      context-switches:u         # 0.000 K/sec                  
             0      cpu-migrations:u           # 0.000 K/sec                  
         1,096      page-faults:u              # 0.002 M/sec                  
   852,125,255      cycles:u                   # 1.319 GHz (83.35%)
    28,475,954      stalled-cycles-frontend:u  # 3.34% frontend cycles idle (83.30%)
    10,460,937      stalled-cycles-backend:u   # 1.23% backend cycles idle (83.28%)
   479,175,388      instructions:u             # 0.56  insn per cycle         
                                               # 0.06  stalled cycles per insn (83.28%)
   122,705,572      branches:u                 # 189.925 M/sec (83.32%)
    19,229,451      branch-misses:u            # 15.67% of all branches (83.47%)

   0.647801770 seconds time elapsed
   0.647278000 seconds user
   0.000000000 seconds sys
```

실행에 0.53초 또는 유효 클럭 속도 1.32 GHz에서 852M 사이클이 소요되었으며, 그동안 479M개의 인스트럭션이 실행되었음을 알 수 있습니다. 또한 122.7M개의 분기(branch)가 있었고 그 중 15.7%가 예측 실패했습니다.

`perf list`로 지원되는 모든 이벤트 목록을 볼 수 있으며, `-e` 옵션으로 원하는 특정 이벤트 목록을 지정할 수 있습니다. 예를 들어 이진 탐색을 진단할 때는 주로 캐시 미스에 관심을 가집니다.

```yaml
> perf stat -e cache-references,cache-misses ./run

91,002,054      cache-references:u                                          
44,991,746      cache-misses:u      # 49.440 % of all cache refs
```

`perf stat` 자체는 프로그램 전체에 대한 성능 카운터만 설정합니다. 총 분기 예측 실패 횟수는 알려줄 수 있지만, 그것이 *어디서* 발생하는지, 하물며 *왜* 발생하는지는 알려주지 못합니다.

이전에 논의한 "중단 후 조사(stop-the-world)" 접근 방식을 시도하려면 `perf record <cmd>`를 사용해야 합니다. 이 명령은 프로파일링 데이터를 기록하고 `perf.data` 파일로 덤프합니다. 그 다음 `perf report`를 호출하여 이를 검사합니다. 마지막 명령은 대화형이며 시각적으로 잘 구성되어 있으므로 직접 시도해 보는 것을 강력히 추천합니다. 지금 당장 할 수 없는 분들을 위해 최선을 다해 설명해 보겠습니다.

`perf report`를 호출하면 먼저 어떤 함수가 얼마나 많은 시간을 차지하고 있는지 알려주는 `top`과 유사한 대화형 보고서가 표시됩니다.

```
Overhead  Command  Shared Object        Symbol
  63.08%  run      run                  [.] query
  24.98%  run      run                  [.] std::__introsort_loop<...>
   5.02%  run      libc-2.33.so         [.] __random
   3.43%  run      run                  [.] setup
   1.95%  run      libc-2.33.so         [.] __random_r
   0.80%  run      libc-2.33.so         [.] rand
```

각 함수에 대해 총 실행 시간이 아닌 해당 함수의 *오버헤드*만 표시된다는 점에 유의하세요 (예: `setup`은 `std::__introsort_loop`를 포함하지만 자체 오버헤드만 3.43%로 집계됨). perf 보고서를 더 명확하게 만들기 위해 [플레임 그래프(flame graphs)](https://www.brendangregg.com/flamegraphs.html)를 생성하는 도구들이 있습니다. 또한 인라이닝(inlining) 가능성을 고려해야 하는데, 여기서는 `std::lower_bound`에서 인라이닝이 발생한 것으로 보입니다. Perf는 공유 라이브러리(예: `libc`)와 일반적으로 생성된 다른 모든 프로세스도 추적합니다. 원한다면 perf로 웹 브라우저를 실행하여 내부에서 어떤 일이 일어나는지 볼 수도 있습니다.

다음으로, 이 함수들 중 하나를 "확대(zoom in)"할 수 있으며, 다른 기능들과 함께 관련 히트맵이 표시된 어셈블리를 보여줍니다. 예를 들어, `query`의 어셈블리는 다음과 같습니다.

```asm
       │20: → call   rand@plt
       │      mov    %r12,%rsi
       │      mov    %eax,%edi
       │      mov    $0xf4240,%eax
       │      nop    
       │30:   test   %rax,%rax
  4.57 │    ↓ jle    52
       │35:   mov    %rax,%rdx
  0.52 │      sar    %rdx
  0.33 │      lea    (%rsi,%rdx,4),%rcx
  4.30 │      cmp    (%rcx),%edi
 65.39 │    ↓ jle    b0
  0.07 │      sub    %rdx,%rax
  9.32 │      lea    0x4(%rcx),%rsi
  0.06 │      dec    %rax
  1.37 │      test   %rax,%rax
  1.11 │    ↑ jg     35
       │52:   sub    %r12,%rsi
  2.22 │      sar    $0x2,%rsi
  0.33 │      add    %esi,%ebp
  0.20 │      dec    %ebx
       │    ↑ jne    20
```

왼쪽 열은 인스트럭션 포인터가 특정 라인에서 멈춘 비율입니다. 점프 인스트럭션 이전에 비교 연산자가 있으므로 제어 흐름이 해당 비교가 결정될 때까지 기다리고 있음을 나타내며, 여기서 약 65%의 시간을 소비하고 있음을 볼 수 있습니다.

[파이프라이닝(pipelining)](/hpc/pipelining) 및 비순차적 실행(out-of-order execution)과 같은 복잡성 때문에 현대 CPU에서 "지금"이라는 개념은 명확하게 정의되지 않습니다. 따라서 인스트럭션 포인터가 약간 앞으로 밀려나 데이터가 다소 부정확할 수 있습니다. 인스트럭션 수준의 데이터는 여전히 유용하지만, 개별 사이클 수준에서는 [더 정밀한 도구](../simulation)로 전환해야 합니다.

<!-- flame graphs -->
