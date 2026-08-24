+++
title = '[KIP-848] The Next Generation of the Consumer Rebalance Protocol'
date = '2025-11-13T18:17:32+09:00'
draft = false
translationKey = 'kip-848-next-generation-consumer-rebalance-protocol'
slug = 'kip-848-next-generation-consumer-rebalance-protocol-en'
aliases = ['/posts/kip-848-next-generation-consumer-rebalance-protocol-en/']
description = 'How KIP-848 moves rebalancing responsibility from the client to the broker and uses epoch-based incremental convergence to eliminate rebalance downtime.'
tags = ['Kafka', 'KIP', 'Consumer Group', 'Distributed Systems']
categories = ['Kafka', 'Distributed Systems']
+++

[KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A%2BThe%2BNext%2BGeneration%2Bof%2Bthe%2BConsumer%2BRebalance%2BProtocol) reached GA in Kafka 4.0. Its two central goals: cut rebalance downtime to nearly zero when consumer group membership changes, and move most of the responsibility for rebalancing from the client to the broker.

## Background

The consumer group rebalancing protocol had been around for eight years, and it was running into structural limits.

The biggest one was a thick-client design that put too much responsibility on the client. When a bug showed up in consumer group rebalancing, fixing it meant fixing the client, and if you're running a cloud service, you can't force your users to patch their own clients. Since most of the logic ran on the client side, diagnosing problems from server-side logs alone was often impossible.

The second problem: a single member's change, joining, leaving, or failing, triggered a rebalance for the entire group. Cooperative rebalancing had already improved on this, but only so much. A rebalance in progress blocked offset commits entirely, stalling processing in some applications outright.

A series of KIPs had chipped away at these problems incrementally, but that accumulated complexity, and it became clear the protocol needed a coherent redesign rather than another patch.

## Design Goals

KIP-848's target is specific. Build a genuinely incremental, cooperative structure where a change to the member list doesn't affect every consumer, only the members whose actual partition assignments change. Move most of the rebalancing complexity out of the client and into the broker's group coordinator, so bugs can be fixed without a client upgrade and operators can diagnose issues from broker logs alone. At the same time, keep supporting cases like Kafka Streams, where client-side assignor logic is a hard requirement. Migration to the new protocol needs to be incremental, and the design has to guarantee at-least-once, with exactly-once available conditionally.

## What Actually Changed

### Declarative Assignment

The most fundamental shift is in how assignment itself works. The group coordinator (the broker) now specifies a Target Assignment: the end state each member should eventually reach. Each member then incrementally converges its own Current Assignment toward that target. Instead of "join the rebalance right now and recompute the whole assignment," the model becomes "move gradually toward the target state."

### An Event Loop in the Group Coordinator

The group coordinator gets an event loop. The reason for adopting it is straightforward: it makes concurrency, handling many requests arriving at once, much simpler to reason about.

### Coordinating Rebalances with Epochs

The real core of the new protocol is three kinds of epochs.

The Group Epoch is a version number for the group's current metadata. It increments, and triggers a rebalance, whenever a member joins or leaves, a subscription changes, an assignor-related update happens, or partition metadata changes (a new topic, a change in partition count, and so on). Once the Group Epoch exceeds the Assignment Epoch, the group coordinator computes a new Target Assignment from the latest group metadata. That involves an Assignor Selection step to pick which assignor to use. Server-side options include range, which assigns matching partition numbers across topics, and uniform, which assigns partitions randomly; both default to sticky behavior that minimizes partition churn. A client-side assignor behaves much like the old client-driven rebalancing protocol.

The Assignment Epoch is the number the group coordinator attaches when it computes a new Target Assignment. Once that target is finalized, the group coordinator sets this value equal to the Group Epoch.

The Member Epoch tells each member how far it's converged, specifically, which Target Assignment it currently reflects. Once a Target Assignment is set, members converge toward it incrementally. The group coordinator asks members to give up partitions they no longer need, then hands those reclaimed partitions to whichever member needs them. Once a member has fully converged to the Target Assignment, it updates its own Member Epoch to match the Assignment Epoch.

### Heartbeats and Session Management

A new `ConsumerGroupHeartbeat` API supports all of this. Members send periodic heartbeat requests through it to keep their session alive, and the group coordinator uses those requests to learn about member state or subscription changes. The heartbeat response carries back information on revoked and assigned partitions, along with signals like `ShouldComputeAssignment`, which tells the member it needs to run a client-side assignor.

## What Changed in Practice

The new consumer protocol first shipped in Kafka 3.7 and [reached GA in 4.0](https://www.confluent.io/blog/latest-apache-kafka-release/).

On the broker side, 4.0 and later use the new protocol by default. The `group.version` flag controls whether it's enabled, and `group.consumer.assignors` picks which assignor to use.

The consumer side works a bit differently. Even on 4.0+, the default is still the old protocol; you have to explicitly set `group.protocol` to `consumer` to opt in. `group.remote.assignor` lets you pick which server-side assignor to use, and switching to the new protocol also adds [new metrics](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1068%3A+New+metrics+for+the+new+KafkaConsumer). In exchange, the new protocol ignores `heartbeat.interval.ms`, `session.timeout.ms`, `partition.assignment.strategy`, and `enforceRebalance(String)`/`enforceRebalance()`.

## Migration

Migrating offline is simple. Once every member of a group disappears, the group automatically flips from `Classic` to `Consumer`. Shut down every member, apply `group.protocol=consumer`, bring them back up, and you're done.

Online migration works too. Roll out consumers configured with `group.protocol=consumer` one at a time, and the moment even one member using the new protocol joins the group, the whole group converts to `Consumer` type. Backward compatibility with existing `Classic` consumers holds, with one exception: assignors that rely on custom metadata.

## What's Still Missing

Client-side assignor support isn't there yet ([KAFKA-18327](https://issues.apache.org/jira/browse/KAFKA-18327)), and rack-aware assignment isn't fully supported either ([KAFKA-17747](https://issues.apache.org/jira/browse/KAFKA-17747)). Right now, a rebalance only triggers when a topic's partition count changes. [KIP-1101](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1101%3A+Trigger+rebalance+on+rack+topology+changes) closed that gap and landed in version 4.1.

## Data Model

For reference, here are the core objects the new protocol passes around.

**Consumer Group**

| Name | Type | Description |
|---|---|---|
| Group ID | string | The group ID configured by the consumer. Uniquely identifies the group. |
| Group Epoch | int32 | The group's current epoch. Incremented by the group coordinator whenever a new assignment is required. |
| Members | []Member | The set of members in the group. |
| Partitions Metadata | []PartitionMetadata | Metadata for the partitions the group subscribes to, used to detect metadata changes. |

**Member**

| Name | Type | Description |
|---|---|---|
| Member ID | string | The member's unique identifier, generated by the coordinator on the first heartbeat and used for the member's whole lifetime. |
| Instance ID | string | The instance ID configured by the consumer. |
| Rack ID | string | The rack ID configured by the consumer. |
| Client ID / Client Host | string | The client ID and host configured by the consumer. |
| Subscribed Topic Names / Regex | []string / string | The current set of subscribed topic names, or the subscription regex. |
| Server Assignor | string | The server-side assignor used by the group. |
| Client Assignors | []Assignor | The client-side assignors this member supports, in priority order. |

**Target Assignment**

| Name | Type | Description |
|---|---|---|
| Assignment Epoch | int32 | The group epoch used to generate this assignment. Eventually matches the Group Epoch. |
| Members | []Member | The assignment computed for each member. |

**Current Assignment**

| Name | Type | Description |
|---|---|---|
| Member Epoch | int32 | The epoch of the assignment this member currently uses. Used to fence the member, e.g. for offset commits. |
| Partitions | []TopicIdPartition | The partitions this member is actually using right now. |

## References
- [KIP-848: The Next Generation of the Consumer Rebalance Protocol](https://cwiki.apache.org/confluence/display/KAFKA/KIP-848%3A%2BThe%2BNext%2BGeneration%2Bof%2Bthe%2BConsumer%2BRebalance%2BProtocol)
- [Apache Kafka 4.0 Release: Default KRaft, Queues, Faster Rebalances](https://www.confluent.io/blog/latest-apache-kafka-release/)
- [Apache Kafka Documentation - Consumer Rebalance Protocol](https://kafka.apache.org/40/documentation.html#consumer_rebalance_protocol)
- [KIP-1101: Trigger rebalance on rack topology changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1101%3A+Trigger+rebalance+on+rack+topology+changes)
