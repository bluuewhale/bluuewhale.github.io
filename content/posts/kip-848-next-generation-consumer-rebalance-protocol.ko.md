+++
title = '[KIP-848] The Next Generation of the Consumer Rebalance Protocol'
date = '2025-11-13T18:17:32+09:00'
draft = false
translationKey = 'kip-848-next-generation-consumer-rebalance-protocol'
slug = 'kip-848-next-generation-consumer-rebalance-protocol'
description = '리밸런싱의 책임을 클라이언트에서 브로커로 옮기고, epoch 기반의 점증적 수렴으로 downtime을 없애려는 KIP-848의 설계를 정리합니다.'
tags = ['Kafka', 'KIP', 'Consumer Group', 'Distributed Systems']
categories = ['Kafka', 'Distributed Systems']
+++

카프카 4.0에서 GA된 [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A%2BThe%2BNext%2BGeneration%2Bof%2Bthe%2BConsumer%2BRebalance%2BProtocol)을 정리해봅니다. 컨슈머 그룹 멤버가 바뀌어도 downtime을 거의 없애는 것, 그리고 리밸런싱의 주요 책임을 클라이언트에서 브로커로 옮기는 것이 이 KIP의 핵심 목표입니다.

## 배경

도입된 지 8년이 지난 기존 컨슈머 그룹 리밸런싱 프로토콜은 몇 가지 구조적인 한계에 부딪힌 상태였습니다.

가장 큰 문제는 클라이언트에 지나치게 많은 역할을 부여한(thick client) 설계였습니다. 컨슈머 그룹 리밸런싱에 버그가 있으면 그 수정이 클라이언트 쪽에서 이뤄져야 하는데, 클라우드 서비스를 운영하는 입장에서는 사용자의 클라이언트를 강제로 고칠 수 없으니 이는 매우 까다로운 제약입니다. 대부분의 로직이 클라이언트에서 실행되다 보니, 문제가 생겨도 서버 쪽 로그만으로는 원인을 진단하기 어렵다는 점 또한 문제를 더 어렵게 만들었습니다.

또 다른 문제는 멤버 하나의 변화, 즉 가입이나 탈퇴, 실패 하나만으로도 그룹 전체의 리밸런스가 촉발된다는 점이었습니다. Cooperative rebalancing 같은 개선책이 이미 적용되어 있었지만 여전히 제한적이었습니다. 리밸런싱이 진행되는 동안에는 오프셋 커밋 자체가 불가능했고, 이로 인해 일부 애플리케이션의 처리가 그대로 막혀버리는 일이 벌어졌습니다.

여러 KIP을 거치며 부분적으로 개선을 이어왔지만, 그 과정에서 복잡성만 쌓여갔고 결국 일관된 재설계가 필요한 시점에 이르렀습니다.

## 설계 목표

KIP-848이 지향하는 지점은 명확합니다. 멤버 목록에 변화가 생기더라도 모든 컨슈머가 영향을 받는 대신, 실제로 할당된 파티션이 바뀌는 멤버만 부분적으로 영향을 받는 진짜 점증적(incremental)·협력적(cooperative) 구조를 만드는 것입니다. 그리고 리밸런싱의 복잡성 대부분을 클라이언트에서 브로커 내부의 group coordinator로 옮겨, 클라이언트 업그레이드 없이도 문제를 수정하고 브로커 로그만으로 문제를 진단할 수 있게 만드는 것입니다. 동시에 Kafka Streams처럼 클라이언트 측 assignor 로직이 필수적인 경우도 계속 지원해야 하고, 새 프로토콜로의 마이그레이션은 점진적으로 이뤄질 수 있어야 하며, at-least-once와 조건부 exactly-once를 모두 보장해야 합니다.

## 무엇이 바뀌었나

### 선언적 할당

가장 근본적인 변화는 할당 방식 자체입니다. 이제 그룹 코디네이터(브로커)가 Target Assignment, 즉 각 멤버가 최종적으로 도달해야 할 목표 상태를 지정합니다. 각 멤버는 자신의 현재 상태(Current Assignment)를 이 Target Assignment와 같아질 때까지 점진적으로 수렴시켜 나갑니다. 예전처럼 "지금 당장 리밸런스에 참여해서 전체 할당을 다시 계산"하는 대신, "목표 상태를 향해 조금씩 움직인다"는 쪽으로 관점이 바뀐 셈입니다.

### 이벤트 루프 구조

그룹 코디네이터에는 이벤트 루프 구조가 도입됩니다. 여러 요청이 동시에 몰려도 동시성 문제를 단순하게 풀 수 있다는 것이 이 구조를 채택한 이유입니다.

### Epoch으로 리밸런싱을 조율하기

새 프로토콜의 실질적인 핵심은 세 종류의 epoch입니다.

Group Epoch은 현재 그룹 메타데이터의 버전 번호입니다. 멤버의 가입·탈퇴, subscription 변경, assignor 관련 업데이트, 파티션 메타데이터 변경(새 토픽 생성이나 파티션 수 변경 등) 중 하나라도 발생하면 이 값이 증가하고, 리밸런싱이 트리거됩니다. Group Epoch이 Assignment Epoch보다 커지면, 그룹 코디네이터는 최신 그룹 메타데이터를 바탕으로 새로운 Target Assignment를 계산합니다. 이때 어떤 Assignor를 쓸지 고르는 Assignor Selection 과정을 거치는데, 서버 측 Assignor로는 토픽 간에 같은 파티션 번호를 배정하는 range 방식과 파티션을 무작위로 배정하는 uniform 방식이 있고, 어느 쪽이든 기본적으로 sticky하게 동작해 파티션 변화를 최소화합니다. 클라이언트 측 Assignor를 쓰면 기존 컨슈머 그룹 리밸런싱 프로토콜과 비슷하게 동작합니다.

Assignment Epoch은 그룹 코디네이터가 새 Target Assignment를 계산할 때 함께 붙이는 번호입니다. Target Assignment가 확정되면 이 값을 Group Epoch과 같은 값으로 맞춥니다.

Member Epoch은 각 멤버에게 "지금 몇 번째 Target Assignment까지 반영했는지"를 알려주는 번호입니다. Target Assignment가 정해지면 멤버들은 그쪽으로 점진적으로 수렴해 갑니다. 그룹 코디네이터는 더 이상 필요 없어진 파티션을 멤버에게서 회수하도록 요청하고, 회수한 파티션을 필요한 다른 멤버에게 넘겨줍니다. 멤버가 Target Assignment에 완전히 수렴하면, 그제야 자신의 Member Epoch을 Assignment Epoch과 같은 값으로 갱신합니다.

### Heartbeat와 세션 관리

이 흐름을 뒷받침하기 위해 `ConsumerGroupHeartbeat`라는 새 API가 추가됩니다. 멤버는 이 API로 주기적인 heartbeat 요청을 보내 세션을 유지하고, 그룹 코디네이터는 이 요청을 통해 멤버 상태나 subscription 변화를 파악합니다. heartbeat 응답에는 회수/할당된 파티션 정보와 함께, 클라이언트 측 assignor를 사용해야 한다는 신호인 `ShouldComputeAssignment` 같은 값이 담겨 돌아옵니다.

## 실제로 무엇이 바뀌었나

새 컨슈머 프로토콜 자체는 Kafka 3.7에서 처음 공개되었고, [4.0에서 정식 GA](https://www.confluent.io/blog/latest-apache-kafka-release/)되었습니다.

브로커 쪽에서는 4.0 이상에서 새 컨슈머 프로토콜이 기본으로 사용되며, `group.version` 플래그로 사용 여부를 제어할 수 있고 `group.consumer.assignors` 설정으로 어떤 Assignor를 쓸지 지정할 수 있습니다.

컨슈머 쪽은 조금 다릅니다. 4.0 이상이라도 기본값은 여전히 기존 프로토콜이며, `group.protocol` 설정을 `consumer`로 명시적으로 바꿔야 새 프로토콜이 적용됩니다. `group.remote.assignor` 설정으로 서버 측 Assignor 중 어떤 것을 쓸지 고를 수 있고, 새 프로토콜을 쓰면 [새로운 메트릭들](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1068%3A+New+metrics+for+the+new+KafkaConsumer)도 함께 추가됩니다. 대신 `heartbeat.interval.ms`, `session.timeout.ms`, `partition.assignment.strategy`, `enforceRebalance(String)`/`enforceRebalance()` 설정은 새 프로토콜에서는 그냥 무시됩니다.

## 마이그레이션

오프라인으로 옮기는 방법은 간단합니다. 그룹 안의 멤버가 모두 사라지면 컨슈머 그룹은 자동으로 `Classic`에서 `Consumer` 타입으로 바뀝니다. 그러니 모든 멤버를 내린 뒤 `group.protocol=consumer` 설정을 적용해서 다시 띄우면 마이그레이션이 끝납니다.

온라인 마이그레이션도 가능합니다. rolling 방식으로 `group.protocol=consumer` 설정을 적용한 컨슈머를 하나씩 배포하면 되는데, 그룹에 새 프로토콜을 쓰는 멤버가 단 하나라도 참여하는 순간 그룹 전체가 `Consumer` 타입으로 전환됩니다. 기존 `Classic` 컨슈머와의 하위 호환성은 보장되지만, assignor에서 커스텀 메타데이터를 쓰는 경우는 예외입니다.

## 아직 남은 한계

클라이언트 측 Assignor 기능은 아직 지원되지 않고([KAFKA-18327](https://issues.apache.org/jira/browse/KAFKA-18327)), rack-aware assignment도 완전히 지원되지는 않습니다([KAFKA-17747](https://issues.apache.org/jira/browse/KAFKA-17747)). 지금은 토픽의 파티션 개수가 바뀔 때만 리밸런싱이 일어나는데, 이 한계는 [KIP-1101](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1101%3A+Trigger+rebalance+on+rack+topology+changes)을 통해 해소되어 4.1 버전에 반영되었습니다.

## 데이터 모델

새 프로토콜이 주고받는 핵심 객체들을 참고용으로 정리해 둡니다.

**Consumer Group**

| Name | Type | Description |
|---|---|---|
| Group ID | string | 컨슈머가 설정한 그룹 ID. 그룹을 고유하게 식별합니다. |
| Group Epoch | int32 | 그룹의 현재 epoch. 새로운 할당이 필요할 때 그룹 코디네이터가 증가시킵니다. |
| Members | []Member | 그룹에 속한 멤버들의 집합. |
| Partitions Metadata | []PartitionMetadata | 그룹이 구독 중인 파티션들의 메타데이터. 파티션 메타데이터 변경을 감지하는 데 쓰입니다. |

**Member**

| Name | Type | Description |
|---|---|---|
| Member ID | string | 멤버의 고유 식별자. 첫 heartbeat 요청 시 코디네이터가 생성하며, 멤버의 생애주기 동안 계속 사용됩니다. |
| Instance ID | string | 컨슈머가 설정한 instance ID. |
| Rack ID | string | 컨슈머가 설정한 rack ID. |
| Client ID / Client Host | string | 컨슈머가 설정한 client ID 및 호스트. |
| Subscribed Topic Names / Regex | []string / string | 현재 구독 중인 토픽 이름 목록 또는 구독 정규식. |
| Server Assignor | string | 그룹이 사용하는 서버 측 assignor. |
| Client Assignors | []Assignor | 멤버가 지원하는 클라이언트 측 assignor 목록. 순서가 곧 우선순위입니다. |

**Target Assignment**

| Name | Type | Description |
|---|---|---|
| Assignment Epoch | int32 | 이 할당을 생성할 때 사용된 그룹 epoch. 궁극적으로 Group Epoch과 같아집니다. |
| Members | []Member | 멤버별 파티션 할당 정보. |

**Current Assignment**

| Name | Type | Description |
|---|---|---|
| Member Epoch | int32 | 이 멤버가 현재 사용 중인 할당의 epoch. 오프셋 커밋 등에서 멤버를 펜싱하는 데 사용됩니다. |
| Partitions | []TopicIdPartition | 멤버가 현재 실제로 사용 중인 파티션 목록. |

## 레퍼런스
- [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A%2BThe%2BNext%2BGeneration%2Bof%2Bthe%2BConsumer%2BRebalance%2BProtocol)
- [Apache Kafka 4.0 Release: Default KRaft, Queues, Faster Rebalances](https://www.confluent.io/blog/latest-apache-kafka-release/)
- [Apache Kafka Documentation - Consumer Rebalance Protocol](https://kafka.apache.org/40/documentation.html#consumer_rebalance_protocol)
- [KIP-1101: Trigger rebalance on rack topology changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1101%3A+Trigger+rebalance+on+rack+topology+changes)
