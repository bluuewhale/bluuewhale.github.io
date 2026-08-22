+++
title = 'JVM Lock Coarsening'
date = '2025-09-16T16:12:56+09:00'
draft = false
slug = 'jvm-lock-coarsening'
description = '같은 모니터에 대해 반복적으로 발생하는 락 획득과 해제를 하나의 큰 단위로 합쳐서 오버헤드를 줄이는 JIT 최적화 기법인 Lock Coarsening을 벤치마크와 함께 살펴봅니다.'
tags = ['JVM', 'JIT', 'Lock Coarsening', 'Java', 'Performance']
categories = ['JVM', 'Performance']
+++

## Lock Coarsening이란?

- 같은 모니터(객체)에 대해 락을 연속적으로 잡았다 풀었다 하는 동작을 더 큰 단위로 묶어서 한 번에 잡고 푸는 형태로 바꿈으로써, 락을 획득하고 해제하는 데 따르는 오버헤드를 최소화하는 JIT 최적화 기법입니다.
- `-XX:+EliminateLocks`  옵션이 활성화되어 있으면 적용됩니다.

## 핵심 아이디어

- 자바의 `synchroinzed`는 바이트코드로 `monitorenter/moniterexit`으로 표현됩니다.
- 반복문이나 인접한 코드 영역에서 같은 객체에 대해 `monitorenter/moniterexit`가 짧은 간격으로 반복되면, JIT 컴파일러는 이를 한 번만 처리하도록 변경합니다.
- 이를 통해 CAS(Compare-and-Set) 연산, 메모리 배리어, 스택 및 헤더 변경과 같은 락 비용을 최소화합니다.

## 예시

### Lock Coarsening 전

```java
for (int i = 0; i < n; i++) {
    synchronized (lock) {
        doWork(i);
    }
}
```

### Lock Coarsening 후

```java
synchronized (lock) {
    for (int i = 0; i < n; i++) {
        doWork(i);
    }
}
```

## 검증

- 아래와 같이 테스트 코드를 설정했습니다.

```java
@Fork(..., jvmArgsPrepend = {"-XX:-UseBiasedLocking"})
@State(Scope.Benchmark)
public class LockRoach {
    int x;

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public void test() {
        for (int c = 0; c < 1000; c++) {
            synchronized (this) {
                x += 0x42;
            }
        }
    }
}
```

- `-prof perfasm`를 사용하여 디스어셈블리를 분석했습니다.
    - 분석 결과, JIT 컴파일러가 Loop Unrolling을 적용하여 락 획득과 해제 빈도가 감소한 것을 확인했습니다.
    - 다만, 루프 전체에 대해 lock coarsening이 적용되지는 않았습니다.
        - JVM은 과도한 코어스닝이 한 스레드가 락을 오래 독점하게 만들 수 있어 위험하다고 판단했습니다.

```
 ↗  0x00007f455cc708c1: lea    0x20(%rsp),%rbx
 │          < blah-blah-blah, monitor enter >     ; <--- coarsened!
 │  0x00007f455cc70918: mov    (%rsp),%r10        ; load $this
 │  0x00007f455cc7091c: mov    0xc(%r10),%r11d    ; load $this.x
 │  0x00007f455cc70920: mov    %r11d,%r10d        ; ...hm...
 │  0x00007f455cc70923: add    $0x42,%r10d        ; ...hmmm...
 │  0x00007f455cc70927: mov    (%rsp),%r8         ; ...hmmmmm!...
 │  0x00007f455cc7092b: mov    %r10d,0xc(%r8)     ; LOL Hotspot, redundant store, killed two lines below
 │  0x00007f455cc7092f: add    $0x108,%r11d       ; add 0x108 = 0x42 * 4 <-- unrolled by 4
 │  0x00007f455cc70936: mov    %r11d,0xc(%r8)     ; store $this.x back
 │          < blah-blah-blah, monitor exit >      ; <--- coarsened!
 │  0x00007f455cc709c6: add    $0x4,%ebp          ; c += 4   <--- unrolled by 4
 │  0x00007f455cc709c9: cmp    $0x3e5,%ebp        ; c < 1000?
 ╰  0x00007f455cc709cf: jl     0x00007f455cc708c1
```

    - Loop Unrolling이란?
        - 루프 본문을 복제하여 반복 횟수와 분기 비용을 줄이는 컴파일러의 루프 최적화 기법입니다.
        - Loop Unrolling이 적용된 코드의 예시는 다음과 같습니다.

```java
@Fork(..., jvmArgsPrepend = {"-XX:-UseBiasedLocking"})
@State(Scope.Benchmark)
public class LockRoach {
    int x;

    @Benchmark
    @CompilerControl(CompilerControl.Mode.DONT_INLINE)
    public void test() {
        for (int c = 0; c < 1000; c += 4) {
            synchronized (this) {
                // 동일한 본문을 4번 복제
                x += 0x42;
                x += 0x42;
                x += 0x42;
                x += 0x42;
            }
        }
    }
}
```

- Loop Unrolling을 통해 락 획득과 해제 빈도를 1/4 수준으로 줄일 수 있었습니다.
    - Loop Unrolling 기능을 제한하면 성능이 4배 저하되는 것을 확인했습니다.

```
Benchmark            Mode  Cnt      Score    Error  Units

# Default
LockRoach.test       avgt    5   5331.617 ± 19.051  ns/op

# -XX:LoopUnrollLimit=1
LockRoach.test       avgt    5  20679.043 ±  3.133  ns/op // 4배 느려짐
```

## 레퍼런스

- https://shipilev.net/jvm/anatomy-quarks/1-lock-coarsening-for-loops/
