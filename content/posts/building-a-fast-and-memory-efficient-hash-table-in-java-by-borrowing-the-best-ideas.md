+++
date = '2025-12-12T22:56:51+09:00'
draft = false
title = 'Building a Fast, Memory-Efficient Hash Table in Java (by borrowing the best ideas)'
+++

One day, I ran into SwissTable—the kind of design that makes you squint, grin, and immediately regret every naive linear-probing table you've ever shipped.

This post is the story of how I tried to bring that same "why is this so fast?" feeling into Java. It's part deep dive, part engineering diary, and part cautionary tale about performance work.  

## 1) The SwissTable project, explained the way it *feels* when you first understand it

SwissTable is an open-addressing hash table design that came out of Google's work and was famously presented as a new C++ hash table approach (and later shipped in Abseil). 

At a high level, it still does the usual hash-table thing: compute `hash(key)`, pick a starting slot, and probe until you find your key or an empty slot. 

The twist is that SwissTable separates *metadata* (tiny "control bytes") from the actual key/value storage, and it uses those control bytes to avoid expensive key comparisons most of the time. Instead of immediately touching a bunch of keys (which are cold in cache and often pointer-heavy), it first scans a compact control bytes that is dense, cache-friendly, and easy to compare in bulk.

To make probing cheap, SwissTable effectively splits the hash into two parts: `h1` and `h2`. Think of `h1` as the part that chooses where to start probing (which group to look at first), and `h2` as a tiny fingerprint stored in the control bytes to quickly rule slots in or out. It's not a full hash—just enough bits to filter candidates before we pay the cost of touching real keys.

So on lookup, you compute a hash, derive `(h1, h2)`, jump to the group from `h1`, and compare `h2` against all control bytes in that group before you even look at any keys. That means most misses (and many hits) avoid touching key memory entirely until the metadata says "there's a plausible candidate here."  

Because probing stays cheap, SwissTable can tolerate higher load factors—up to about 87.5% (7/8) in implementations like Abseil's `flat_hash_map`—without falling off a performance cliff, which directly improves memory efficiency.  
The net effect is a design that is simultaneously faster (fewer cache misses, fewer key compares) and tighter (higher load factor, fewer side structures like overflow buckets).

## 2) Watching SwissTable become the "default vibe" in multiple languages (Go, Rust)

The first sign you're looking at a generational design is when it stops being a cool library trick and starts showing up in standard libraries.

Starting in Rust 1.36.0, `std::collections::HashMap` switched to the SwissTable-based [hashbrown](https://github.com/rust-lang/hashbrown) implementation. It's described as using **quadratic probing and SIMD lookup**, which is basically SwissTable territory in spirit and technique. That was my "okay, this isn't niche" moment.

Then Go joined the party: Go 1.24 ships a new built-in map implementation based on the Swiss Table design, straight from the [Go team's own blog post](https://go.dev/blog/swisstable). In their microbenchmarks, map operations are reported to be **up to 60% faster** than Go 1.23, and in full application benchmarks they saw about a **1.5% geometric-mean CPU time improvement**. And if you want a very practical "this matters in real systems" story, [Datadog](https://www.datadoghq.com/blog/engineering/go-swiss-tables/) wrote about Go 1.24's SwissTable-based maps and how the new layout and growth strategy can translate into serious memory improvements at scale.

At that point, SwissTable stopped feeling like "a clever C++ trick" and started feeling like *the modern baseline*. I couldn't shake the thought: Rust did it, Go shipped it… so why not Java? And with modern CPUs, a strong JIT, and the Vector API finally within reach, it felt less like a technical impossibility and more like an itch I had to scratch.  

That's how I fell into the rabbit hole.

## 3) SwissTable's secret sauce meets the Java Vector API

A big part of SwissTable's speed comes from doing comparisons *wide*: checking many control bytes in one go instead of looping byte-by-byte and branching constantly.  That's exactly the kind of workload SIMD is great at: load a small block, compare against a broadcasted value, get a bitmask of matches, and only then branch into "slow path" key comparisons. In other words, SwissTable is not just "open addressing done well"—it's "open addressing shaped to fit modern CPUs."  

Historically, doing this portably in Java was awkward: you either trusted auto-vectorization, used `Unsafe`, wrote JNI, or accepted the scalar loop. But the **Vector API** has been incubating specifically to let Java express vector computations that reliably compile down to good SIMD instructions on supported CPUs.

In Java 25, the Vector API is still incubating and lives in `jdk.incubator.vector`. The important part for me wasn't "is it final?"—it was "is it usable enough to express the SwissTable control-byte scan cleanly?" Because if I can write "compare 16 bytes, produce a mask, act on set bits" in plain Java, the rest of SwissTable becomes mostly careful data layout and resizing logic. And once you see the control-byte scan as *the* hot path, you start designing everything else to make that scan cheap and predictable.  

So yes: the Vector API was the permission slip I needed to try something I'd normally dismiss as "too low-level for Java."

## 4) So I started implementing it (and immediately learned what Java makes easy vs. hard)

I began with the core SwissTable separation: a compact **control array** plus separate **key/value storage**. The control bytes are the main character—if those stay hot in cache and the scan stays branch-light, the table feels fast even before micro-optimizations.  

I used the familiar `h1/h2` split idea: `h1` selects the initial group, while `h2` is the small fingerprint stored in the control byte to filter candidates. Lookup became a two-stage pipeline: (1) vector-scan the control bytes for `h2` matches, (2) for each match, compare the actual key to confirm. Insertion reused the same scan, but with an extra "find first empty slot" path once we know the key doesn't already exist.

Where Java started pushing back was *layout realism*. 

In C++ you can pack keys/values tightly; in Java, object references mean the "key array" is still an array of pointers, and touching keys can still be a cache-miss parade. So the design goal became: **touch keys as late as possible**, and when you must touch them, touch as few as possible—again, the SwissTable worldview.  

Deletion required tombstones (a "deleted but not empty" marker) so probing doesn't break, but tombstones also accumulate and can quietly degrade performance if you never clean them up.  

Resizing was its own mini-project: doing a full rehash is expensive, but clever growth strategies (like Go's use of table splitting/extendible hashing) show how far you can take this if you're willing to complicate the design.

I also had to treat the Vector API as an optimization tool, not a magic wand. Vector code is sensitive to how you load bytes, how you handle tails, and whether the JIT can keep the loop structure stable. I ended up writing the control-byte scan as a very explicit "`load` → `compare` → `mask` → `iterate matches`" loop.

At this stage, the prototype already *worked*, but it wasn't yet "SwissTable fast"—it was "promising, and now the real work begins."

## 5) The pieces of SwissMap that actually mattered

Here's what survived after the usual round of "this feels clever but isn't fast" refactors:

- **Control-plane first**: allocate control bytes plus sentinel padding so SIMD loads never run past the end. Keys/values stay separate (and cold) until metadata says otherwise.
- **h1/h2 split**: the high bits pick the starting group; the low 7 bits become the control-byte fingerprint. Empty/deleted/full states live in the control bytes as values—not as scattered flags.
- **Vectorized probe loop**: load one group of control bytes, compare against a broadcast `h2`, turn the vector comparison into a bitmask, and only then touch candidate keys. The same loaded vector is reused to detect empties (and optionally tombstones) without extra passes.

```java
protected int findIndex(Object key) {
    if (size == 0) return -1;
    int h = hash(key);
    int h1 = h1(h);
    byte h2 = h2(h);
    int nGroups = numGroups();
    int visitedGroups = 0;
    int mask = nGroups - 1;
    int g = h1 & mask; // optimized modulo operation (same as h1 % nGroups)
    for (;;) {
        int base = g * DEFAULT_GROUP_SIZE;
        ByteVector v = loadCtrlVector(base);
        long eqMask = v.eq(h2).toLong();
        while (eqMask != 0) {
            int bit = Long.numberOfTrailingZeros(eqMask);
            int idx = base + bit;
            if (Objects.equals(keys[idx], key)) { // found
                return idx;
            }
            eqMask &= eqMask - 1; // clear LSB
        }
        long emptyMask = v.eq(EMPTY).toLong(); // reuse loaded vector
        if (emptyMask != 0) { // empty slot found. no need to probe further.
            return -1;
        }
        if (++visitedGroups >= nGroups) { // guard against infinite probe when table is full of tombstones
            return -1;
        }
        g = (g + 1) & mask;
    }
}
```

- **Tombstone accounting**: deletions mark `DELETED` and increment a tombstone counter; the table triggers rehash when load or tombstones cross a threshold to keep probe lengths stable.
- **Minimal rehash path**: rehashing repopulates into a fresh control/key/value triad using the same insert path, but without duplicate checks.
- **Iterators without extra buffers**: a precomputed pseudorandom stride walks the table without auxiliary arrays, while avoiding [accidental quadratic behavior during iteration](https://accidentallyquadratic.tumblr.com/post/153545455987/rust-hash-iteration-reinsertion)

```java
private void advance() {
    next = -1; 
    while (iter < capacity) {
        // LCG - see https://en.wikipedia.org/wiki/Linear_congruential_generator
        int idx = (start + (iter++ * step)) & mask;
        if (isFull(ctrl[idx])) { 
            next = idx; 
            return; 
        } 
    }
}
```

These were the hotspots the JIT could keep tight; everything else (`equals`/`hashCode` plumbing) stayed intentionally boring.

## 6) Benchmarks

I didn't want to cherry-pick numbers that only look good on synthetic cases.  
So I used a simple, repeatable JMH setup that stresses high-load probing and pointer-heavy keys—the exact situation SwissTable-style designs are meant to handle.  

All benchmarks were run on Windows 11 (x64) with Eclipse Temurin JDK 21.0.9, on an AMD Ryzen 5 5600 (6C/12T).

For context, I compared against `HashMap`, fastutil's `Object2ObjectOpenHashMap`, and Eclipse Collections' `UnifiedMap`.

### The headline result
Even near the maximum load factor, SwissMap stays competitive with other open-addressing tables and remains close to `HashMap` in throughput.  

| get hit | get miss |
| --- | --- |
| ![CPU: get hit](/images/hash-smith/map-cpu-get-hit.png) | ![CPU: get miss](/images/hash-smith/map-cpu-get-miss.png) |

| put hit                                     | put miss                                      |
|---------------------------------------------|-----------------------------------------------|
| ![CPU: put hit](/images/hash-smith/map-cpu-put-hit.png) | ![CPU: put miss](/images/hash-smith/map-cpu-put-miss.png) |

On memory, the flat layout (no buckets/overflow nodes) plus a 0.875 (7/8) max load factor translated to a noticeably smaller retained heap in small-payload scenarios—over 50% less than `HashMap` in this project's measurements.

![Memory Footprint](/images/hash-smith/map-memory-bool.png)

### Caveat
These numbers are pre-release; the Vector API is still incubating, and the table is tuned for high-load, reference-key workloads. Expect different results with primitive-specialized maps or low-load-factor configurations.

## P.S. If you want the code

This post is basically the narrative version of an experiment I'm building in public: [**HashSmith**](https://github.com/bluuewhale/hash-smith), a small collection of fast, memory-efficient hash tables for the JVM.  

It includes `SwissMap` (SwissTable-inspired, SIMD-assisted probing via the incubating Vector API), plus a `SwissSet` variant to compare trade-offs side by side.

It's explicitly **experimental** (not production-ready), but it comes with JMH benchmarks and docs so you can reproduce the numbers and poke at the implementation details.

If you want to run the benchmarks, sanity-check an edge case, or suggest a better probe/rehash strategy, I'd love issues/PRs