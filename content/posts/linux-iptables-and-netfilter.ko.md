+++
title = 'A Deep Dive into iptables and Netfilter'
date = '2022-03-07T18:50:53+09:00'
draft = false
slug = 'linux-iptables-and-netfilter'
description = 'iptables가 netfilter의 다섯 가지 훅과 테이블·체인·룰을 통해 패킷을 제어하는 방식을 정리한 번역 글입니다.'
tags = ['Linux', 'iptables', 'Netfilter', 'Networking']
categories = ['Linux', 'Networking']

[cover]
image = 'images/linux-iptables-and-netfilter/image1.png'
alt = 'iptables mangle 테이블의 패킷 처리 흐름'
hiddenInSingle = true
+++

> 이 글은 원문 [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)를 번역하여 작성한 글입니다.

## iptables란?
`iptables`는 `netfilter` 프레임워크를 제어하는 데 사용되는, 리눅스 커널에 내장된 네트워크 스택입니다. NIC를 거쳐 유입되는 모든 네트워크 패킷은 `netfilter`에 등록된 룰을 거쳐 제어됩니다. `iptables`는 `netfilter`가 패킷 조작을 위해 제공하는 훅에서 룰을 관리하는 역할을 합니다. `iptables`를 활용하면 특정 조건에 부합하는 패킷을 드롭하거나, 패킷의 출발지(source) 또는 목적지(destination)를 수정하는 것 등이 가능합니다.

## Netfilter Hook
`netfilter`는 패킷 제어를 위한 다섯 가지 훅을 제공합니다. 모든 패킷에는 전송 방향(`incoming` 혹은 `outgoing`)에 따라 각 훅에 등록된 패킷 제어 규칙이 적용됩니다. `netfilter`에서 지원하는 훅은 다음과 같습니다.

- `NF_IP_PRE_ROUTING`: 호스트로 들어오는(`incoming`) 트래픽에 대해 가장 먼저 적용되는 훅으로, 패킷을 라우팅하기 이전에 적용됩니다.

- `NF_IP_LOCAL_IN`: 로컬 시스템으로 향하는 `incoming` 트래픽에 대해 적용되는 훅입니다.

- `NF_IP_FORWARD`: 다른 호스트로 포워딩되는 트래픽에 대해 적용되는 훅입니다.

- `NF_IP_LOCAL_OUT`: 로컬 시스템에서 외부로 나가는(`outgoing`) 트래픽에 대해 가장 먼저 적용되는 훅입니다.

- `NF_IP_POST_ROUTING`: 로컬 시스템에서 나가거나, 외부로 포워딩되는 모든 트래픽이 최종적으로 거치는 훅입니다.

각 훅에 등록되는 체인(규칙)은 서로 다른 우선순위를 가지고 있으며, 같은 훅 안에서는 우선순위가 높은 체인이 먼저 적용됩니다.

## Chain
`iptables`는 테이블과 체인을 통해 `netfilter`에 등록된 패킷 제어 룰을 관리합니다. 각 테이블은 저마다 특수한 목적을 가지고 있으며, 룰의 성격에 따라 어떤 테이블에 등록될지 결정됩니다. 예를 들어, 출발지/목적지 주소 변환과 관련된 룰은 `nat` 테이블에 등록됩니다. 마찬가지로, 패킷을 드롭할지 여부를 결정하는 룰은 `filter` 테이블에 등록됩니다.

각 테이블에 포함된 모든 패킷 제어 룰은 체인(`chain`)이라 불리는 그룹으로 묶여서 관리됩니다. `iptables`에 내장된 대표적인 체인들은 앞서 소개한 `netfilter hook`과 1:1 대응 관계에 있습니다. 따라서, `netfilter`에서 특정 훅이 실행될 때, `iptables`에서 해당 훅에 대응되는 체인에 등록된 룰이 실행됩니다. `iptables`에서 제공하는 대표적인 훅들은 다음과 같습니다.

- `PREROUTING`: `NF_IP_PRE_ROUTING` 훅이 실행될 때, 적용됩니다.
- `INPUT`: `NF_IP_LOCAL_IN` 훅이 실행될 때 적용됩니다.
- `FORWARD`: `NF_IP_FORWARD` 훅이 실행될 때 적용됩니다.
- `OUTPUT`: `NF_IP_LOCAL_OUT` 훅이 실행될 때 적용됩니다.
- `POSTROUTING`: `NF_IP_POST_ROUTING` 훅이 실행될 때 적용됩니다.

각 테이블은 한 개 이상의 체인으로 구성되며, 테이블의 성격에 따라 등록되는 체인의 종류에 차이가 있습니다. 예를 들어, `filter` 테이블에는 `PREROUTING`, `POSTROUTING` 체인은 등록되지 않습니다.

## Table
`iptables`는 다섯 개의 테이블을 지원합니다.

### Filter Table
`filter` 테이블은 `iptables`의 기본(`default`) 테이블입니다. `filter` 테이블은 등록된 룰에 따라 들어오거나(`incoming`) 나가는(`outgoing`) 패킷의 허용/거부 여부를 결정하는 방화벽의 역할을 수행합니다.

### NAT Table
`nat` 테이블은 이름처럼 주소 변환(`network address translation`)을 담당합니다. `nat` 테이블은 등록된 룰에 따라 패킷의 출발지(`source`) 혹은 목적지(`destination`)를 조작하여 패킷을 라우팅하는 역할을 합니다.

### Mangle Table
`mangle` 테이블은 패킷의 IP 헤더 일부를 변환하기 위해 사용됩니다.

![](/images/linux-iptables-and-netfilter/image1.png)

예를 들어, `mangle` 테이블에 규칙을 등록하면 패킷의 TTL 헤더를 조작하여 패킷이 거칠 수 있는 네트워크 홉의 수를 줄이거나 늘리는 것이 가능합니다.

`mangle` 테이블은 패킷에 커널 `mark`를 남길 수 있습니다. `mark`는 패킷에 직접적인 영향을 주지는 않지만, 다른 테이블에서 패킷을 제어할 때 추가 정보로 활용될 수 있습니다.

### Raw Table
`raw` 테이블은 앞서 소개한 세 가지 테이블과는 조금 다른 성격을 가지고 있습니다. `iptables`는 `connection tracking` 기능을 제공합니다.

`iptables`에서 모든 `connection`은 상태를 가지며, `iptables`에 등록된 모든 규칙은 연결 상태에 영향을 받습니다. 예를 들어, `INVALID` 상태에 놓인 `connection`에서 오는 패킷은 유효하지 않은 것으로 간주됩니다. `raw` 테이블은 이러한 연결 상태를 제어하고, 연결 상태를 패킷에 마킹하는 고유한 역할을 수행합니다. 따라서, 사용자가 `raw` 테이블을 직접 제어할 일은 많지 않습니다.

### Security Table
`security` 테이블은 `raw` 테이블과 마찬가지로 다소 특수한 테이블입니다. 이 테이블은 `SELinux` 내부적으로 패킷에 `security context`를 마킹하는 역할을 합니다. `security` 테이블을 거쳐 생성된 마크는 `SELinux`가 패킷을 제어하는 데 영향을 줍니다.

## Table과 Chain의 관계
`iptables`에는 다섯 개의 테이블이 있으며, 각 테이블은 그 성격에 따라 서로 다른 체인들을 가지고 있습니다. `netfilter`에서는 각 체인들이 패킷의 성격(`incoming`, `outgoing`)에 따라 테이블의 우선순위대로 적용됩니다.

주의할 점은, `nat` 테이블에서 `SNAT`과 `DNAT` 룰이 적용되는 체인이 서로 다르다는 것입니다. 예를 들어, `PREROUTING` 체인에서는 `DNAT`과 관련된 룰만 적용됩니다.

![](/images/linux-iptables-and-netfilter/image2.png)

### Incoming Packet

NIC를 거쳐 들어온 패킷에는 가장 먼저 `NF_IP_PRE_ROUTING` 훅이 실행됩니다. 이때, `NF_IP_PRE_ROUTING` 훅과 연관된 `PREROUTING` 체인이 실행됩니다. `PREROUTING` 훅이 등록된 모든 테이블 중 `raw` 테이블이 가장 먼저 적용되어 연결 상태를 제어하고 패킷에 마킹을 남깁니다. 그 후, `mangle`, `nat(DNAT)` 테이블을 각각 거쳐 라우팅 여부가 결정됩니다.

패킷이 로컬 시스템을 향하는 경우에는, `NF_IP_LOCAL_IN` 훅과 연관된 `INPUT` 체인이 실행됩니다. `mangle`, `filter` 테이블에 등록된 `INPUT` 체인이 순서대로 적용되어 최종적으로 허용/거부 여부가 결정됩니다.

만약 패킷이 외부 호스트로 포워딩되는 경우에는 `NF_IP_FORWARD` 훅과 연관된 `FORWARD` 체인이 실행됩니다. `FORWARD` 체인은 `INPUT` 체인과 동일하게 `mangle`, `filter` 테이블 순으로 적용됩니다. 이후, `mangle`, `nat(SNAT)` 테이블에 등록된 `POSTROUTING` 체인들이 순서대로 적용됩니다.

### Outgoing Packet
로컬 호스트에서 출발하는 패킷에는 가장 먼저 `OUTPUT` 체인이 적용됩니다. `PREROUTING` 체인과 유사하게, `raw` 테이블을 거쳐 연결 상태가 가장 먼저 마킹되며, `mangle`, `nat(DNAT)`, `filter` 테이블에 등록된 `OUTPUT` 체인이 순서대로 실행됩니다. 이후, `mangle`, `nat(SNAT)` 테이블에 등록된 `POSTROUTING` 체인을 거쳐 최종적으로 NIC를 통해 외부로 나가게 됩니다.

### Chain Traversal Order
`netfilter packet traversal` 과정에서 체인이 적용되는 순서를 요약하면 아래와 같습니다.

- 로컬 시스템으로 들어오는 패킷: `PREROUTING` -> `INPUT`
- 외부 호스트로 포워딩되는 패킷: `PREROUTING` -> `FORWARD` -> `POSTROUTING`
- 외부로 나가는 패킷: `OUTPUT` -> `POSTROUTING`


## Rules
룰은 패킷을 어떻게 제어할지 결정하는 구체적인 내용을 담고 있는 규칙으로, 각 테이블에 등록되는 체인들은 한 개 이상의 룰(`Rule`)을 가지고 있습니다. `Chain Traversal Order`에 따라 특정 체인이 실행될 때, 해당 체인에 속한 룰이 순서대로 적용됩니다.

룰은 크게 두 파트(`Matching`, `Target`)로 구분됩니다.

### Matching
`matching`은 룰을 적용하기 위한 조건(`criteria`)에 해당하는 파트로 `if` 절의 조건에 해당한다고 볼 수 있습니다. 만약, 특정 패킷이 현재 룰의 `matching` 조건에 부합한 경우, `target`에 등록된 구체적인 행동이 적용됩니다.

### Target
현재 룰의 `matching` 조건에 부합하는 패킷을 어떻게 처리할지를 결정하는 실질적인 행위(`action`)에 해당하는 파트입니다. `target`의 종류는 크게 `terminating`과 `non-terminating`으로 나뉩니다.

#### Terminating target
`terminating target`은 패킷의 허용/거부 여부를 결정합니다.
`terminating target`이 실행된 경우에는, 해당 체인에 포함된 나머지 규칙들이 실행되지 않고 다음 체인으로 넘어갑니다.

#### Non-terminating target
`non-terminating target`은 체인 내부에서 `terminating target` 이전에 실행되는 `action`들입니다. 하나의 체인에서 `terminating target`이 실행되기 전까지 여러 개의 `non-terminating target`이 실행될 수 있습니다.

## References
- [A Deep Dive into Iptables and Netfilter Architecture](https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture)
