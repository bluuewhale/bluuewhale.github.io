+++
title = '[KIP-932] Queues for Kafka'
date = '2025-11-13T18:24:57+09:00'
draft = false
translationKey = 'kip-932-queues-for-kafka'
slug = 'kip-932-queues-for-kafka'
description = '파티션과 컨슈머의 1:1 결합을 깨고, 여러 컨슈머가 하나의 파티션을 나눠 처리할 수 있게 해주는 Kafka의 공유 그룹(Share Group)을 살펴봅니다.'
tags = ['Kafka', 'KIP', 'Queue', 'Distributed Systems']
categories = ['Kafka', 'Distributed Systems']

[cover]
image = 'images/kip-932-queues-for-kafka/image3.png'
hiddenInSingle = true
+++

카프카의 파티션은 오랫동안 한 번에 하나의 컨슈머만 처리할 수 있다는 제약을 갖고 있었습니다. 그 결과 파티션 개수와 컨슈머 개수가 강하게 묶여 있었고, 처리량을 늘리려면 필요 이상으로 파티션을 쪼개야 하는 경우가 잦았습니다. 여러 컨슈머가 하나의 파티션을 나눠 처리하는 큐(queue) 스타일이 더 어울리는 워크로드도 분명 있는데, 기존 구조로는 그것을 흉내 내기가 어려웠습니다. [KIP-932: Queues for Kafka](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)는 공유 그룹(Share Group)이라는 새로운 그룹 유형을 도입해서 이 제약을 풀어냅니다.

## Share Group

공유 그룹은 기존 컨슈머 그룹의 대안으로 등장한 그룹 유형(`share`)입니다. 가장 큰 차이는 여러 컨슈머가 하나의 파티션을 동시에 공유할 수 있다는 점입니다. 그룹 안의 컨슈머 수가 토픽의 전체 파티션 개수를 넘어설 수도 있고, ack는 레코드 단위로 이루어지며, 메시지가 전달된 횟수도 함께 기록됩니다.

공유 그룹의 컨슈머가 레코드를 읽으면, 그 레코드는 기본값 30초 동안 acquired 상태가 되어 다른 컨슈머는 손댈 수 없습니다. 컨슈머는 이 레코드에 대해 네 가지 동작을 할 수 있습니다. 처리에 성공했다고 알리는 acknowledge, 다른 컨슈머가 다시 가져갈 수 있도록 lock을 풀어주는 release, 아예 처리 불가능하다고 판단해 archived 상태로 넘겨버리는 reject, 그리고 지정된 시간 동안 아무 동작도 하지 않으면 자동으로 풀리는 auto-release입니다. 정해진 재시도 횟수를 넘기면 메시지는 archive 상태가 되어 더 이상 어떤 컨슈머에게도 전달되지 않습니다.

## 레코드의 생애주기

공유 그룹에서 레코드는 네 가지 상태를 오갑니다. 아직 전달되지 않아 컨슈머에게 넘겨줄 수 있는 Available, 특정 컨슈머에게 전달되어 시간 제한이 걸린 lock처럼 붙잡혀 있는 Acquired, 처리가 끝나 확정된 Acknowledged, 그리고 재시도 횟수를 초과했거나 처리가 끝나 더는 전달되지 않는 Archived입니다.

![](/images/kip-932-queues-for-kafka/image1.png)

## Share-Partition

공유 그룹은 Share-Partition이라는 단위로 관리됩니다. Share-Partition은 Share-Group과 Topic-Partition의 매핑 정보를 담고 있고, in-flight 레코드를 관리하고 상태를 추적하는 기본 단위 역할을 합니다.

여기서 핵심이 되는 두 오프셋이 SPSO(Share-Partition Start Offset)와 SPEO(Share-Partition End Offset)입니다. SPSO는 소비 가능한 구간의 시작점으로, 메시지들이 acknowledge될 때마다 점진적으로 오른쪽으로 밀립니다. SPEO는 현재 in-flight window의 끝점으로, 컨슈머들이 메시지를 fetch할 때마다 오른쪽으로 밀립니다. 이 둘 사이의 구간이 바로 in-flight window입니다.

![](/images/kip-932-queues-for-kafka/image2.png)

## 아키텍처

![](/images/kip-932-queues-for-kafka/image3.png)

공유 그룹을 구성하는 컴포넌트는 크게 네 가지입니다.

**Group Coordinator**는 공유 그룹의 멤버 목록을 관리하고, SimpleAssignor를 통해 각 멤버에게 배정할 Topic-Partition을 결정합니다. `ShareGroupPartitionMetadata`, `ShareGroupMemberMetadata`, `ShareGroupStatePartitionMetadata` 같은 공유 그룹의 메타데이터는 `__consumer_offsets` 토픽에 기록됩니다.

**Share-Partition Leader**는 브로커 내부의 컴포넌트로, replica manager에서 레코드를 읽어 컨슈머에게 전달하는 역할을 합니다. Share-Partition의 상태는 Share-Group State Persister를 통해 기록합니다.

**Share-Group State Persister**는 Share Coordinator와 통신하기 위한 컴포넌트입니다.

**Share Coordinator**는 Share-Partition 상태의 영속성을 관리하는 컴포넌트로, 내부 토픽(`__share_group_state`)에 `ShareSnapshot`, `ShareUpdate` 같은 상태를 기록합니다.

## 공유 그룹 멤버십

공유 그룹의 핵심 프로토콜은 [KIP-848: The Next Generation of the Consumer Rebalance Protocol](/posts/kip-848-next-generation-consumer-rebalance-protocol/)을 기반으로 합니다. 멤버십은 그룹 코디네이터가 관리하고, 컨슈머는 하트비트로 참여와 탈퇴 의사를 전달합니다. 다만 공유 그룹은 서버 측 Assignor(`org.apache.kafka.coordinator.group.share.SimpleAssignor`)만 지원하며, fencing 개념이 아예 존재하지 않기 때문에 기존 컨슈머 그룹보다 리밸런싱의 영향 범위가 작고 단순합니다.

리밸런싱 자체는 KIP-848과 같은 세 종류의 epoch으로 조율됩니다. Group Epoch은 멤버의 가입·탈퇴, subscription 변경, assignor 업데이트, 파티션 메타데이터 변경이 있을 때마다 증가하며 리밸런싱을 유발합니다. Assignment Epoch은 그룹 코디네이터가 Group Epoch을 기반으로 새 Target Assignment를 계산할 때 붙는 번호입니다. 각 멤버는 자신의 Current Assignment가 Target Assignment와 같아질 때까지 점진적으로 수렴하며, 이 과정이 Member Epoch에 반영됩니다.

한 가지 눈에 띄는 차이는, 공유 그룹에는 Static Membership 개념이 아예 없다는 점입니다.

## SimpleAssignor

공유 그룹은 파티션을 분배할 Assignor를 지정해야 하는데, KIP-932는 지금 단 하나의 구현체, `SimpleAssignor`만 제공합니다. 이 Assignor는 파티션마다 배정된 컨슈머 수가 최대한 균등해지는 방향으로 동작합니다. 실제로 멤버가 하나씩 늘어날 때 파티션이 어떻게 재배정되는지 살펴보면 동작 방식을 구체적으로 이해할 수 있습니다.

| 상태 변화 | 구독 중인 멤버 | Topic:Partitions | 할당 변경 | 최종 할당 |
|---|---|---|---|---|
| M1이 T1을 구독 | M1 | T1:0 | 0번 파티션을 M1에게 할당 | M1 → T1:0 |
| M2도 T1을 구독 | M1, M2 | T1:0 | 0번 파티션을 M2와 공동 할당 | M1 → T1:0 / M2 → T1:0 |
| T1에 파티션 3개 추가 | M1, M2 | T1:0~3 | 2번을 M1에, 1·3번을 M2에 할당하고 M2에서 0번 회수 | M1 → T1:0,2 / M2 → T1:1,3 |
| M3가 T1을 구독 | M1, M2, M3 | T1:0~3 | M1에서 2번을 회수해 M3에 할당 | M1 → T1:0 / M2 → T1:1,3 / M3 → T1:2 |
| M4가 T1을 구독 | M1~M4 | T1:0~3 | M2에서 3번을 회수해 M4에 할당 | M1→T1:0 / M2→T1:1 / M3→T1:2 / M4→T1:3 |
| M5~M8이 T1을 구독 | M1~M8 | T1:0~3 | 0,1,2,3번을 각각 M5,M6,M7,M8에도 공동 할당 | 8명이 파티션 4개를 2명씩 나눠 가짐 |
| M2를 제외한 전원 탈퇴 | M2 | T1:0~3 | 0,2,3번을 M2에게 할당 | M2 → T1:0,1,2,3 |

파티션 수보다 멤버가 많아지면 여러 멤버가 같은 파티션을 공유하고, 멤버가 줄어들면 남은 멤버가 파티션을 흡수하는 식으로 계속 균형을 맞춰가는 것을 볼 수 있습니다.

## 순서 보장은 없다

공유 그룹은 여러 컨슈머가 하나의 파티션을 동시에 처리할 수 있는 구조이다 보니, 레코드의 순서를 보장하지 않습니다. 순서가 중요한 워크로드라면 기존 컨슈머 그룹을 쓰는 것이 맞습니다.

## 배치 처리

기본적으로는 레코드 단위로 생애주기가 관리되지만, 처리량을 끌어올리기 위해 배치 단위로도 다룰 수 있습니다. 레코드 배치를 한 번에 가져오고, 처리를 마친 뒤 배치에 속한 모든 레코드에 대해 한꺼번에 ack를 보내는 방식입니다.

## 트랜잭션 레코드 읽기와 exactly-once

기존 컨슈머 그룹에서는 컨슈머마다 격리 수준(isolation level)을 다르게 지정할 수 있었지만, 공유 그룹에서는 그룹 단위로만 지정할 수 있습니다. 그리고 현재로서는 exactly-once semantic을 지원하지 않습니다. 다만 이를 지원하는 방향으로 고민이 이어지고 있는 것으로 보입니다.

![](/images/kip-932-queues-for-kafka/image6.png)

## 예시 코드

```java
Properties props = new Properties();
props.setProperty("bootstrap.servers", "localhost:9092");
props.setProperty("group.id", "myshare");
props.setProperty("share.acknowledgement.mode", "explicit");
 
KafkaShareConsumer<String, String> consumer = new KafkaShareConsumer<>(props, new StringDeserializer(), new StringDeserializer());
consumer.subscribe(Arrays.asList("foo"));
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));    // Return a batch of acquired records
    for (ConsumerRecord<String, String> record : records) {
        try {
            doProcessing(record);
            consumer.acknowledge(record, AcknowledgeType.ACCEPT);                       // Mark the record as processed successfully
        } catch (Exception e) {
            consumer.acknowledge(record, AcknowledgeType.REJECT);                       // Mark the record as unprocessable
        }
    }
    consumer.commitAsync();                                                             // Commit the acknowledgements of all the records in the batch
}
```

`KafkaShareConsumer`로 레코드를 poll하고, 처리 결과에 따라 `ACCEPT` 또는 `REJECT`로 acknowledge한 뒤, 배치 단위로 커밋하는 흐름이 기존 `KafkaConsumer`와 크게 다르지 않다는 것을 알 수 있습니다.

## 현재 상태

[Kafka for Queues 기능은 4.0 버전에서 Early Access로 릴리즈](https://cwiki.apache.org/confluence/display/KAFKA/Queues+for+Kafka+%28KIP-932%29+-+Early+Access+Release+Notes)되었습니다. 아직 RPC 스펙이 확정되지 않았고, 4.1 버전에서 확정하는 것을 목표로 하고 있습니다. 그래서 지금 단계에서는 프로덕션 환경에 적용하는 것을 권장하지 않습니다.

## 레퍼런스
- [KIP-932: Queues for Kafka](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)
- [Queues for Kafka (KIP-932) - Early Access Release Notes](https://cwiki.apache.org/confluence/display/KAFKA/Queues+for+Kafka+%28KIP-932%29+-+Early+Access+Release+Notes)
- [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A%2BThe%2BNext%2BGeneration%2Bof%2Bthe%2BConsumer%2BRebalance%2BProtocol)
