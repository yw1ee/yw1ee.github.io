---
title: 메모리 동시성
description: Atomic
author: yw1ee
categories: [CS, concurrency]
tags: [Rust, C++20, concurrency, atomic, ordering]
---

## 메모리 동시성 (Concurrency)

Multi-thread 환경에서는 여러 thread가 같은 변수(메모리)를 바라보고 공유하는 경우가 많다. 이런 경우 흔하게 mutex나 atomic을 사용하여 동시성 프로그래밍을 하게 된다. Atomic의 경우 memory ordering이라는 개념이 함께 사용되는데, Atomic과 memory ordering의 정확한 개념을 알아보자.

## Atomic

먼저 atomic이란 여러 연산을 쪼개질 수 없는 하나의 동작으로 수행하는 개념이다. 연산이 쪼개진다는 것은 이런 것이다.
가령 아래와 같은 프로그램을 짜면 `x += 1` 이 한 줄이 (아무 최적화가 없다면) 3개의 어셈블리 명령어로 쪼개지게 될 것이다.
```rust
let mut x = 0;
x += 1;
// 1. `x`의 주소로부터 값을 load
// 2. load한 값에 1을 더함
// 3. 다시 `x`의 주소에 더한 값을 store
```
만약 여러 thread가 `x`라는 변수를 공유하고, 동시에 변수 `x`에 더하기를 실행한다면 data race가 발생할 수 있다.
`x`가 0으로 시작한 상황에서 thread A, B가 각자 1을 더하고 저장하는 코드를 실행하면 아래와 같이 실행될 수 있기 때문이다.

|A | x_A |  B | x_B | x |
|--+-----+----+-----+---|
|load | 0 | _ | _ | 0 |
|--+-----+----+-----+---|
|add | 1 | load | 0 | 0 |
|--+-----+----+-----+---|
|store | 1 | add | 1 | 1 (x_A로부터)|
|--+-----+----+-----+---|
|_ | 1 | store | 1 | 1 (x_B로부터)|

Atomic 연산은 위의 3가지 과정을 쪼개질 수 없는 하나의 동작으로 수행한다. 따라서 thread A가 먼저 작업을 시작했으면 thread B가 값을 load하기 전에 thread A의 store작업까지 모두 끝마쳐짐을 보장할 수 있고, data race를 방지할 수 있다. `AtomicU64` type의 `fetch_add` 함수는 atomic하게 `load - add - store`를 수행하고, 기존 값을 내뱉는다.
```rust
let x = AtomicU64::new(0);
let y = x.fetch_add(1, Ordering::Relaxed); // y == 0
let z = x.load(Ordering::Relaxed); // z == 1
```
그런데 `fetch_add` 함수와 `load` 함수는 `Ordering` 이라는 추가적인 인자를 하나 더 필요로 한다. 이 인자가 뜻하는 의미는 무엇일까?

## Redordering

Memory Ordering을 설명하기 위해서는 현대의 컴파일러와 아키텍쳐의 최적화에 대해 조금 이해할 필요가 있다.
기억해야 할 사실은 프로그램이 내가 작성대로 실행되지 않을 수 있다는 것이다.

### Compiler & Hardware reordering
[참고한 문서](https://sabrinajewson.org/rust-nomicon/atomics/atomics.html)의 예시를 통해 알아보자.

예를 들어 이렇게 작성한 코드는 컴파일러에 의해 최적화되어 line 1이 사라질 수 있다. 또한 어떠한 최적화 때문에 line 2와 line 3의 실행 순서도 뒤바뀔 수 있다.
```
x = 1; (이 한 줄은 사라질 수 있다)
y = 2;
x = 3;
```

그럼 컴파일러가 생성한 어셈블리 코드는 온전히 그대로 실행된다고 볼 수 있을까? 그렇지 않다. 이번에는 CPU 아키텍쳐에 의해 순서가 뒤바뀌거나, 전혀 엉뚱한 동작을 하는 것 처럼 보일 수 있다. 근본적인 원인은 CPU core cache 때문인데 자세히는 다루지 않고 넘어가도록 하겠다.
```
initial state: x = 0, y = 1

thread A
- y = 3;
- x = 1;

thread B
- if x == 1 { y *= 2 }
```
이 실행의 최종 결과로 가능한 시나리오는 다음과 같다.
- x = 1, y = 6 (thread A가 thread B 시작 전에 끝난 경우)
- x = 1, y = 3 (thread B가 thread A 시작 전에 끝난 경우)
- x = 1, y = 2 (????)

마지막 시나리오가 굉장히 수상하다. thread B가 x = 1임을 보았지만, 그 순간 y = 1임을 보는 상황이고, thread A에서의 y = 3을 덮어쓰고 2를 기록하는 상황이다. 이상하지만 이 또한 가능한 시나리오 중 하나이다.

따라서 우리는 내가 작성한 코드가 컴파일러와 아키텍쳐에 의해 순서가 뒤바뀌어 실행될 수 있음을 기억해야 한다. 그렇다면 이렇게 컴파일러와 하드웨어에 의해서 내 의도대로 동작하지 않게 된다면, 무엇을 믿고 코드를 작성할 수 있을까? 어떻게 나의 의도대로 동작하게 수정할 수 있을까?

### Happens-before (sequenced-before)

이를 위해선 happens-before/sequenced-before 라는 개념이 필요하다. 이 개념을 이해하기 위해 우리는 "시간" 개념을 지워야 한다. 하나의 thread 안에서 "시간 순서대로 흘러간다" 라는 개념도 없을 뿐더러, 모든 thread가 시간을 공유하고 동기화되는 개념을 지워야한다 (이 부분이 나에게는 아주 혼란스러웠다). 대신 happens-before/sequenced-before 라는 개념으로 메모리에 대한 순서를 정의할 수 있다.

아래에서 `x`는 atomic 변수이다.
```
initial state: x = 0

thread A:
- x.store(1)
- x_A = x.load()


thread B:
- x_B = x.load()
- x.store(x_B + 1)
```

Sequenced-before 관계는 단일 thread 내에서 연산들의 순서를 나타내기 위한 개념이다. 이는 언어의 규칙으로 정의되며 코드에 명시된 순서대로 실행될 것이라는 우리의 직관은 이 관계를 배경으로 한다.
이제 thread A만 먼저 살펴보자. `x`에 대해 1을 `store`한 후에 `load`를 수행하는데, 코드에 작성된 대로 `store`와 `load`는 sequenced-before 관계가 성립한다. `load`는 `store` 이후에 수행되는데, `load`한 값을 `store`가 항상 볼 수 있다. 만약 이러한 sequenced-before 관계가 정의되지 않았다면, `x_A`의 값이 1임이 보장될 수 없다 (thread B 제외). 이렇게 한 thread 내에서, 우리가 코드에 기술한 순서대로의 실행은 sequenced-before 관계를 통해 보장될 수 있다.

> 엄밀히 말하자면 `store`와 `load`의 sequenced-before 관계는 ordering에 의해 결정될 요소이지만 설명의 편의를 위해 성립한다고 가정했다
{: .prompt-info }

그럼 이번엔 thread B도 함께 살펴보자. Thread B안에서의 `load`와 `store` 또한 sequenced-before 관계가 성립한다. 하지만 thread A와 thread B 사이에는 sequenced-before 관계가 성립될 수 없다. 여기서 thread 간의 메모리 순서를 보장할 수 있는 방법이 happens-before 관계이다.
만약 우리가 어떠한 방법으로 thread A의 `store` 이후에 thread B의 `load`가 실행됨을 나타낼 수 있다면 `x`의 최종 결과 값은 2임이 보장될 것이다. 또한 thread B의 `store` 이후에 thread A의 `load`가 실행됨을 나타낼 수 있다면 `x_A`의 값도 2임을 보장할 수 있을 것이다. 그렇다면 이렇게 thread 간의 load/store 동작의 순서를 어떻게 나타낼 수 있을까? 위에서 보았던 `Ordering::Relaxed` 같은 memory ordering을 통해 나타낼 수 있다.

## Memory Ordering

Rust의 memory ordering에는 총 4가지 종류가 있다. 이는 [C++20의 specification](https://en.cppreference.com/w/cpp/atomic/memory_order)에서 (consume을 제외하고) 따온 것이라고 한다.

### Relaxed

먼저 Relaxed ordering은 아무 동기화도 제공하지 않고 오직 원자적 operation만 지원한다. 아래의 프로그램을 실행시키면 최종 값은 항상 10000임이 보장된다.
```rust
use std::sync::atomic::Ordering;
use std::sync::{atomic::AtomicUsize, Arc};

fn main() {
    let x = Arc::new(AtomicUsize::new(0));

    let mut threads = vec![];
    for _ in 0..100 {
        let x_ = x.clone();
        threads.push(std::thread::spawn(move || {
            for _ in 0..100 {
                x_.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for t in threads {
        t.join().unwrap();
    }

    assert_eq!(100 * 100, x.load(Ordering::Relaxed));
}
```

Relaxed ordering은 아무 동기화도 제공하지 않기 때문에 단일 thread내에서 최종 결과가 같다면 컴파일러/아키텍쳐가 마음대로 순서를 바꿔서 실행할 수 있는 자유도를 갖고, 성능상 가장 유리하다. 하지만 이러한 특성 때문에 발생하는 Out-of-the-air 문제를 살펴보자.
```rust
// thread A
let r1 = y.load(Ordering::Relaxed); // 1
x.store(r1, Ordering::Relaxed);     // 2
// thread B
let r2 = x.load(Ordering::Relaxed); // 3
y.store(42, Ordering::Relaxed);     // 4
```
`x`와 `y`의 초기값이 0일 때, 이 프로그램을 실행시키면 `r1`과 `r2`의 값으로 가능한 조합은 어떤게 있을까? 아마도 (0, 0), (0, 42), (42, 0) 이 가능할 것이다.

혹시 모두 42가 나오는 결과를 관찰할 수 있을까? 이론적으로 모든 operation이 Relaxed이고, CPU가 4번과 3번의 순서를 뒤바꿔 실행해도 최종 결과는 같을 것이라고 판단한다면, 4 -> 1 -> 2 -> 3 의 순서로 실행되어 두 값 모두 42가 나올 수 있다.
따라서 Relaxed ordering은 아무 동기화도 필요 없는 상황에 사용하는 것이 좋다.

### Release / Acquire

Release와 Acquire은 서로 쌍으로 사용된다. Thread A에서 Release store를 수행하면 되면 해당 store보다 이 전에 발생한 모든 (non-atomic과 relaxed를 포함한) write operation 들은 reordering 되더라도 이 전에 위치함을 보장한다. 따라서 thread B에서 Acquire load를 수행하면 thread A가 Release store 이 전에 수행한 모든 memory write를 볼 수 있음이 보장된다. x86과 같은 strongly-ordered system에서는 기본적으로 Release/Acquire ordering이 적용된다. Release/Acquire ordering은 Mutex나 atomic spinlock에서 사용될 수 있다.

Relaxed와 함께 사용되는 간단한 예시를 살펴보자.
```
initial state: a = 0, b = 0, c = 0, state = false

thead A
- a.store(1, Relaxed)
- b.store(2, Relaxed)
- c.store(3, Relaxed)
- state.store(true, Release)

thread B
- while !state.load(Acquire) {}
- assert(c.load(Relaxed), 3)
- assert(b.load(Relaxed), 2)
- assert(a.load(Relaxed), 1)
```

먼저 thread A의 동작을 살펴보자. 변수 a, b, c 에 값을 store하는 operation들은 모두 Relaxed이므로 세 operation 간에는 순서가 뒤바뀔 수 있다. 하지만 state.store는 Release ordering이기 때문에 state.store 보다 뒤에 실행될 수는 없다.
이제 thread B의 동작을 보자. state에 대한 Acquire load를 수행했으므로, thread A에서 state.store 보다 happens-before 관계에 있는 모든 memory operation을 볼 수 있다. 따라서 thread A가 기록한 a, b, c의 값을 볼 수 있게 된다.

이번에는 아래와 같이 Mutex를 구현해보자. 디테일은 많이 생략했으므로, pseudo code정도로 보면 된다.
```rust
impl<T> Mutex<T> {
    pub fn lock(&self) -> Option<Guard<T>> {
        match self.locked.compare_exchange(
            false,
            true,
            atomic::Ordering::Relaxed,
            atomic::Ordering::Relaxed,
        ) {
            Ok(_) => Some(Guard(self)),
            Err(_) => None,
        }
    }
}

impl<T> Drop for Guard<T> {
    fn drop(&mut self) {
        self.0.locked.store(false, atomic::Ordering::Relaxed);
    }
}

// initial state
let mutex = Mutex::new(0);
// thread A
if let Some(guard) = mutex.lock() {
    *guard += 1;
}
// thread B
if let Some(guard) = mutex.lock() {
    println!("{}", *guard);
}
```

만약 thread A가 먼저 실행되고, thread B가 나중에 실행된다면 어떤 값이 출력될까? 답은 0과 1 모두 가능하다. Thread A가 실행될 때 4가지 memory operation이 일어나는데 locked에 대한 operation이 Relaxed이므로, 순서가 뒤바뀌어 실행될 수 있다.
1. locked.compare_exchange(false, true)
2. load guard
3. store guard
4. locked.store(false)

Thread B가 관찰하기에 3번과 4번이 순서가 뒤바뀌어 실행되었다면 0을 관찰할 수 있게 된다. 하지만 아래와 같이 Release/Acquire ordering을 사용하게 된다면, 4번 이 전에 3번이 수행되는 것이 thread B에도 동기화 되기 때문에 1을 출력한다.

```rust
impl<T> Mutex<T> {
    pub fn lock(&self) -> Option<Guard<T>> {
        match self.locked.compare_exchange(
            false,
            true,
            atomic::Ordering::Acquire,
            atomic::Ordering::Relaxed,
        ) {
            Ok(_) => Some(Guard(self)),
            Err(_) => None,
        }
    }
}

impl<T> Drop for Guard<T> {
    fn drop(&mut self) {
        self.0.locked.store(false, atomic::Ordering::Release);
    }
}
```

### SeqCst

Sequentially-Consistent의 약자로, 어떠한 reordering도 허용하지 않고 코드에 작성된 그대로 memory operation을 수행하는 방법이다. Multiple producer-multiple consumer 환경에서 유용하게 사용된다.

간단한 예시로 위의 Out-of-the-air 문제를 다시 가져와보자.

```rust
// thread A
let r1 = y.load(Ordering::SeqCst); // 1
x.store(r1, Ordering::SeqCst);     // 2
// thread B
let r2 = x.load(Ordering::SeqCst); // 3
y.store(42, Ordering::SeqCst);     // 4
```

이 경우에는 SeqCst ordering을 사용했으므로, reordering이 불가능하여 `r1`과 `r2`가 모두 42가 나올 수 없다. 

## 정리

현대의 컴파일러와 아키텍쳐는 상당한 수준의 최적화를 진행하며 memory reordering도 수행한다. 이 때 올바른 메모리 동기화 메커니즘을 제공할 수 있는 방법이 Memory Ordering이다. 이를 잘못 사용할 경우 프로그램이 undefined behavior를 유발하거나, 불필요한 성능적 overhead를 감수해야 한다. 다음 글에서는 이러한 이해를 바탕으로 lockfree data structure를 직접 구현해보며 이에 대해 알아보자.

## 참조
- [https://sabrinajewson.org/rust-nomicon/atomics/atomics.html](https://sabrinajewson.org/rust-nomicon/atomics/atomics.html)
- [https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html)
- [https://en.cppreference.com/w/cpp/atomic/memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order)
- [https://timsong-cpp.github.io/cppwp/n4868/intro.multithread#intro.races](https://timsong-cpp.github.io/cppwp/n4868/intro.multithread#intro.races)