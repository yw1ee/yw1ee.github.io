---
title: Rust의 impl Trait과 dyn Trait
description: impl vs dyn
author: yw1ee
categories: [Rust]
tags: [Rust, dyn, impl, trait]
---

## Rust에서 generic type을 나타내는 두가지 방법

Rust에는 구체적인 type 대신 "어떤 trait을 구현한 타입"을 표현하기 위해 `impl Trait`과 `dyn Trait`을 사용할 수 있다. 큰 그림에서는 두 방법이 비슷해 보이지만 명확한 차이점과 장단점이 있다.

## `impl Trait`이란

`impl Trait`은 unnamed이지만 특정 trait을 구현한 구체적인 타입을 명시하기 위해 사용된다. `impl Trait`은 함수의 인자 혹은 함수의 리턴 타입에 사용될 수 있다.

`impl Trait`이 함수의 인자 타입으로 사용되는 경우 generic type의 syntactic sugar 정도로 이해하면 된다. 아래의 두 함수는 거의 동일하다. 사용자는 `MyTrait`을 구현한 어떠한 타입이든 함수의 인자로 넘겨줄 수 있다.

```rust
trait MyTrait {};

fn with_generic_type<T: MyTrait>(arg: T) {..}

fn with_impl_triat(arg: impl MyTrait) {..}
```

`impl Trait`이 함수의 리턴 타입으로 사용될 때는 unboxed type을 리턴할 수 있게 해준다. closure와 iterator를 나타낼 때 유용하다. 

```rust
fn return_closure_dyn() -> Box<dyn Fn(u8) -> u8> {
    Box::new(|x| x + 1)
}

fn return_closure_impl() -> impl Fn(u8) -> u8 {
    |x| x + 1
}
```

한가지 주의해야 할 점이 있는데 `impl Trait`은 컴파일 타임에 실제 concrete type을 resolve 하기 때문에, 함수로부터 리턴되는 concrete 한 타입은 언제나 같아야 한다. 덕분에 box로 인한 heap allocation을 없앨 수 있고 runtime의 dynamic dispatching을 피할 수 있어 성능상 이점이 있다.

iterator를 나타낼 경우에는 함수의 리턴 값을 trait으로 노출된 함수만 사용할 것이라는 뜻을 내포하고 있으므로, iterator의 복잡한 실제 타입을 기술하지 않아도 되는 장점이 있다.

### generic type과의 비교

위에서 `impl Trait`은 어떻게 보면 generic type의 syntactic sugar로 볼 수 있다고 말했는데, `impl Trait`을 함수의 리턴 타입으로 사용할 경우 약간의 차이가 있다. generic type을 사용하면 함수의 리턴 타입을 caller가 결정할 수 있는 반면 `impl Trait`을 사용하면 함수가 리턴 타입을 결정한다.

```rust
fn return_generic<T: MyTrait>() -> T {..}

fn return_impl() -> impl Mytrait {..}
```

## `dyn Trait`이란

Rust의 `dyn` 키워드는 trait object 타입의 prefix이다. trait object란 trait 집합을 구현한 다른 타입의 opaque value를 뜻한다. 첫번째 trait 외에 다른 trait들은 모두 auto trait이어야 하며 lifetime은 한번만 명시될 수 있다.

```rust
dyn MyTrait
dyn MyTrait + Send + Sync
dyn Mytrait + 'static
```

trait object는 concrete type을 컴파일 타임에 알 수 없고 런타임에만 알 수 있으므로 DST(Dynamically Sized Type)이고, 포인터와 함께 사용되며 (ex, `&dyn Trait`, `Box<dyn Trait>`), vtable 포인터를 참조해 indirect하게 호출(dynamic dispatching)된다.

`impl Trait`과는 다르게 컴파일타임에 concrete type이 결정될 필요가 없으므로 아래와 같이 사용할 수 있다.

```rust
struct A {
    method: Box<dyn Fn(u8) -> u8>
}

let a0 = A { method: Box::new(|x| x + 1) };
let a1 = A { method: Box::new(|x| x + 2) };
```

### object safety

`dyn Trait`의 trait 집합은 object safe trait를 기본으로 auto trait들을 추가하여 정의될 수 있다. 여기서 object safe trait을 만족하려면 아래 조건을 모두 만족해야 한다.
- supertrait들이 모두 object safe trait이어야 함
- `Sized` trait이 supertrait이 아니어야 함
- 연관된 constant가 없어야 함
- 연관된 generic type이 없어야 함
- 모든 연관된 함수들은 trait object로부터 dispatchable하거나 명시적으로 non-dispatchable이어야 함
  - 이와 관련된 조건은 조금 더 복잡한데 많이 어렵지 않으므로 직접 한번 읽어 보는 것을 추천한다


## 결론

- `impl Trait`은 컴파일타임에 concrete type으로 모두 resolve하기 때문에
  - **성능상 유리하지만**
  - 유연성이 부족하고
  - 생성되는 코드의 크기가 커진다.
- `dyn Trait`은 런타임에 dynamic dispatching을 수행하기 때문에
  - 성능상 오버헤드가 존재하지만
  - **유연성이 높고**
  - 생성되는 코드의 크기가 작아진다.



## 참고
- [https://doc.rust-lang.org/reference/types/impl-trait.html](https://doc.rust-lang.org/reference/types/impl-trait.html)
- [https://doc.rust-lang.org/reference/types/trait-object.html](https://doc.rust-lang.org/reference/types/trait-object.html)
- [https://doc.rust-lang.org/std/keyword.dyn.html](https://doc.rust-lang.org/std/keyword.dyn.html)