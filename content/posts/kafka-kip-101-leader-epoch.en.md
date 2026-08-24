+++
title = 'Kafka Improvement Proposals 101 - Leader Epoch'
date = '2023-04-16T20:40:48+09:00'
draft = false
translationKey = 'kafka-kip-101-leader-epoch'
slug = 'kafka-kip-101-leader-epoch-en'
aliases = ['/posts/kafka-kip-101-leader-epoch-en/']
description = 'How high-watermark-based replication protocols in Kafka can lose data or diverge, and how KIP-101 fixes it by introducing the leader epoch.'
tags = ['Kafka', 'KIP', 'Replication', 'Distributed Systems']
categories = ['Kafka', 'Distributed Systems']

[cover]
image = 'images/kafka-kip-101-leader-epoch/image1.png'
hiddenInSingle = true
+++

This post walks through [KIP-101 — Alter Replication Protocol to use Leader Epoch rather than High Watermark for Truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation), the proposal that introduced the leader epoch to Kafka, along with some notes of my own.

![](/images/kafka-kip-101-leader-epoch/image1.png)

---

## Log Replication Protocol

Before we get to the leader epoch, let's look at how a partition log replicates from the leader broker to its followers. The exact flow shifts a bit depending on the producer's `ack` setting and `min.isr`, but I want to focus on the core mechanics here.

Assume a partition with three replicas. `broker-101` is the leader, and `broker-102` and `broker-103` are followers. `min.isr` is 1.

#### Follower Fetch Request

![](/images/kafka-kip-101-leader-epoch/image2.png)

Each follower periodically sends a `FetchRequest` to the leader to replicate the log. The request carries the offset the follower currently holds.

#### Follower Fetch Response

![](/images/kafka-kip-101-leader-epoch/image3.png)

The leader reads the offset in the follower's `FetchRequest` and returns everything written after it in the `FetchResponse`. That's how one follower catches up.

#### Committing Partition Offsets

![](/images/kafka-kip-101-leader-epoch/image4.png)

Once a message has replicated to every node, the leader commits the offset up to that point. Only after the commit can consumers read that data.

But Kafka skips sending an explicit ack for `FetchResponse` to keep RPC calls to a minimum. So how does the leader learn that a follower has replicated successfully?

It reuses the offset already in the `FetchRequest`. If a follower sends a `FetchRequest` with offset N, that alone tells the leader the follower has everything up to N. So when the leader receives a `FetchRequest` from a follower in the ISR, it checks that offset and advances the commit point accordingly, then relays the new commit offset back to the followers in the next `FetchResponse`. That committed offset is what we call the `high watermark`.

## Where the High-Watermark Protocol Breaks Down

This design has a few problems. The core one: the leader and its followers update their high watermark at different times. The leader updates it the moment it receives a `FetchRequest`, but a follower only updates its own high watermark once it receives the corresponding `FetchResponse`. That lag can lead to more than a lost message. It can break consistency outright.

Let's look at two scenarios where this bites.

### Scenario 1: High Watermark Truncation Followed by Immediate Leader Election

Here, a broker with a stale high watermark gets elected leader.

![](/images/kafka-kip-101-leader-epoch/image5.png)

Take a partition with two replicas, A and B, where B is the leader.

1. Follower A, having replicated up through offset 1 (`m2`), sends `FetchRequest(offset=2)` to leader B.
2. B sees that A is caught up through offset 1, updates its high watermark, and since there's nothing left to replicate, sends back a `FetchResponse` carrying only the updated high watermark.
3. A restarts before it receives that `FetchResponse`. On restart, it truncates everything after its own (stale) high watermark and sends a fresh `FetchRequest` to B.
4. B goes down.
5. Message `m2` is gone.

This scenario can also produce a phantom read. Until A restarted and became the new leader, `m2` was committed and consumers could read it through B.

Jun Rao proposed two straightforward fixes:

> 1. Delay the leader's high-watermark update until the follower has confirmed its own update.

He ruled this out: it adds another RPC round trip and drives up latency across the whole replication protocol, undoing the very efficiency gain Kafka got from dropping explicit acks in the first place.

> 2. When a follower restarts, don't truncate immediately. Send a fetch request to the leader first, and decide what to truncate based on the response.

This fixes the message loss in Scenario 1. But a recovery scheme that depends on the high watermark alone still can't handle the next scenario.

### Scenario 2: Replica Divergence on Restart after Multiple Hard Failures

Here, every broker goes down at once, and an incompletely replicated follower comes back first and gets elected leader.

![](/images/kafka-kip-101-leader-epoch/image6.png)

Assume broker B received the `FetchResponse` for message `m2` but never flushed it to disk. Same setup: two replicas, A as leader, B as follower.

1. A power loss or similar event takes down both A and B.
2. B restarts first and becomes leader.
3. As the new leader, B accepts and writes message `m3` from the producer.
4. A restarts as a follower.

A and B now hold different messages at the same offset. Follower A is no longer a valid replica. Worse, once A sends its `FetchRequest`, the replication protocol proceeds as if nothing were wrong. Left unchecked, this can corrupt the topic.

## Leader Epoch

KIP-101 introduces the `leader epoch` to fix both of these.

![](/images/kafka-kip-101-leader-epoch/image7.png)

Think of the leader epoch as a temporary ID for whoever currently leads a partition. It starts at 0 and increments by 1 every time a new leader gets elected. A leader change bumps the epoch, and even if the same broker returns to leadership for the same partition later, it gets a new epoch value rather than reusing the old one.

The controller manages the leader epoch for each partition, persists it in ZooKeeper, and updates it on every new leader election. The internal structure in newer Kafka versions may not match this diagram exactly. Treat it as a reference for the structure as originally proposed.

Every broker records the leader epoch and the offset at which it started, each time a new leader is elected. Think of it as a key-value log of who led and when they took office. Kafka calls this the `leader epoch sequence`. On restart, a broker uses this sequence instead of the high watermark to decide what to keep.

So instead of truncating everything after its local high watermark on restart, a broker asks the leader for its epoch and compares before deciding what to discard.

Let's revisit both scenarios with leader epoch in place.

### Scenario 1: High Watermark Truncation Followed by Immediate Leader Election

First, this is the case where an incompletely-updated follower restarts right as the leader also restarts.

![](/images/kafka-kip-101-leader-epoch/image8.png)

1. Follower A has replicated everything from leader B, but shuts down before updating its high watermark.
2. On restart, A sends a `LeaderEpochRequest` to the leader and gets back offset 2, its current position.
3. Since the length of A's local data matches that offset, A keeps its data regardless of what its high watermark says.
4. Leader B goes down.
5. A gets elected the new leader, receives leader epoch 1, and records it in its leader epoch sequence.

The outcome is the same even if B never manages to respond to the `LeaderEpochRequest` before restarting.

### Scenario 2: Replica Divergence on Restart after Multiple Hard Failures

Now, let's revisit the case where every broker goes down together and an incompletely replicated follower comes back first.

![](/images/kafka-kip-101-leader-epoch/image9.png)

1. Leader A and follower B both go down mid-replication.
2. Follower B restarts first and becomes the new leader.
3. As leader, B skips the leader-epoch exchange: there's no one to ask.
4. B accepts a new message, `m3`, from the producer.
5. A restarts as a follower.
6. A sends a `LeaderEpochRequest` to leader B and gets back epoch 1.
7. A's local offset is 2, but the new leader's epoch started at offset 1, so A discards everything from offset 1 onward, including `m2`.

With leader-epoch-based recovery, even a full cluster restart in a partially replicated state doesn't produce a consistency break. `m2` is still lost here, but that's a consequence of the `min.isr` setting, not a flaw in the recovery protocol itself.

## Wrap-up

This post covered why Kafka introduced the leader epoch and how it works under the hood. Thanks for reading.

## References
- [Data Plane: Replication Protocol](https://developer.confluent.io/learn-kafka/architecture/data-replication/)
- [KIP-101 - Alter Replication Protocol to use Leader Epoch rather than High Watermark for Truncation](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation)
