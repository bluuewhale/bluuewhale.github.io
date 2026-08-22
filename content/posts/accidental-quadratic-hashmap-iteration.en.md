+++
title = 'Accidental Quadratic Hashmap Iteration'
date = '2025-12-08T22:53:14+09:00'
draft = false
translationKey = 'accidental-quadratic-hashmap-iteration'
slug = 'accidental-quadratic-hashmap-iteration-en'
aliases = ['/posts/accidental-quadratic-hashmap-iteration-en/']
description = 'Why iterating one open-addressing hash table and reinserting its elements into another can blow up from O(n) to O(n^2), once the two tables share a hash ordering.'
tags = ['Data Structure', 'Hash Table', 'Rust', 'Performance']
categories = ['Data Structures', 'Performance']

[cover]
image = 'images/accidental-quadratic-hashmap-iteration/image3.png'
hiddenInSingle = true
+++

> This post is my own write-up of [Rust hash iteration+reinsertion](https://accidentallyquadratic.tumblr.com/post/153545455987/rust-hash-iteration-reinsertion) and the related Rust issue/PR, written the way I understood them. The original is the authoritative source, so check it directly for the exact details.

I want to walk through an interesting bug that showed up in Rust's `HashMap`. The code looks completely ordinary, yet under the right conditions, an operation that should be O(n) blows up to O(n²).

## Reproducing the Bug

Look at the code below. It inserts values 1 through 5,000,000 into a first hash map (`one`, that's T1), then iterates over `one` and reinserts every value into a second hash map (`two`, that's T2). Nothing fancy.

```rust
use std::collections::hash_set::HashSet;

fn main() {
    println!("inserting...");
    let mut one = HashSet::new();
    for i in 1..5000000 {
        one.insert(i);
    }

    println!("reinserting...");
    let mut two = HashSet::new();
    for v in one {
        two.insert(v);
    }
}
```

Both T1 and T2 insert elements one at a time, so you'd expect both to run in O(n). T1 does. T2 takes something close to O(n²). Same kind of work, wildly different cost. Why?

## When This Bug Shows Up

This isn't a problem with hash tables in general. It needs all of these conditions at once:

- An open-addressing hash table
- Linear probing to resolve collisions
- Iteration that walks the internal bucket array front to back
- A table size that's a power of two
- Bucket position computed as `hash & (capacity - 1)`
- A high load factor (say, 0.9)

The uncomfortable part is that most open-addressing hash table implementations satisfy every one of these by default. This isn't some exotic misconfiguration. It's the ordinary shape of the data structure itself.

## Why It Happens

Say `one` finishes T1 with a final size of 2n. Now picture T2 midway through, with `two` still at size n.

T2 walks `one`'s bucket array from the front and inserts each element into `two` in that order. Pull out `k0`, the element sitting at index 0 in `one`. That tells you `hash(k0) % 2n = 0` is likely. Now compute where `k0` lands in `two`: `hash(k0) % n`, which also comes out to 0 with high probability, because `hash(k0) % 2n` and `hash(k0) % n` tend to agree.

Generalize that, and for the first half of `one` (indices 0 through n-1), `hash(k) % n` and `hash(k) % 2n` keep landing on the same value. So `k0` goes into slot 0 of `two`, `k1` into slot 1, and so on. `two` fills up neatly, front to back.

![](/images/accidental-quadratic-hashmap-iteration/image1.png)

The real trouble starts on the second half (indices n through 2n-1). Take `kn`, sitting at index n in `one`, where `hash(kn) % 2n = n` with high probability. Compute where it lands in `two`: `hash(kn) % n = 0`. If `hash(kn) % 2n` equals n exactly, then subtracting n from it to get `hash(kn) % n` leaves exactly 0. By the same logic, `kn+1` wants slot 1, and `kn+m` wants slot m.

![](/images/accidental-quadratic-hashmap-iteration/image2.png)

Those slots are already full. The front half of `two` filled up while processing `one`'s first half, and now the second half's elements all want to land in that same crowded region. Collisions spike, and linear probing has to walk the array looking for an empty slot, stretching probe length out to O(n). This extreme clustering keeps going until `two` crosses its load factor and triggers a resize.

Measure the actual latency and you get a graph that matches this story exactly: fast through the first half, a sharp cliff once the second half starts, and a return to speed only after `two` resizes.

![](/images/accidental-quadratic-hashmap-iteration/image3.png)

## Reproducing It Myself

I [built my own open-addressing hash table and ran the experiment](https://github.com/bluuewhale/HashSmith/blob/8ff3a288b547eab6813e8a509f94090e005910b5/src/test/java/io/github/bluuewhale/hashsmith/MapSmokeTest.java#L131), and the same behavior showed up. As the load factor climbs toward 0.9, reinsert time explodes.

| entry size | load factor | insert | reinsert |
|---|---|---|---|
| 3,000,000 | 0.5 | 677ms | 253ms |
| 3,000,000 | 0.75 | 426ms | 1,189ms |
| 3,000,000 | 0.9 | 418ms | 493,540ms |

## The Fix

The root cause comes down to one thing: two different tables end up with nearly identical hash ordering, meaning `hash(k) % 2n` and `hash(k) % n` agree most of the time. So the fix is just as direct: make sure each table scrambles its hash ordering differently.

Rust fixed this by [randomizing the seed used for hashing, per table instance](https://github.com/rust-lang/rust/pull/37470). Give each table a different seed, and there's no longer any reason for `one` and `two`'s hash orderings to line up.

There's another angle: scramble the iteration order itself. Using the properties of modular arithmetic over coprime numbers (a [Linear Congruential Generator, or LCG](https://en.wikipedia.org/wiki/Linear_congruential_generator)), you can visit every slot in a 2ⁿ-sized array exactly once, in an order that looks random, without allocating any extra memory. It's a trick that shows up often when you want to add randomness to iteration or probing in an open-addressing table.

```java
private final class EntryIterator implements Iterator<Map.Entry<K, V>> {
	private final int start;
	private final int step;
	private final int mask = capacity - 1;
	private int iter = 0;
	private int nextIdx = -1;

	EntryIterator() {
		ThreadLocalRandom r = ThreadLocalRandom.current();
		// With power-of-two capacity, an odd step is coprime to capacity, so (start + i*step) & mask
		// walks every slot exactly once (full cycle) without extra allocations.
		this.start = r.nextInt() & mask;
		this.step = r.nextInt() | 1; // odd step yields full cycle (capacity is power of two)
		advance();
	}

	private void advance() {
		nextIdx = -1;
		while (iter < capacity) {
			// & mask == mod capacity; iter grows monotonically, step scrambles the visit order.
			int idx = (start + (iter++ * step)) & mask;
			if (arr[idx] != null) {
				nextIdx = idx;
				return;
		}
	}
}
```

## References
- [Rust hash iteration+reinsertion](https://accidentallyquadratic.tumblr.com/post/153545455987/rust-hash-iteration-reinsertion)
- [Exposure of HashMap iteration order allows for O(n²) blowup. · Issue #36481 · rust-lang/rust](https://github.com/rust-lang/rust/issues/36481)
- [Don't reuse RandomState seeds by arthurprs · Pull Request #37470 · rust-lang/rust](https://github.com/rust-lang/rust/pull/37470)
