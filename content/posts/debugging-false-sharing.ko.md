+++
title = 'Debugging False Sharing'
date = '2025-12-12T10:52:00+09:00'
draft = false
translationKey = 'debugging-false-sharing'
slug = 'debugging-false-sharing'
description = '넷플릭스가 CPU를 3배로 늘리고도 성능이 25%밖에 개선되지 않았던 사건을, PMC 분석으로 False Sharing을 찾아내고 해결한 과정을 정리합니다.'
tags = ['JVM', 'CPU', 'False Sharing', 'Performance']
categories = ['Performance', 'Systems']

[cover]
image = 'images/debugging-false-sharing/image4.png'
hiddenInSingle = true
+++

> 이 글은 넷플릭스 테크 블로그의 [Seeing through hardware counters: a journey to threefold performance increase](https://netflixtechblog.com/seeing-through-hardware-counters-a-journey-to-threefold-performance-increase-2721924a2822)를 요약하고, 관련 배경 지식을 제 나름대로 정리해 덧붙인 글입니다.

## 문제 발생

넷플릭스가 운영하던 어떤 서비스가 CPU 부족을 겪어서, 노드 스펙을 3배로 늘리는 scale-up을 진행했습니다. CPU-intensive한 워크로드였으니 처리량도 그에 비례해서 늘어날 거라 기대했지만, 실제로 개선된 폭은 25% 남짓이었습니다. 심지어 tail latency는 오히려 더 나빠졌습니다.

![](/images/debugging-false-sharing/image1.png)

이상하다 싶어 더 들여다보니, 노드마다 CPU 부하가 크게 갈리는 현상을 발견했습니다. 일부 노드(약 15%)는 빠르게 처리하는데, 대다수 노드(약 85%)는 눈에 띄게 느렸습니다. 워크로드는 라운드로빈으로 균등하게 분배되고 있었으니, 이론상으로는 모든 노드의 처리량이 비슷해야 정상입니다.

![](/images/debugging-false-sharing/image2.png)

## 원인을 찾아서

처음에는 JVM profiling, JFR(Java Flight Recorder), JIT 컴파일러 분석 같은 익숙한 도구들을 동원했지만 별다른 소득이 없었습니다. 결국 한 단계 더 아래로 내려가 CPU metric과 PMC(Performance Monitoring Counter)를 들여다보고서야 실마리를 찾았습니다.

느린 노드는 빠른 노드보다 CPI(Cycle per Instruction)가 3배 가까이 높았습니다. CPI가 이렇게 튄다는 건 CPU stall이 빈번하게 일어난다는 신호입니다. L1, L3 캐시의 부하도 훨씬 컸는데, 이는 캐시 일관성으로 인한 cache miss가 많다는 뜻입니다. MACHINE_CLEAR 카운터 역시 자주 발생했습니다.

![](/images/debugging-false-sharing/image3.png)

이 조합, 즉 높은 CPI와 늘어난 캐시 부하는 False Sharing의 전형적인 증상입니다.

## False Sharing이란

CPU는 메모리에서 특정 주소를 읽을 때 항상 고정된 크기(보통 64바이트) 단위로 데이터를 가져옵니다. 이 단위가 캐시 라인입니다. 문제는 서로 아무 관계도 없는 두 데이터가, 크기가 작다는 이유만으로(예: int32 두 개) 메모리 상에서 우연히 인접해 같은 캐시 라인에 들어갈 수 있다는 점입니다. False Sharing은 바로 이 우연한 인접성 때문에 생기는 문제입니다.

서로 다른 코어에서, 같은 캐시 라인에 속한 두 변수 x와 y를 각각 수정하는 상황을 생각해 보겠습니다.

```c
// Assume x and y are in the same cache line
int x = 0;
int y = 0;

// Thread 1
for (int i = 0; i < 1000; ++i) {
    x++;
}

// Thread 2
for (int i = 0; i < 1000; ++i) {
    y++;
}
```

두 스레드는 어떤 변수도 공유하지 않습니다. 그런데도 CPU는 캐시 라인의 일부만 바뀌어도 캐시 일관성 프로토콜에 따라 그 캐시 라인 전체를 무효화합니다. 대표적인 프로토콜인 MESI는 캐시 라인의 상태를 Modified(이 코어가 수정한 최신 값), Exclusive(이 코어만 보유, 아직 미수정), Shared(여러 코어가 읽기 전용 공유), Invalid(내 캐시에 없음) 네 가지로 나눕니다. 어딘가에 쓰려면 M이나 E 상태를 가져야 하고, 한 코어가 소유권을 요청(RFO)하면 인터커넥트가 이를 중개해 다른 코어들의 캐시를 무효화 상태로 바꿉니다.

그러니 스레드1이 x를 바꾸면, 스레드2는 L1에 있던 캐시 라인을 지우고 L3에서 다시 읽어와야 합니다. 이게 바로 false sharing입니다.

![](/images/debugging-false-sharing/image4.png)

False sharing이 벌어지면 캐시 무효화 탓에 CPU stall이 늘어 CPI가 치솟고, 캐시를 계속 비우고 다시 채워야 하니 L1/L3 대역폭도 덩달아 높아집니다. 정확히 넷플릭스가 관측한 현상과 일치합니다.

## 원인 규명과 해결

실제로 문제를 일으킨 코드를 찾기 위해 CPU instruction profiling을 돌렸고, CPI가 100을 넘는 instruction을 발견했습니다. 범인은 JVM 내부에서 서브타입 체크를 빠르게 하려고 쓰는 두 변수, `secondary_supers_addr`와 `secondary_super_cache`였습니다. 이 최적화 기법 자체는 [Fast Subtype Checking in the HotSpot JVM](https://www.researchgate.net/publication/221552851_Fast_subtype_checking_in_the_HotSpot_JVM) 논문에 자세히 나와 있습니다.

![](/images/debugging-false-sharing/image5.png)

해결책은 두 변수 사이에 64바이트 패딩을 넣어 서로 다른 캐시 라인에 놓이도록 강제하는 것이었습니다.

```c
int x __attribute__((aligned(64)));
int y __attribute__((aligned(64)));
```

컴파일러 힌트를 쓰기 어려운 경우라면, 아래처럼 구조체 사이에 더미 버퍼를 끼워 넣는 방식으로도 같은 효과를 낼 수 있습니다.

```c
struct {
    int x;
    char padding[64]; // Ensures x and y are on different cache lines
    int y;
} vars;
```

![](/images/debugging-false-sharing/image6.png)

수정된 JDK를 적용하자 CPU 사용률이 정상 범위로 돌아왔습니다.

![](/images/debugging-false-sharing/image7.png)

## 남아있던 True Sharing

그런데 false sharing 병목이 풀리자, 이번엔 true sharing 문제가 수면 위로 떠올랐습니다. False sharing이 서로 무관한 변수가 같은 캐시 라인에 우연히 겹쳐서 생기는 문제라면, true sharing은 실제로 서로 관련 있는 변수를 여러 코어가 동시에 빈번하게 읽고 쓰면서 생기는 문제입니다. 즉, 자주 접근되는 공유 변수가 실제로 존재한다는 뜻입니다.

이번 범인은 `super_cache_addr`라는 변수였습니다. 넷플릭스 팀은 이 값을 아예 캐싱하지 않도록 설정을 바꾸는 방식으로 true sharing을 해소했습니다.

![](/images/debugging-false-sharing/image8.png)

## 최종 결과

두 문제를 모두 해결하고 나자, 처리량과 지연 시간이 함께 개선되었습니다.

![](/images/debugging-false-sharing/image9.png)

## 왜 일부 노드에서만 발생했을까

일반적인 캐시 라인 크기는 64바이트고, 문제가 된 두 변수 `_secondary_super_cache`와 `_secondary_supers`는 각각 8바이트입니다. 메모리 레이아웃이 사실상 무작위로 결정된다고 보면, 인접한 두 8바이트 데이터가 같은 64바이트 캐시 라인에 들어갈 확률은 87.5%입니다. 운 좋게 두 변수가 서로 다른 캐시 라인에 떨어진 12.5%의 노드는 멀쩡했고, 나머지 87.5%의 노드에서만 성능 저하가 나타난 겁니다.

## 읽으면서 든 의문들

원문을 끝까지 읽고도 완전히 풀리지 않은 부분이 두 가지 있었습니다.

첫 번째는 MACHINE_CLEAR 빈도가 왜 함께 증가했는가입니다. False sharing으로 L1 캐시 무효화와 CPU stall이 늘어나는 것까지는 직관적으로 이해가 되는데, false sharing이 MACHINE_CLEAR를 유발하는 hazard의 직접적인 원인이라는 연결 고리는 원문만으로는 다소 불명확했습니다.

두 번째는 CPU를 업그레이드하기 전에는 왜 같은 문제가 드러나지 않았는가입니다. 물리 코어 수가 늘어나면 false sharing이 심해지는 것 자체는 자연스럽지만, 그렇다면 업그레이드 이전에도 같은 문제가 정도만 약하게 존재했어야 하는 게 아닌가 하는 의문이 남습니다.

## References
- [Seeing through hardware counters: a journey to threefold performance increase (Netflix Tech Blog)](https://netflixtechblog.com/seeing-through-hardware-counters-a-journey-to-threefold-performance-increase-2721924a2822)
- [Fast Subtype Checking in the HotSpot JVM](https://www.researchgate.net/publication/221552851_Fast_subtype_checking_in_the_HotSpot_JVM)
