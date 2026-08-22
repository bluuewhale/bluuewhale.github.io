+++
title = 'Linux Transparent Huge Pages'
date = '2025-09-16T16:15:43+09:00'
draft = false
slug = 'linux-transparent-huge-pages'
description = '대형 페이지(Huge Page)로 TLB 미스 오버헤드를 줄이는 원리와, hugetlbfs·THP의 차이, 벤치마크로 확인한 성능 개선 결과를 정리합니다.'
tags = ['Linux', 'THP', 'Huge Pages', 'TLB', 'Performance', 'Memory']
categories = ['Linux', 'Performance']

[cover]
image = 'images/linux-transparent-huge-pages/image1.png'
alt = '가상 메모리 주소를 물리 메모리 주소로 매핑하는 Page Table 구조'
hiddenInSingle = true
+++

## 요약

- 대형 페이지(Huge Page) 기능을 활용하여 TLB 미스로 인한 오버헤드를 최소화할 수 있습니다.
- 워킹셋 크기가 크고(수십 MB 이상), 메모리 접근이 매우 빈번한 워크로드에서 유용한 최적화 기법입니다.
    - 주로 데이터베이스 시스템과 관련된 레퍼런스([Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/cwlin/reviewing-hugespages-memory-allocation.html), [Postgres](https://www.postgresql.org/docs/current/kernel-resources.html#LINUX-HUGE-PAGES), [AWS RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Oracle.Concepts.HugePages.html))가 많습니다.
    - 대형 JVM 애플리케이션
    - 하이퍼바이저(KVM/QEMU)
    - 대규모 분석·검색 시스템(예: ClickHouse)
- CPU 및 I/O 바운드가 지배적인 워크로드에서는 별다른 효용이 없을 수 있습니다.

## 배경

- 모든 프로세스는 가상 메모리 주소 공간을 할당받습니다.
- 프로세스에서 가상 메모리 주소에 접근하면 CPU는 이를 실제 물리 메모리 주소로 치환하여 처리합니다.
- OS는 가상 메모리 주소를 물리 메모리 주소로 매핑하기 위한 Page Table을 관리합니다.

![](/images/linux-transparent-huge-pages/image1.png)

- 프로세스가 메모리에 접근할 때마다, 가상 메모리 주소를 물리 메모리 주소로 변환하는 과정이 매번 발생합니다.
- CPU는 이 변환 과정을 최적화하기 위해 최근 변환된 메모리 주소를 캐싱하는데, 이 캐시를 TLB(Translation Lookaside Buffer)라고 부릅니다.

![](/images/linux-transparent-huge-pages/image2.png)

- 문제는 TLB의 크기가 일반적으로 매우 작다는 점입니다(~4KB).
- 따라서 워킹셋 크기가 큰 경우에는 TLB 미스가 빈번하게 발생합니다.
- TLB의 크기는 제한적이므로, 대신 페이지 크기를 늘려 TLB 적중률을 높이는 방법이 고안되었습니다.

## Huge Pages

- 리눅스에서는 페이지 크기를 늘릴 수 있는 두 가지 옵션을 제공합니다.
    - [hugetlbfs](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
        - 리눅스에서 대형 페이지(Huge Page)를 활용하기 위해 제공되는 특별한 파일시스템입니다.
            - 커널이 제공하는 가상 파일시스템으로, `/dev/hugepages`와 같은 경로를 마운트할 수 있습니다.
            - 프로세스가 이 파일시스템에 파일을 만들고 `mmap()`을 하면, 그 매핑은 대형 페이지 단위로 할당됩니다.
            - 즉, 일반적인 `malloc`이 아니라 `hugetlbfs`로 마운트한 경로를 통해 대형 페이지 메모리를 사용합니다.
        - 특징
            - 시스템 메모리 중에서 hugetlbfs가 사용할 대형 페이지 개수를 미리 설정해야 합니다.
                - 예약된 대형 페이지는 다른 용도로 사용되지 않습니다.
                - 애플리케이션이 이를 사용하지 않으면 그대로 낭비됩니다.
            - 애플리케이션 코드에서 메모리를 사용하는 방식을 바꿔야 합니다.
                - C처럼 메모리를 명시적으로 할당하는 언어로 작성된 프로그램은 코드를 수정해야 합니다.
                - JVM과 같은 런타임에서는 `-XX:+UseHugeTLBFS` 옵션을 사용하면 됩니다.
        - 활용 레퍼런스
            - 초단타 매매(HFT), 데이터베이스([Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/cwlin/reviewing-hugespages-memory-allocation.html), [Postgres](https://www.postgresql.org/docs/current/kernel-resources.html#LINUX-HUGE-PAGES), [AWS RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Oracle.Concepts.HugePages.html)) 및 캐시 서버 등에서 활용됩니다.
    - [**Transparent Huge Pages (THP)**](https://www.kernel.org/doc/Documentation/vm/transhuge.txt)
        - 리눅스 커널에서 일반 페이지를 대형 페이지로 자동 승격·강등해주는 기능입니다.
            - 애플리케이션에서 기존과 동일하게 메모리를 할당하면 커널이 자동으로 일반 페이지를 대형 페이지로 승격(promotion)시키거나 대형 페이지를 일반 페이지로 강등(demotion)합니다.
            - 커널이 주기적으로 defrag를 수행하여 페이지를 압축합니다.
        - 특징
            - hugetlbfs와 달리 메모리 공간 예약이 필요하지 않습니다.
            - 애플리케이션 코드를 수정할 필요가 없습니다.
            - defrag의 영향으로 지연 시간 스파이크가 발생할 수 있습니다.
        - 활용 레퍼런스
            - [RedHat 리눅스 배포판에서는 기본값으로 THP 기능이 활성화](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-configuring_transparent_huge_pages)되어 있습니다.
            - [MS SQL Server에서는 THP 기능을 사용하도록 권장](https://learn.microsoft.com/en-us/sql/linux/sql-server-linux-performance-best-practices?view=sql-server-ver17#leave-transparent-huge-pages-thp-enabled)합니다.
            - 반대로 사용을 경고하는 사례도 일부 존재합니다.
                - [Redis에서는 THP 기능을 반드시 끄도록 권장하고 있습니다.](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/#ive-little-time-give-me-the-checklist)

## 검증

- 아래는 벤치마크 테스트에 활용된 코드 예시입니다.
    - 임의 크기의 `byte[]`를 생성하고 무작위 인덱스에 접근합니다.
    - 바이트 배열의 크기가 커질수록 많은 페이지가 할당되며 TLB 미스가 발생할 가능성이 높아집니다.

```java
public class ByteArrayTouch {

    @Param(...)
    int size;

    byte[] mem;

    @Setup
    public void setup() {
        mem = new byte[size];
    }

    @Benchmark
    public byte test() {
        return mem[ThreadLocalRandom.current().nextInt(size)];
    }
}
```

- 벤치마크 테스트 결과, 바이트 배열의 크기가 큰 구간에서 최대 15% 수준의 성능 개선이 이루어졌습니다.

```
Benchmark               (size)  Mode  Cnt   Score   Error  Units

# Baseline
ByteArrayTouch.test       1000  avgt   15   8.109 ± 0.018  ns/op
ByteArrayTouch.test      10000  avgt   15   8.086 ± 0.045  ns/op
ByteArrayTouch.test    1000000  avgt   15   9.831 ± 0.139  ns/op
ByteArrayTouch.test   10000000  avgt   15  19.734 ± 0.379  ns/op
ByteArrayTouch.test  100000000  avgt   15  32.538 ± 0.662  ns/op

# -XX:+UseTransparentHugePages
ByteArrayTouch.test       1000  avgt   15   8.104 ± 0.012  ns/op
ByteArrayTouch.test      10000  avgt   15   8.060 ± 0.005  ns/op
ByteArrayTouch.test    1000000  avgt   15   9.193 ± 0.086  ns/op // !
ByteArrayTouch.test   10000000  avgt   15  17.282 ± 0.405  ns/op // !!
ByteArrayTouch.test  100000000  avgt   15  28.698 ± 0.120  ns/op // !!!

# -XX:+UseHugeTLBFS
ByteArrayTouch.test       1000  avgt   15   8.104 ± 0.015  ns/op
ByteArrayTouch.test      10000  avgt   15   8.062 ± 0.011  ns/op
ByteArrayTouch.test    1000000  avgt   15   9.303 ± 0.133  ns/op // !
ByteArrayTouch.test   10000000  avgt   15  17.357 ± 0.217  ns/op // !!
ByteArrayTouch.test  100000000  avgt   15  28.697 ± 0.291  ns/op // !!!
```

- CPU 카운터를 분석한 결과는 다음과 같습니다.
    - 일반 페이지를 사용한 경우에는 100%에 가까운 TLB 미스가 발생했습니다.
    - 반면, THP를 사용한 경우 TLB 미스가 거의 발생하지 않았습니다.

```
Benchmark                                (size)  Mode  Cnt    Score    Error  Units

# Baseline
ByteArrayTouch.test                   100000000  avgt   15   33.575 ±  2.161  ns/op
ByteArrayTouch.test:cycles            100000000  avgt    3  123.207 ± 73.725   #/op
ByteArrayTouch.test:dTLB-load-misses  100000000  avgt    3    1.017 ±  0.244   #/op  // !!!
ByteArrayTouch.test:dTLB-loads        100000000  avgt    3   17.388 ±  1.195   #/op

# -XX:+UseTransparentHugePages
ByteArrayTouch.test                   100000000  avgt   15   28.730 ±  0.124  ns/op
ByteArrayTouch.test:cycles            100000000  avgt    3  105.249 ±  6.232   #/op
ByteArrayTouch.test:dTLB-load-misses  100000000  avgt    3   ≈ 10⁻³            #/op  // !!!
ByteArrayTouch.test:dTLB-loads        100000000  avgt    3   17.488 ±  1.278   #/op
```

## 레퍼런스

- https://shipilev.net/jvm/anatomy-quarks/2-transparent-huge-pages/
