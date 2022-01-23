---
title: Rust Enum Layout 的优化
date: 2022-01-23
author: TennyZhuang
categories: TennyZhuang
tags:
    - Rust
---

今天学到了一点关于 Rust Enum 的冷知识，在开始阅读之前，大家可以猜一下下面的 Rust 代码在常见的 64 bit 机器上的输出是什么？

```rust
struct A (i64, i8);
struct B (i64, i8, bool);
fn main() {
    dbg!(std::mem::size_of::<A>());
    dbg!(std::mem::size_of::<Option<A>>());
    dbg!(std::mem::size_of::<B>());
    dbg!(std::mem::size_of::<Option<B>>());
}
```

在这个 [Rust Playground][playground1] 里可以看到结果。

[playground1]: https://play.rust-lang.org/?version=nightly&mode=release&edition=2021&gist=bab644e35609b5475978821378d3560f

<!-- more -->

---

Rust enum 本质是一种 [tagged union][tagged union]，对应代数数据类型中的 sum type，这里不过多展开。在 Rust enum 的实现中，通常用一个 byte 来存储 type tag（大部分 enum 不会超过 256 种类型，更多地会相应扩展），也就是说，理想情况下，以下两个结构体是等价的：

```rust
enum Attr {
    Color(u8, u8, u8),
    Shape(u16, u16),
}
```

```c
struct Color { uint8_t r, g, b };
struct Shape { uint16_t w, h };
struct Attr {
    uint8_t tag;
    union {
        Color color;
        Shape shape;
    }
}
```

在这个实现下，enum 在很多场景下并不是 zero overhead 的。幸运的是，Rust 从未定义过 ABI，而带有数据的 enum 甚至是无法被 `repr(C)` 表示的，这给了 Rust 充分的空间对 enum memory layout 进行细粒度的优化。这篇文章会涉及一些在 `rustc 1.60.0-nightly` 下相关优化的介绍。

在开始具体的探索之前，我们需要准备一个辅助函数，来帮我我们查看变量的内存结构：

```rust
fn print_memory<T>(v: &T) {
    println!("{:?}", unsafe {
        core::slice::from_raw_parts(v as *const _ as *const u8, std::mem::size_of_val(v))
    })
}
```

以上面的 `Attr` 为例：

```rust
print_memory(&Attr::Color(1, 2, 3));
// [0, 1, 2, 3, 0, 0]
//  ^  ^  ^  ^  ^^^^
//  |  |  |  |    |
//  |  |  |  |    --- padding
//  |  |  |  -------- b
//  |  |  |---------- g
//  |  |------------- r
//  ----------------- tag

print_memory(&Attr::Shape(257, 258));
// [1, 0, 1, 1, 2, 1]
//  ^  ^  ^^^^  ^^^^
//  |  |    |     |
//  |  |    |     --- h, 256 + 2
//  |  |    --------- w, 256 + 1
//  |  |------------- padding
//  ----------------- tag
```

[tagged union]: https://en.wikipedia.org/wiki/Tagged_union

## `Option<P<T>>`

P 是常见的智能指针类型，包括 `&`/`&mut`/`Box`。这应该是关于 enum layout 优化里最著名的一个例子了。Rust 推荐使用 `Option<P<T>>` 来处理可空指针，这实现了 [null safety][null safety wiki].

[null safety wiki]: https://en.wikipedia.org/wiki/Void_safety

`Option<T>` 在 rust 中被表示为一种 enum：

```rust
pub enum Option<T> {
    None,
    Some(T),
}
```

如果不作任何优化的话，显然是存在不必要的 overhead 的，空指针可以完整地表示 `None` 的语义。由于这种情况太过常见，rustc 不仅针对性地做了优化，而且将其标准化了。

> If T is an FFI-safe non-nullable pointer type, Option<T> is guaranteed to have the same layout and ABI as T and is therefore also FFI-safe. As of this writing, this covers &, &mut, and function pointers, all of which can never be null.

```rust
let p = Box::new(0u64);
print_memory(&p);
// [208, 185, 38, 40, 162, 85, 0, 0]
print_memory(&Some(p));
// [208, 185, 38, 40, 162, 85, 0, 0]
```

一个不算太冷的冷知识是，这种 hack 并不是针对 `Option` 的，而是针对指针类型的。任何自定义的 enum 满足条件也可以达到相同的效果。

```rust
enum MyOption<T> {
    MySome(T),
    MyNone,
}

let p = Box::new(0u64);
print_memory(&MyOption::MySome(p));
// [208, 185, 38, 40, 162, 85, 0, 0]
// The address of `p`.
print_memory(&MyOption::<Box<u64>>::MyNone);
// [0, 0, 0, 0, 0, 0, 0, 0]
// Use nullptr to represent `MyNone`.
```

`Option<P<T>>` 可以优化的根本原因是，P 的内存表示下有一个永远非法的值，而相应的 enum 仅需要表示一个额外的值来表达多余的类型。超出这个约束就会导致这个优化失效。

```rust
enum MyOption2<T> {
    MySome(T),
    MyNone,
    MyNone2,
}

let p = Box::new(0u64);
print_memory(&MyOption2::MySome(p));
// [0, 0, 0, 0, 0, 0, 0, 0, 208, 185, 38, 40, 162, 85, 0, 0]
print_memory(&MyOption2::<Box<u64>>::MyNone2);
// [2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
```


## `bool`, `Ordering`

rust 中的 `bool` 占用一个 byte，且仅有两个合法的值，`True` 和 `False`，对应的内存表示为 `1u08` 和 `0u08`。我们可以理解为 `bool` 有 254 个非法值可以供 type tag 挥霍。

```rust
print_memory(&Some(false));
// [0]
print_memory(&Some(true));
// [1]
print_memory(&(None as Option<bool>));
// [2]
```

更进一步地，我们可以更给力一点：

```rust
print_memory(&Some(Some(false)));
// [0]
print_memory(&Some(Some(true)));
// [1]
print_memory(&(Some(None) as Option<Option<bool>>));
// [2]
print_memory(&(None as Option<Option<bool>>));
// [3]
```

对应的，Ordering 有三种合法值，同样适用于这个优化。

```rust
print_memory(&Some(std::cmp::Ordering::Less));
// [255]
print_memory(&Some(std::cmp::Ordering::Greater));
// [1]
print_memory(&Some(std::cmp::Ordering::Equal));
// [0]
print_memory(&(None as Option<std::cmp::Ordering>));
// [2]
```

## Enum

事实上，编译器并没有对 bool、Ordering 进行特判，任何种类少于 256 的 enum 本身都满足被优化的条件，即 type tag 里会有 (256 - kinds) 个空位。

```rust
enum ShapeKind { Square, Circle }
print_memory(&Some(ShapeKind::Square));
// [0]

print_memory(&MyOption::MySome(MyOption::MySome(1u8)));
// [0, 1]
print_memory(&(MyOption::MyNone as MyOption<MyOption<u8>>));
// [2, 0]
// 尽管 u8 本身没有空位，但是 MyOption 的 type tag 有 254 个空位，因此外层的 MyOption 的 typetag 被优化掉了。
```

## Struct、Tuple

Struct、Tuple 等都属于 Product type。在实现中，往往是将所有字段依次存下来，并做额外的 padding。那么一个理所当然的优化是，如果 struct 的其中一个字段有空位，那么就可以将 enum tag 塞进去。

```rust
struct A1 (i8, bool);
print_memory(&Some(A1(1, false)));
// [1, 0]
print_memory(&Some(A1(1, true)));
// [1, 1]
print_memory(&(None as Option<A1>));
// [0, 2]
```

我们再回到文章开头的例子。


```rust
struct A (i64, i8);
struct B (i64, i8, bool);

dbg!(std::mem::size_of::<A>());
// 16
dbg!(std::mem::size_of::<Option<A>>());
// 24
dbg!(std::mem::size_of::<B>());
// 16
dbg!(std::mem::size_of::<Option<B>>());
// 16

print_memory(&Some(A(1, 1)));
// [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0]
//  ^  ^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^ ^  ^^^^^^^^^^^^^^^^^^^
//  |           |                      |            |           |
//  |           |                      |            |           ------------ padding
//  |           |                      |            ------------------------ .1
//  |           |                      ------------------------------------- .0
//  |           ------------------------------------------------------------ padding
//  ------------------------------------------------------------------------ tag
```

A 和 B 由于 padding，都需要占用 16 个 byte，而 `Option<B>` 由于存在一个 bool 字段 `.2`，tag 被优化进 bool 了，因此也只需要 16 个 byte。反而 `Option<A>` 实打实地用了 24 个 byte。

一个很容易想到的优化是，使用 padding 中未定义的内存来存储 type tag，然而比较遗憾的是，在 padding 中存储的数据在 copy 等行为下都是未定义行为。这也导致了一个比较滑稽的结果，多存了一个字段，`Option` 占用的空间反而减少了。

## 优化自定义结构的可能性

目前，所有 enum layout 相关的优化都适合由编译器针对特定的类型进行 hack 来实现的，我们无法自己控制我们自定义的 struct 在 enum 中的 layout。为了优化一些常用场景，rust 又提供了 [`NonZero*`][NonZeroU8] 等辅助结构体，用来表示非 0 的整数，与此同时编译器会让 `size_of::<Option<NonZeroU8>> == size_of::<u8>`。但这只能由标准库 case by case 处理，而真实的需求是非常复杂的，比如有时候我们可能需要 `NonMaxU64`，或者例如使用了第三方库的 [`Option<ordered_float::NotNan<f64>>`][NotNan] 就无法被优化。

针对自定义接口的优化需要引入非常复杂的机制，在编译期告诉编译器一个类型非法的内存结构有哪些。我目前感觉一个可能的实现是 const trait + const iterator，给编译器提供潜在的非法值的迭代器。不过目前没有看到相关的 RFC。

[NonZeroU8]: https://doc.rust-lang.org/std/num/struct.NonZeroI8.html
[NotNan]: https://docs.rs/ordered-float/2.10.0/ordered_float/struct.NotNan.html#

这篇文章所有的 example 可以在 [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=2cecbb4523443229d32a943bea2e48aa) 找到。
