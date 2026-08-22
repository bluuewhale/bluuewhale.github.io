+++
title = 'Spinlock vs Mutex'
date = '2025-12-12T10:20:40+09:00'
draft = false
translationKey = 'spinlock-vs-mutex'
slug = 'spinlock-vs-mutex'
description = '유저 공간에서 CAS로 도는 Spinlock과, 커널 공간의 futex를 거치는 Mutex가 왜 서로 다른 워크로드에 적합한지, 캐시 일관성 프로토콜과 함께 살펴봅니다.'
tags = ['Concurrency', 'Linux', 'CPU', 'Lock-Free']
categories = ['Concurrency', 'Systems']

[cover]
image = 'images/spinlock-vs-mutex/image1.png'
hiddenInSingle = true
+++

락을 구현하는 방식은 크게 두 갈래로 나뉩니다. 유저 공간에서 CPU 명령어만으로 도는 Spinlock과, 커널의 도움을 받는 Mutex입니다. 둘 다 결국 상호 배제(mutual exclusion)를 보장한다는 목적은 같지만, 내부 동작 방식이 다르기 때문에 적합한 워크로드도 갈립니다.

## Spinlock

Spinlock은 유저 공간(userspace)에서 구현되는 락입니다. 동작 방식은 단순합니다. CAS(Compare-And-Swap) 연산이 성공할 때까지 무한히 반복합니다.

```c
while (!CAS(lock, 0, 1))
```

대부분의 CPU는 이런 CAS 연산을 위한 전용 명령어를 갖고 있습니다. x86이라면 `LOCK CMPXCHG`가 여기에 해당합니다. 문제는 이 CAS 연산이 성공하려면 해당 캐시 라인에 대한 독점적인 쓰기 권한이 필요하다는 점입니다. 이 권한을 얻는 과정에서 CPU 코어 사이에 캐시 일관성 프로토콜이 개입하고, 캐시 라인의 소유권을 주고받는 과정에서 잦은 invalidation, 이른바 캐시 라인 바운싱이 발생합니다. 이 과정은 보통 4~80ns 정도 걸립니다.

캐시 라인은 CPU 캐시가 데이터를 옮기고 일관성을 관리하는 최소 단위입니다. 현대 CPU는 대부분 64바이트 크기의 캐시 라인을 쓰는데, 8바이트짜리 데이터 하나를 읽어도 그 데이터가 속한 64바이트 전체를 통째로 가져옵니다. 공간 지역성(locality)을 살리기 위한 설계지만, 이 때문에 서로 무관한 변수 두 개가 같은 캐시 라인에 우연히 걸쳐 있으면 False Sharing 같은 문제가 생기기도 합니다.

캐시 일관성 프로토콜은 멀티코어 환경에서 각 코어의 캐시가 서로 어긋나지 않도록 맞춰주는 코어 간 통신 규약입니다. 가장 널리 쓰이는 것이 MESI로, 캐시 라인의 상태를 Modified(이 코어가 수정한 최신 값이고 메모리는 아직 옛값), Exclusive(이 코어만 갖고 있지만 아직 수정 전), Shared(여러 코어가 읽기 전용으로 공유), Invalid(내 캐시엔 없음) 네 가지로 나눕니다. 어딘가에 쓰기를 하려면 그 라인에 대해 M이나 E 상태를 가져야 하고, S 상태에서는 읽기만 가능합니다. 대부분의 현대 CPU는 이를 invalidate 방식으로 구현합니다. 한 코어가 소유권을 요청(RFO, Read For Ownership)하면 인터커넥트가 이를 중개해 다른 코어들의 캐시를 무효화 상태(I)로 바꾸고, 그제서야 요청한 코어가 소유권(M 또는 E)을 갖습니다. 이 메시지들은 L1/L2 캐시 컨트롤러와 인터커넥트를 거쳐 오갑니다.

Spinlock은 락 처리가 전부 유저 공간에서 끝나기 때문에 시스템 콜이 발생하지 않는다는 장점이 있습니다. 다만 경합이 발생하면 스레드가 무한 루프를 계속 돌기 때문에, 그 시간만큼 CPU를 그대로 소모합니다.

## Mutex

Mutex는 유저 공간과 커널 공간의 명령어를 모두 아우르는 락입니다. 현대적인 커널 대부분은 이중 구조로 구현합니다. 경합이 없을 때는 spinlock과 똑같이 CAS로 빠르게 락을 얻는 fast path를 타고, 실패하면 스레드를 대기 상태로 보내는 시스템 콜(`futex(FUTEX_WAIT)`)을 호출하는 slow path로 넘어갑니다.

그래서 경합이 없는 환경에서는 Mutex도 CAS 한 번으로 빠르게 락을 얻습니다. 경합이 있는 환경에서는 컨텍스트 스위칭을 통해 불필요한 CPU 소모를 막지만, 그 대신 spinlock보다 훨씬 비싼 비용을 치릅니다. 시스템 콜 자체가 대략 500ns, 컨텍스트 스위치는 3~5μs 정도 걸립니다.

## Spinlock vs Mutex

둘을 나란히 놓고 보면 이렇습니다. Spinlock은 40~80ns 수준으로 빠른 경량 연산이지만, 무한 루프를 도는 방식이라 경합이 심해질수록 CPU 부하가 그대로 치솟습니다. 그래서 메모리 접근처럼 임계 구역이 매우 짧게 끝나는 워크로드에 잘 맞습니다. Mutex는 3~5μs로 상대적으로 무겁지만, spinlock과 비슷한 fast path를 갖고 있어 경합이 없을 때는 성능 차이가 크지 않습니다. 대신 경합이 있을 때 CPU를 다른 스레드에 양보하기 때문에 CPU 사용 효율이 높고, 디스크나 네트워크 접근처럼 락을 쥐고 있는 시간이 상대적으로 긴 워크로드에 적합합니다.

실제로 메모리 접근이 대부분인 Redis는 job queue 같은 곳에서 내부적으로 spinlock을 주로 쓰고, I/O를 유발하는 구간이 많은 PostgreSQL 같은 DB는 mutex를 주로 활용합니다.

![](/images/spinlock-vs-mutex/image1.png)

## Lock-free 자료구조

일반적인 thread-safe 자료구조는 내부적으로 spinlock이나 mutex를 획득한 뒤 데이터를 수정합니다. 반면 일부 자료구조는 아예 락 없이, 원자적 연산만으로 thread-safety를 구현합니다. 대부분 spinlock과 비슷하게 CAS와 무한 루프를 조합한 형태입니다. 이런 자료구조를 lock-free 자료구조라고 부르는데, 대표적인 예가 Treiber Stack입니다.

```java
// 코드 예시 - from Wikipedia
import java.util.concurrent.atomic.*;

import net.jcip.annotations.*;

/**
 * ConcurrentStack
 *
 * Nonblocking stack using Treiber's algorithm
 *
 * @author Brian Goetz and Tim Peierls
 */
@ThreadSafe
public class ConcurrentStack <E> {
    AtomicReference<Node<E>> top = new AtomicReference<Node<E>>();

    public void push(E item) {
        Node<E> newHead = new Node<E>(item);
        Node<E> oldHead;

        do {
            oldHead = top.get();
            newHead.next = oldHead;
        } while (!top.compareAndSet(oldHead, newHead));
    }

    public E pop() {
        Node<E> oldHead;
        Node<E> newHead;

        do {
            oldHead = top.get();
            if (oldHead == null)
                return null;
            newHead = oldHead.next;
        } while (!top.compareAndSet(oldHead, newHead));

        return oldHead.item;
    }

    private static class Node <E> {
        public final E item;
        public Node<E> next;

        public Node(E item) {
            this.item = item;
        }
    }
}
```

push와 pop 모두 현재 top을 읽고, 새 노드를 연결한 다음, CAS로 top을 교체하는 구조입니다. 다른 스레드가 그 사이에 끼어들어 top을 바꿔버리면 CAS가 실패하고, 그러면 그냥 처음부터 다시 시도합니다. 락을 잡지 않고도 스택의 일관성을 지킬 수 있는 이유입니다.

## References
- [Spinlocks vs. Mutexes: When to Spin and When to Sleep](https://howtech.substack.com/p/spinlocks-vs-mutexes-when-to-spin)
- [Treiber stack](https://en.wikipedia.org/wiki/Treiber_stack)
