---
title: 현대 하드웨어 (Modern Hardware)
weight: 1
ignoreIndexing: true
---

1960년대 슈퍼컴퓨터의 주요 단점은 속도가 느리다는 것이 아니라(상대적으로는 느리지 않았습니다), 거대하고 사용하기 복잡하며 너무 비싸서 세계 초강대국 정부만이 감당할 수 있었다는 점이었습니다. 크기가 비싼 이유였습니다. 매크로 세계에서 전기 공학 학위를 가진 사람들이 매우 정밀하게 조립해야 하는 수많은 맞춤형 부품이 필요했기 때문이며, 이는 대량 생산을 위해 확장할 수 없는 프로세스였습니다.

전환점은 단일하고 작으며 완전한 회로인 *마이크로칩*의 개발이었습니다. 이는 산업을 혁명적으로 변화시켰으며 아마도 20세기에서 가장 중요한 발명품이 되었을 것입니다. 1965년에 수백만 달러짜리 찬장만 했던 컴퓨팅 기계는 1975년에 25달러면 살 수 있는 [4mm × 4mm 크기의 실리콘 조각](https://en.wikipedia.org/wiki/MOS_Technology_6502)[^size] 위에 올라가게 되었습니다. 이러한 경제성의 비약적인 향상은 이후 10년 동안 Apple II, Atari 2600, Commodore 64, IBM PC와 같은 컴퓨터가 대중에게 보급되면서 홈 컴퓨터 혁명을 일으켰습니다.

[^size]: 실제 CPU 크기는 전력 관리, 열 발산, 그리고 과도한 욕설 없이 메인보드에 장착해야 할 필요성 때문에 센티미터 단위입니다.

### 마이크로칩이 만들어지는 방법 (How Microchips are Made)

마이크로칩은 [포토리소그래피(photolithography)](https://en.wikipedia.org/wiki/Photolithography)라고 불리는 공정을 사용하여 결정질 실리콘 조각 위에 "인쇄"됩니다. 이 공정은 다음 단계를 포함합니다:

1. [매우 순수한 실리콘 결정(wafer)](https://en.wikipedia.org/wiki/Wafer_(electronics))을 성장시키고 얇게 자름,
2. [광자가 닿으면 녹는 물질(photoresist)](https://en.wikipedia.org/wiki/Photoresist) 층으로 덮음,
3. 정해진 패턴으로 광자를 쏨,
4. 노출된 부분을 화학적으로 [식각(etching)](https://en.wikipedia.org/wiki/Etching_(microfabrication))함,
5. 남은 감광액(photoresist)을 제거함,

...그리고 CPU의 나머지 부분을 완성하기 위해 몇 달에 걸쳐 또 다른 40~50단계의 과정을 거칩니다.

![](../img/lithography.png)

이제 "광자를 쏘는" 부분을 생각해 봅시다. 이를 위해 패턴을 훨씬 더 작은 영역에 투사하는 렌즈 시스템을 사용할 수 있으며, 이를 통해 원하는 모든 특성을 가진 미세 회로를 효과적으로 만들 수 있습니다. 이런 방식으로 1970년대의 광학 기술은 손톱만한 크기에 수천 개의 트랜지스터를 넣을 수 있었고, 이는 마이크로칩에 매크로 세계의 컴퓨터가 갖지 못한 몇 가지 주요 이점을 제공했습니다:

- 더 높은 클록 속도 (이전에는 빛의 속도에 의해 제한되었음);
- 생산 확장 능력;
- 훨씬 낮은 재료 및 전력 사용량으로 인한 단위당 비용 절감.

이러한 즉각적인 이점 외에도 포토리소그래피는 성능을 더욱 향상시킬 수 있는 명확한 경로를 열어주었습니다. 렌즈를 더 강력하게 만들기만 하면 비교적 적은 노력으로 기능적으로 동일하지만 더 작은 소자를 만들 수 있었기 때문입니다.

### 데너드 스케일링 (Dennard Scaling)

마이크로칩의 크기를 줄일 때 어떤 일이 일어나는지 생각해 보십시오. 회로가 작아지면 재료가 비례해서 적게 들고, 트랜지스터가 작아지면 스위칭 시간이 짧아져(칩 내의 다른 모든 물리적 프로세스와 함께) 전압을 낮추고 클록 속도를 높일 수 있습니다.

*데너드 스케일링(Dennard scaling)*으로 알려진 더 상세한 관찰에 따르면, 트랜지스터 크기를 30% 줄이면

- 트랜지스터 밀도가 두 배가 되고 ($0.7^2 \approx 0.5$),
- 클록 속도가 40% 증가하며 ($\frac{1}{0.7} \approx 1.4$),
- 전체적인 *전력 밀도*는 일정하게 유지됩니다.

단위당 제조 비용은 면적의 함수이고 운영 비용은 주로 전력 비용[^power]이므로, 새로운 "세대"마다 총 비용은 거의 동일하면서도 클록은 40% 더 빠르고 트랜지스터는 두 배 더 많아야 합니다. 이는 예를 들어 새로운 명령어를 추가하거나 워드 크기를 늘려 메모리 마이크로칩에서 일어나는 소형화 속도에 맞추는 데 즉시 사용될 수 있습니다.

[^power]: 바쁜 서버를 2~3년 동안 운영하는 전기 비용은 칩 자체를 만드는 비용과 거의 맞먹습니다.

설계 중에 할 수 있는 에너지와 성능 간의 절충과, 트랜지스터 밀도로 직결되는 "180nm"나 "65nm"와 같은 제조 공정의 정밀도는 CPU 효율성의 상징이 되었습니다[^fidelity].

[^fidelity]: 무어의 법칙이 둔화되기 시작한 어느 시점부터 칩 제조사들은 부품의 실제 크기로 칩을 구분하는 것을 중단했습니다. 이제는 마케팅 용어에 가깝습니다. [특별 위원회](https://en.wikipedia.org/wiki/International_Technology_Roadmap_for_Semiconductors)가 2년마다 회의를 열어 이전 노드 이름을 가져와 루트 2로 나누고 반올림하여 새로운 노드 이름을 선언한 다음 와인을 잔뜩 마십니다. "nm"은 이제 더 이상 나노미터를 의미하지 않습니다.

컴퓨팅 역사의 대부분 동안 광학적 축소는 성능 향상의 주요 원동력이었습니다. 인텔의 전 CEO인 고든 무어(Gordon Moore)는 1975년에 마이크로프로세서의 트랜지스터 수가 2년마다 두 배로 늘어날 것이라고 예측했습니다. 그의 예측은 오늘날까지 유효하며 *무어의 법칙(Moore's law)*으로 알려지게 되었습니다.

![](../img/dennard.ppm)

데너드 스케일링과 무어의 법칙은 실제 물리 법칙이 아니라 노련한 엔지니어들의 관찰 결과일 뿐입니다. 둘 다 근본적인 물리적 한계로 인해 어느 시점에서는 멈출 수밖에 없으며, 궁극적인 한계는 실리콘 원자의 크기입니다. 실제로 데너드 스케일링은 전력 문제로 인해 이미 멈췄습니다.

열역학적으로 컴퓨터는 전력을 열로 변환하는 매우 효율적인 장치일 뿐입니다. 이 열은 결국 제거되어야 하는데, 밀리미터 단위의 결정에서 방출할 수 있는 전력에는 물리적 한계가 있습니다. 성능 극대화를 목표로 하는 컴퓨터 엔지니어들은 기본적으로 전체 전력 소비가 동일하게 유지되도록 가능한 최대 클록 속도를 선택합니다. 트랜지스터가 작아지면 커패시턴스(capacitance)가 작아져서 상태를 바꾸는 데 필요한 전압이 낮아지고, 이는 결과적으로 클록 속도를 높일 수 있게 해줍니다.

2005~2007년경, 이러한 전략은 *누설(leakage)* 효과 때문에 더 이상 작동하지 않게 되었습니다. 회로의 특징들이 너무 작아져서 자기장이 인접한 회로의 전자들을 엉뚱한 방향으로 움직이게 만들기 시작했고, 이는 불필요한 열 발생과 간헐적인 비트 플립(bit flipping)을 유발했습니다.

이를 완화하는 유일한 방법은 전압을 높이는 것이지만, 전력 소비의 균형을 맞추기 위해 클록 주파수를 낮춰야 합니다. 이는 트랜지스터 밀도가 높아짐에 따라 전체 프로세스의 이득을 점진적으로 줄어들게 만듭니다. 어느 시점부터는 스케일링을 통해 클록 속도를 더 이상 높일 수 없게 되었고, 소형화 추세는 둔화되기 시작했습니다.

<!--

### Power Efficiency

It may come as a surprise, but the primary metric for modern CPUs is not the clock frequency, but rather "useful operations per joule," or, more practically put, "useful operations per dollar."

Thermodynamically, a computer is just a very efficient device for converting electrical power into heat. This heat eventually needs to be removed, and it's not straightforward to do when you are working with a millimeter-scale crystal. There are physical limits to how much power you can consume and then dissipate.

Historically, the three main variables guiding microchip designs are power, performance, and area (PPA), commonly defined in watts, hertz, and nanometers. Until ~2005, cost, which was mainly a function of area, and performance, used to be the most important criteria. But as battery-driven mobile devices started replacing PCs, power quickly and firmly moved up on top of the list, followed by cost and performance.

Leakage: interfering magnetic fields make electrons move in the directions they are not supposed to and cause unnecessary heating. It isn't bad by itself: to mitigate it you need to increase the voltage, and it won't flick any bits. But the problem is that the smaller a circuit is, the harder it is to cope with this by isolating the wires. So modern chips keep the clock frequency at a level that won't cause overheat, although physically there aren't other reasons why they shouldn't.

-->

### 현대 컴퓨팅 (Modern Computing)

데너드 스케일링은 끝났지만, 무어의 법칙은 아직 죽지 않았습니다.

클록 속도는 정체되었지만 트랜지스터 수는 여전히 증가하고 있으며, 이는 새롭고 *병렬적인* 하드웨어의 생성을 가능하게 합니다. 더 빠른 사이클을 쫓는 대신, CPU 설계는 단일 사이클에서 더 많은 유용한 일을 처리하는 데 집중하기 시작했습니다. 트랜지스터는 작아지는 대신 모양이 변하고 있습니다.

그 결과 단일 사이클에 수십, 수백, 혹은 수천 가지의 서로 다른 일을 할 수 있는 점점 더 복잡한 아키처가 탄생했습니다.

![AMD의 Zen CPU 코어 다이 샷 (약 1,400,000,000개의 트랜지스터)](../img/die-shot.jpg)

다음은 더 많은 트랜지스터를 활용하여 최근 컴퓨터 설계를 주도하고 있는 핵심적인 접근 방식들입니다:

- CPU의 여러 부분이 계속 바쁘게 움직이도록 명령어 실행을 중첩시킴 (파이프라이닝);
- 이전 명령어의 완료를 반드시 기다리지 않고 연산을 실행함 (추측 실행 및 비순차 실행);
- 독립적인 연산을 동시에 처리하기 위해 여러 실행 유닛을 추가함 (슈퍼스칼라 프로세서);
- 머신 워드 크기를 늘려, 여러 그룹으로 나뉜 128, 256, 또는 512비트 데이터 블록에 대해 동일한 연산을 실행할 수 있는 명령어를 추가함 ([SIMD](/hpc/simd/));
- [RAM 및 외부 메모리](/hpc/external-memory/) 액세스 시간을 단축하기 위해 칩에 [캐시 계층](/hpc/cpu-cache/)을 추가함 (메모리는 실리콘 스케일링 법칙을 그대로 따르지 않음);
- 칩에 동일한 코어를 여러 개 추가함 (병렬 컴퓨팅, GPU);
- 메인보드에 여러 개의 칩을 사용하고 데이터 센터에 여러 대의 저렴한 컴퓨터를 사용함 (분산 컴퓨팅);
- 특정 문제를 더 나은 칩 활용도로 해결하기 위해 맞춤형 하드웨어를 사용함 (ASIC, FPGA).

현대 컴퓨터에서 알고리즘 성능을 예측하기 위해 "[모든 연산을 세어보자](../)"는 식의 접근 방식은 단순히 약간 틀린 수준이 아니라 수십 배(orders of magnitude)나 차이가 납니다. 이는 새로운 계산 모델과 알고리즘 성능을 평가하는 다른 방법들을 필요로 합니다.
