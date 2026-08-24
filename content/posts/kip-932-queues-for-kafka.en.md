+++
title = '[KIP-932] Queues for Kafka'
date = '2025-11-13T18:24:57+09:00'
draft = false
translationKey = 'kip-932-queues-for-kafka'
slug = 'kip-932-queues-for-kafka-en'
aliases = ['/posts/kip-932-queues-for-kafka-en/']
description = 'How Share Groups break the 1:1 coupling between Kafka partitions and consumers, letting multiple consumers split the work on a single partition.'
tags = ['Kafka', 'KIP', 'Queue', 'Distributed Systems']
categories = ['Kafka', 'Distributed Systems']

[cover]
image = 'images/kip-932-queues-for-kafka/image3.png'
hiddenInSingle = true
+++

A Kafka partition has always been limited to one consumer at a time. That coupled partition count and consumer count tightly together, and scaling throughput often meant splitting a topic into far more partitions than the data itself justified. Plenty of workloads fit a queue-style model better, where several consumers split the work on one partition, but the old structure had no way to express that. [KIP-932: Queues for Kafka](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka) removes that constraint with a new group type: the Share Group.

## Share Group

A Share Group is a new group type (`share`) that sits alongside the traditional consumer group. The biggest difference: multiple consumers can share a single partition at the same time. A group's consumer count can exceed the topic's total partition count, acknowledgment happens per record, and each message tracks how many times it's been delivered.

When a consumer in a Share Group reads a record, that record enters the Acquired state for a fixed window (30 seconds by default), during which no other consumer can touch it. From there, a consumer can take one of four actions: acknowledge, marking the record processed successfully; release, unlocking it so another consumer can pick it up; reject, marking it unprocessable and moving it to Archived; or do nothing, in which case it auto-releases once the window expires. Once a record exceeds its configured retry count, it moves to Archived and stops being delivered to anyone.

## Record Lifecycle

A record in a Share Group moves through four states: Available, meaning it hasn't been delivered and can go to any consumer; Acquired, meaning it's held by a specific consumer under something like a time-limited lock; Acknowledged, meaning processing finished successfully; and Archived, meaning it either exceeded its retry count or finished processing, and won't be delivered again.

![](/images/kip-932-queues-for-kafka/image1.png)

## Share-Partition

A Share Group is managed at the granularity of a Share-Partition, which holds the mapping between a Share-Group and a Topic-Partition and serves as the basic unit for tracking in-flight records and state.

Two offsets sit at the center of this: SPSO (Share-Partition Start Offset) and SPEO (Share-Partition End Offset). SPSO marks the start of the consumable range and moves right as messages get acknowledged. SPEO marks the end of the current in-flight window and moves right as consumers fetch more messages. The span between them is the in-flight window.

![](/images/kip-932-queues-for-kafka/image2.png)

## Architecture

![](/images/kip-932-queues-for-kafka/image3.png)

Four components make up a Share Group.

The **Group Coordinator** manages the Share Group's member list and decides which Topic-Partitions go to which member, via the SimpleAssignor. It records the group's metadata (`ShareGroupPartitionMetadata`, `ShareGroupMemberMetadata`, `ShareGroupStatePartitionMetadata`, and so on) into the `__consumer_offsets` topic.

The **Share-Partition Leader** is a broker-internal component that reads records from the replica manager and delivers them to consumers. It records Share-Partition state through the Share-Group State Persister.

The **Share-Group State Persister** is the component that communicates with the Share Coordinator.

The **Share Coordinator** manages the persistence of Share-Partition state, writing state (`ShareSnapshot`, `ShareUpdate`) into an internal topic, `__share_group_state`.

## Share Group Membership

The core protocol behind Share Groups builds on [KIP-848: The Next Generation of the Consumer Rebalance Protocol](/posts/kip-848-next-generation-consumer-rebalance-protocol-en/). Membership is managed by the group coordinator, and consumers signal joining or leaving through heartbeats. Share Groups only support a server-side assignor (`org.apache.kafka.coordinator.group.share.SimpleAssignor`), and since the concept of fencing doesn't exist here at all, rebalancing has a smaller, simpler blast radius than in a traditional consumer group.

Rebalancing itself is coordinated by the same three epochs as KIP-848. The Group Epoch increments, triggering a rebalance, whenever a member joins or leaves, a subscription changes, an assignor updates, or partition metadata changes. The Assignment Epoch is the number the group coordinator attaches when it computes a new Target Assignment from the Group Epoch. Each member incrementally converges its Current Assignment toward that Target Assignment, and that progress is reflected in its Member Epoch.

One notable difference: Share Groups have no concept of Static Membership at all.

## SimpleAssignor

A Share Group needs an assignor to distribute partitions, and KIP-932 currently ships exactly one: `SimpleAssignor`. It works to keep the number of consumers assigned to each partition as even as possible. Walking through what happens as members join one at a time makes the behavior clear.

| State Change | Subscribed Members | Topic:Partitions | Assignment Change | Resulting Assignment |
|---|---|---|---|---|
| M1 subscribes to T1 | M1 | T1:0 | Assign partition 0 to M1 | M1 → T1:0 |
| M2 also subscribes to T1 | M1, M2 | T1:0 | Co-assign partition 0 to M2 | M1 → T1:0 / M2 → T1:0 |
| T1 gets 3 more partitions | M1, M2 | T1:0-3 | Assign 2 to M1, 1 and 3 to M2, reclaim 0 from M2 | M1 → T1:0,2 / M2 → T1:1,3 |
| M3 subscribes to T1 | M1, M2, M3 | T1:0-3 | Reclaim 2 from M1, assign to M3 | M1 → T1:0 / M2 → T1:1,3 / M3 → T1:2 |
| M4 subscribes to T1 | M1-M4 | T1:0-3 | Reclaim 3 from M2, assign to M4 | M1→T1:0 / M2→T1:1 / M3→T1:2 / M4→T1:3 |
| M5-M8 subscribe to T1 | M1-M8 | T1:0-3 | Co-assign 0,1,2,3 to M5,M6,M7,M8 respectively | 8 members split 4 partitions, two members per partition |
| Everyone but M2 leaves | M2 | T1:0-3 | Assign 0, 2, 3 to M2 | M2 → T1:0,1,2,3 |

Once members outnumber partitions, multiple members end up sharing a partition. As members leave, the ones remaining absorb the freed-up partitions. The assignor keeps rebalancing toward even coverage.

## No Ordering Guarantee

Because multiple consumers can process the same partition at once, Share Groups don't guarantee record ordering. If ordering matters for your workload, stick with a regular consumer group.

## Batch Processing

By default, lifecycle is managed per record, but you can also work in batches to push throughput higher: fetch a batch of records, process it, then acknowledge every record in the batch at once.

## Reading Transactional Records and Exactly-Once

A regular consumer group lets each consumer set its own isolation level. In a Share Group, isolation level can only be set at the group level. And right now, Share Groups don't support exactly-once semantics, though there are signs the direction is being considered.

![](/images/kip-932-queues-for-kafka/image6.png)

## Example Code

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

`KafkaShareConsumer` polls records, acknowledges each one `ACCEPT` or `REJECT` depending on how processing went, then commits the whole batch at once. The shape isn't far from a regular `KafkaConsumer` loop.

## Current Status

[Kafka for Queues shipped as Early Access in 4.0](https://cwiki.apache.org/confluence/display/KAFKA/Queues+for+Kafka+%28KIP-932%29+-+Early+Access+Release+Notes). The RPC spec isn't finalized yet, with 4.1 targeted for that. At this stage, it's not recommended for production use.

## References
- [KIP-932: Queues for Kafka](https://cwiki.apache.org/confluence/display/KAFKA/KIP-932%3A+Queues+for+Kafka)
- [Queues for Kafka (KIP-932) - Early Access Release Notes](https://cwiki.apache.org/confluence/display/KAFKA/Queues+for+Kafka+%28KIP-932%29+-+Early+Access+Release+Notes)
- [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A%2BThe%2BNext%2BGeneration%2Bof%2Bthe%2BConsumer%2BRebalance%2BProtocol)
