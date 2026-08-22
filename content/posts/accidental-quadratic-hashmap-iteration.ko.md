+++
title = 'Accidental Quadratic Hashmap Iteration'
date = '2025-12-08T22:53:14+09:00'
draft = false
translationKey = 'accidental-quadratic-hashmap-iteration'
slug = 'accidental-quadratic-hashmap-iteration'
description = '해시 순서(hash ordering)가 겹칠 때, 오픈 어드레싱 해시 테이블을 순회하며 다른 테이블에 재삽입하는 흔한 코드가 왜 O(n^2)으로 폭발하는지 살펴봅니다.'
tags = ['Data Structure', 'Hash Table', 'Rust', 'Performance']
categories = ['Data Structures', 'Performance']

[cover]
image = 'images/accidental-quadratic-hashmap-iteration/image3.png'
hiddenInSingle = true
+++

Rust의 `HashMap`에서 발생했던 흥미로운 버그를 하나 소개하려고 합니다. 겉보기에는 지극히 평범한 코드인데, 특정 조건이 갖춰지면 O(n)이어야 할 연산이 O(n²)으로 폭발합니다.

## 버그 재현

다음 코드를 보겠습니다. 첫 번째 해시맵(`one`)에 1부터 5,000,000까지의 값을 삽입한 뒤(T1), 그 해시맵을 순회하면서 두 번째 해시맵(`two`)에 값을 그대로 재삽입(T2)하는 것이 전부입니다.

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

T1과 T2 모두 원소를 하나씩 삽입하는 동작이니, 둘 다 O(n)이어야 할 것 같습니다. 그런데 실제로 실행해보면 T1은 예상대로 O(n)에 끝나지만, T2는 O(n²)에 가까운 시간이 걸립니다. 같은 일을 하는 코드인데 왜 이런 차이가 발생할까요?

## 버그가 발생하는 조건

이 문제는 아무 해시 테이블에서나 일어나지 않습니다. 다음 조건이 함께 성립할 때 나타납니다.

- 오픈 어드레싱 방식의 해시 테이블
- 선형 탐사(linear probing)로 충돌을 해소
- 내부 버킷 배열을 앞에서부터 순서대로 순회
- 테이블 크기가 2의 거듭제곱
- 버킷 위치를 `hash & (capacity - 1)`로 계산
- 로드 팩터가 높은 상태(예: 0.9)

문제는 이 조건들이 오픈 어드레싱 해시 테이블 구현체 대부분에서 자연스럽게 만족된다는 점입니다. 즉, 특별히 이상한 구현이 아니라 흔하게 쓰이는 방식 그 자체가 이 버그를 유발합니다.

## 왜 이런 일이 벌어질까

T1이 끝난 시점, 첫 번째 해시맵 `one`의 크기는 2n이라고 하겠습니다. T2를 진행하는 도중, 두 번째 해시맵 `two`의 크기가 아직 n인 상태를 생각해 보겠습니다.

T2는 `one`의 버킷 배열을 앞에서부터 순서대로 훑으면서 원소를 하나씩 꺼내 `two`에 넣습니다. `one`에서 0번째 자리에 있는 원소 `k0`을 꺼냈다고 하면, 이는 `hash(k0) % 2n = 0`일 확률이 높다는 뜻입니다. 그런데 이 `k0`을 `two`에 넣을 위치는 `hash(k0) % n`으로 계산되고, 이 값 역시 0일 확률이 매우 높습니다. `hash(k0) % 2n`과 `hash(k0) % n`이 우연히 같은 값을 낼 확률이 상당히 크기 때문입니다.

이 관찰을 일반화하면, `one`의 앞쪽 절반(인덱스 0부터 n-1까지)을 순회하는 동안은 `hash(k) % n`과 `hash(k) % 2n`이 거의 같은 값을 냅니다. 그러니 `k0`은 `two`의 0번째에, `k1`은 1번째에 들어가는 식으로, `two`는 앞에서부터 순서대로 착실히 채워집니다.

![](/images/accidental-quadratic-hashmap-iteration/image1.png)

진짜 문제는 나머지 절반(인덱스 n부터 2n-1까지)을 순회할 때 시작됩니다. `one`의 n번째 자리에 있는 원소 `kn`은 `hash(kn) % 2n = n`일 확률이 높은데, 이걸 `two`에 넣을 위치를 계산하면 `hash(kn) % n = 0`이 되어버립니다. `hash(kn) % 2n`이 n이라면, 그 값에서 n을 뺀 나머지인 `hash(kn) % n`은 정확히 0이 되기 때문입니다. 같은 논리로 `kn+1`은 `two`의 1번째에, `kn+m`은 m번째에 들어가려고 합니다.

![](/images/accidental-quadratic-hashmap-iteration/image2.png)

문제는 그 자리들이 이미 앞 절반을 처리하면서 꽉 채워졌다는 것입니다. `two`의 앞부분은 이미 가득 찬 상태에서, 뒤 절반의 원소들도 하필 그 앞부분에 배정되려고 몰려드는 셈입니다. 충돌이 폭발적으로 늘어나고, 선형 탐사는 빈 자리를 찾을 때까지 배열을 쭉 훑어야 하니 탐사 거리가 O(n)까지 늘어납니다. 이 극단적인 클러스터링은 `two`가 로드 팩터를 넘어서 리사이징이 일어날 때까지 계속됩니다.

실제로 지연 시간을 측정해보면 이 설명과 정확히 들어맞는 그래프가 나옵니다. 앞쪽 절반을 처리할 때는 빠르게 채워지다가, 뒤쪽 절반으로 넘어가는 순간 급격히 느려지고, 로드 팩터를 넘어 리사이징이 일어난 뒤에야 다시 빨라집니다.

![](/images/accidental-quadratic-hashmap-iteration/image3.png)

## 직접 재현해보기

[직접 오픈 어드레싱 해시 테이블을 구현해서 실험해본 결과](https://github.com/bluuewhale/HashSmith/blob/8ff3a288b547eab6813e8a509f94090e005910b5/src/test/java/io/github/bluuewhale/hashsmith/MapSmokeTest.java#L131), 같은 현상을 재현할 수 있었습니다. 로드 팩터가 0.9에 가까워질수록 재삽입(reinsert) 시간이 폭발적으로 늘어나는 것을 확인할 수 있습니다.

| entry size | load factor | insert | reinsert |
|---|---|---|---|
| 3,000,000 | 0.5 | 677ms | 253ms |
| 3,000,000 | 0.75 | 426ms | 1,189ms |
| 3,000,000 | 0.9 | 418ms | 493,540ms |

## 해결 방법

근본 원인은 결국 하나입니다. 서로 다른 두 테이블의 해시 순서(hash ordering)가 거의 동일하다는 것, 즉 `hash(k) % 2n`과 `hash(k) % n`이 대부분 같은 값을 낸다는 것입니다. 그러니 해법도 명확합니다. 테이블마다 해시 순서가 서로 다르게 뒤섞이도록 만들면 됩니다.

Rust는 이 문제를 [해시 생성에 쓰이는 seed를 테이블 인스턴스마다 랜덤하게 만드는 방식](https://github.com/rust-lang/rust/pull/37470)으로 해결했습니다. 테이블마다 seed가 다르면, `one`과 `two`의 해시 순서가 우연히 겹칠 이유가 없어집니다.

또 다른 접근으로는, 순회(iteration) 자체의 순서를 뒤섞는 방법이 있습니다. 서로소 관계에 있는 두 수를 이용한 모듈러 산술의 성질([LCG, Linear Congruential Generator](https://en.wikipedia.org/wiki/Linear_congruential_generator))을 활용하면, 추가 메모리 없이도 2ⁿ 크기의 배열을 마치 무작위인 것처럼 한 번씩 빠짐없이 훑을 수 있습니다. 오픈 어드레싱 테이블에서 순회나 프로빙에 무작위성을 섞고 싶을 때 종종 쓰이는 트릭입니다.

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
