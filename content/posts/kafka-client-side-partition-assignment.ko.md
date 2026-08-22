+++
title = 'Kafka Client-Side Partition Assignment'
date = '2024-04-14T11:04:47+09:00'
draft = false
slug = 'kafka-client-side-partition-assignment'
description = 'Kafka 컨슈머 그룹 리밸런싱 시 파티션 분배가 브로커가 아닌 클라이언트에서 이뤄지는 이유와, JoinGroup/SyncGroup 프로토콜을 통해 이를 구현한 방식을 살펴봅니다.'
tags = ['Kafka', 'Consumer Group', 'Partition Assignment', 'Distributed Systems']
categories = ['Kafka', 'Distributed Systems']

[cover]
image = 'images/kafka-client-side-partition-assignment/image1.png'
alt = 'Kafka JoinGroupRequest를 통한 컨슈머 그룹 리더 선출 과정'
hiddenInSingle = true
+++

카프카 클라이언트 라이브러리 코드를 살펴보던 와중에, 컨슈머 그룹 리밸런싱이 발생한 시점에 컨슈머가 구독할 파티션을 결정하는 `ConsumerPartitionAssignor` 인터페이스와 몇 가지 구현체 (`RangePartitionAssignor`, `CooperativeStickyAssignor`)가 있는 것을 확인할 수 있습니다.

연결된 컨슈머 정보를 기반으로 파티션을 분배하는 작업은 서버(e.g. 브로커)에서 처리 해야 할 작업 같은데, 이러한 내용이 클라이언트 라이브러리 코드에 있는 것이 다소 생소하게 느껴졌습니다.

이에 대해 조금 더 살표보다가 카프카에서 이러한 구조를 채택하게 된 배경을 설명하는 문서를 찾아 공유 드립니다.

### client-side 파티션 분배 정책의 필요성
파티션 분배 작업은 반드시 서버가 처리해야 하는 것은 아닙니다. 예를 들어, 파티션 분배의 책임이 브로커에 있을 경우, 새로운 파티션 분배 정책를 적용하기 위해서는 브로커를 재시작해야 하는 부담이 있습니다. 파티션 분배 정책은 매우 다양할 수 있으며 이에 대한 획일화된 validation 규칙을 적용하기 어렵습니다. 예를 들어, 하나의 파티션을 여러 컨슈머에게 동시에 배정 해야 할 수도 있습니다. 커스텀한 파티션 분배 정책이 필요한 대표적인 사례는 다음과 같습니다.
  - Co-partitioning: 토픽 조인 작업을 수행할 때, 서로 다른 토픽의 특정 파티션을 동일한 컨슈머에게 배정해야 할 수 있습니다.
  - Sticky partitioning: stateful한 컨슈머는 파티션 리밸런싱 과정에서 파티션 재분배를 최소화 해야할 필요가 있습니다.
  - Redundant partitioning: 동일한 파티션을 여러 컨슈머에게 동시에 배정해야할 수 있습니다. 예를 들어 search index를 생성하는 경우가 있습니다.
  - metadata-based assignment: 하드웨어 스펙과 같은 메타데이터를 활용하여 파티션을 배정하는 경우가 있습니다. (ex, rack-awareness)

### client-side 파티션 분배 프로토콜
이러한 요구사항에 대응하기 위해 클라이언트에서 파티션 분배를 수행하도록 프로토콜을 개선합니다. 컨슈머 그룹 리밸런싱은 다음의 과정을 통해 이뤄집니다.

#### JoinGroupRequest
컨슈머가 스스로를 컨슈머 그룹에 등록하는 과정입니다. 브로커는 일정 기간동안 컨슈머로부터 `JoinGroupRequest`를 받으며 일정 시간 대기한 후, 하나의 컨슈머를 랜덤하게 선택하여 리더로 지정합니다. 선출된 리더는 컨슈머에게 파티션을 분배할 수 있는 권한을 부여 받습니다. `JoinGroupResponse`를 통해 리더 선출에 참여한 컨슈머는 자신이 리더로 선출되었는지 여부를 확인할 수 있습니다. 이 과정에서 컨슈머 그룹 리더가 어떤 파티션 분배 정책을 사용할지 결정됩니다. 파티션 분배 정책은 브로커가 결정하는데, 모든 컨슈머가 공통적으로 사용할 수 있는 분배 정책을 사용합니다. 이는 클라이언트 구현체가 보유하고 있는 파티션 분배 정책의 차이로 인해 발생할 수 있는 불일치를 방지하기 위함입니다.

![](/images/kafka-client-side-partition-assignment/image1.png)

#### SyncGroupRequest
리더가 멤버들에게 파티션을 분배하고 이를 동기화 시키는 단계입니다. 모든 컨슈머는 브로커에 `SyncGroupRequest`를 전송합니다. 브로커는 리더 컨슈머로부터 파티션 분배 결과를 받아서 이를 다른 멤버들에게 공유하는 역할을 수행합니다. `SyncGroupResponse`의 `MemberState` 필드에는 해당 멤버에게 배정된 파티션 정보가 포함되어 있습니다.

![](/images/kafka-client-side-partition-assignment/image2.png)

![](/images/kafka-client-side-partition-assignment/image3.png)

### Coordination State Machine
coordinator 브로커는 컨슈머 그룹 리밸런싱을 제어하기 위한 state machine을 관리합니다.
- Down: 아직 컨슈머 그룹에 멤버가 없는 상태
- Initialize: Zookeeper로부터 상태를 읽어서 coordinator를 초기화 시키는 단계
- Stable: active generation을 보유한 상태이거나, 아직 active generation이 없어서 새로운 `JoinGroupRequest`를 받아야 하는 상태
- Joining: 새로운 멤버로 부터 `JoinGroupRequest`를 수신하여 신규 generation을 생성하는 상태
- AwaitingSync: Joining 단계가 끝나고, 리더 컨슈머로부터 `SyncGroupRequest`를 수신할 때까지 임시로 대기하는 상태

![](/images/kafka-client-side-partition-assignment/image4.png)

### Implementation
client-side 파티션 분배 정책은 kafka 클라이언트 라이브러리에서 확인할 수 있습니다. Jav에서는 apache kafka-client를 기준으로 `ConsumerPartitionAssignor` ,`RangePartitionAssignor`, `CooperativeStickyAssignor`, `ConsumerCoordinator` 등의 클래스에서 위 프로토콜과 관련된 구현체를 확인할 수 있습니다. apache kafka-client 3.4.1 버전을 기준으로는 `RangePartitionAssignor가` 기본 값으로 적용되는데, 해당 `RangePartitionAssinor`는 소위 stop-the-world를 유발하는 파티션 분배 정책입니다. `CooperativeStickyAssignor`는 컨슈머 그룹 리밸런싱에 따른 stop-the-world를 개선한 파티션 분배 정책입니다.
