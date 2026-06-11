+++
title = "Subtyping & Variance in Rust Lifetime"
date = 2024-12-16

[taxonomies]
tags = ["rust"]
+++


# Rust生命周期 Subtyping & Variance

Rust 使用生命周期来跟踪引用和所有权，而生命周期检查实现的背后是 **subtyping**（子类型） 和 **variance**（变型）。

Subtyping 和 variance 本是继承中使用的概念，但是 Rust 中并没有继承机制，Rust 中这两概念用在了生命周期检查上。

## Subtyping

先看一个例子：
```rust
// Note: debug expects two parameters with the *same* lifetime
fn debug<'a>(a: &'a str, b: &'a str) {
    println!("a = {a:?} b = {b:?}");
}

fn main() { // 'static
    let hello: &'static str = "hello";
    { // 'world
        let world = String::from("world");
        let world = &world; // 'world has a shorter lifetime than 'static
        debug(hello, world);
    }
}
```
一个**保守**的生命周期实现不会允许这样的行为，因为 `'static` 并不等于 `'world`。但是直觉告诉我们，这段代码这样做并没有安全问题。尝试编译这段代码，发现确实是可以正常通过检查的。这样灵活的约束得益于 **subtyping**。

首先我们要知道：
- 一个生命周期定义了代码的一个区域
- `'long <: 'short` 表示 `'long` 是 `'short` 的 subtype；当且仅当 `'long` 定义的代码区域完全包括了 `'short` 定义的区域

回到上面的例子，我们可以知道 `'static <: 'world`。

同时，`debug(&'static str, &'world world)` 合法，因此我们说 `&'static str` 是 `&'a str` 的一个 subtype

可以认为在这里 `&'static str` 被“降级”为 `&'a str`，从而匹配了 `debug` 的函数签名。

也就是说，尽管生命周期不完全匹配，只要参数是函数签名要求的subtype，这个行为就是允许的。

## Variance

Variance 描述了 subtype 属性在不同情况下是否可以保持下去的性质。

给定类型 `F<T>`，其中 `T` 为 `Sub` or `Super`， variance 定义了三种关系：
- 如果 `F<Sub>` 是 `F<Super>` 的 subtype，那么我们说 `F` 是 co-variant （协变）的；就好像 subtype 属性传递下去了
- 如果 `F<Super>` 是 `F<Sub>` 的 subtype，那么我们说 `F` 是 contra-variant （逆变）的；就好像 subtype 属性扭转了；这种情况在实际中非常少出现
- 其余情况，`F` 是 in-variant （不变）的；不能应用 subtype 属性

各类型和它们的 variances 如下表：

|                 |     'a    |         T         |     U     |
|-----------------|:---------:|:-----------------:|:---------:|
| `&'a T `        | covariant | covariant         |           |
| `&'a mut T`     | covariant | invariant         |           |
| `Box<T>`        |           | covariant         |           |
| `Vec<T>`        |           | covariant         |           |
| `UnsafeCell<T>` |           | invariant         |           |
| `Cell<T>`       |           | invariant         |           |
| `fn(T) -> U`    |           | **contra**variant | covariant |
| `*const T`      |           | covariant         |           |
| `*mut T`        |           | invariant         |           |

简单来说：
- `Vec<T>` 和其他指针和集合同 `Box<T>`
- `Cell<T>` 和其他具有内部可变性的同 `UnsafeCell<T>`
- `UnsafeCell<T>` 因为它的内部可变性因此同 `&mut T`
- `*const T` 同 `&T`
- `*mut T` 同 `&mut T`

下面结合例子来理解这三种关系。

## Covariance

### `&'a T` over `'a` is covariant

If `'a <: 'b`, then `&'a T <: &'b T`.

```rust
fn debug<T: std::fmt::Debug>(a: T, b: T) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let a: &'static = "hello";
    { // 'b
        let string = String::from("world");
        let b: &str = &string;
        debug(a, b);
    }
}
```

`debug` 的参数必须是相同类型 `T`，在这里类型推断为 `&'a str`，完整函数签名：

`fn debug<'a>(a: &'a str, b: &'a str)`

 `main` 中有 `a: &'static str` 和 `b: &'b str`，且 `'static <: 'b`。

因为 `&'a T` 对 `'a` 是 covariant 的，所以 `&'static str` 在这里可以“降级”为 `&'b str`，被 `debug` 接受。

### `&'a T` over `T` is covariant

If `t1 <: t2`, then `&'a t1 <: &'b t2`.

```rust
fn debug<'a, T: std::fmt::Debug>(a: &'a T, b: T) {
    println!("a = {a:?} b = {b:?}");
}

fn main() {
    let a: &'static str = "hello";
    { // 'b
        let string = String::from("world");
        let b: &str = &string;
        debug(&a, b);
    }
}
```
这次参数变成了 `(a: &'a T, b: T)`。

我们传入的参数为 `(&'b &'static str, &'b str)`

类型推导过程中，

当 `T = &'static str'`，编译器判定不能满足生命周期约束，因为 `&'b str <: &'static` 不成立；

当 `T = &'b str`，根据规则， `'static <: 'b` => `&'static str <: &'b str`。
从而可以将 `a: &'a &'static str` “降级”为 `&'a &'b str`，成功匹配了函数签名。

### `&'a mut T` over `'a` is covariant

If `'a <: 'b`, then `&'a mut T <: &'b mut T`.

```rust
fn debug<'a, T: std::fmt::Debug>(a: &'a mut T) {
    println!("a={:?}", a);
}

fn main() {
    let mut a: &'static str = "hello";
    debug(&mut a);
}
```

参数：`(&'static mut &'static str)`

根据规则： `'static <: 'a` => `&'static mut &'static str <: &'a mut &'static str`

类型推断 `T = &'static str`，匹配成功。

## Invariance

### `&'a mut T'` over `T` is invariant

Even if `t1 <: t2`, there's no `&'a mut t1 <: &'a mut t2`.

```rust
fn debug<'a, T: std::fmt::Debug>(a: &'a mut T, b: &'a T) {
    println!("a={:?}, b={:?}", a, b);
}

fn main() {
    let mut a: &'static str = "hello";
    { // 'b
        let string = String::from("world");
        let b = string.as_str();
        debug(&mut a, &b);  // won't compile
    }
}
```

参数：`(&'b mut &'static str, &'b '&'b str)`

尽管有 `&'static str <: &'b str`（根据 `&'a T` over `'a` is covariant）

我们不能得出 `&'b mut &'static str <: &'b mut &'b str`

因此 `T` 必须是 `&'static str` （即不允许“降级”）

而 `&'b str <: &'static str` 不成立，所以两个参数的 `T` 无法统一，编译无法通过。

----

**TIP**：

下面的代码是可以正常运行的

```rust
fn debug<T: std::fmt::Debug>(a: &mut T, b: T) {
    println!("a={:?}, b={:?}", a, b);
}

fn debug_str<T: std::fmt::Debug>(a: &T) {
    println!("{:?}", a);
}

fn main() {
    let mut a = "hello"; // &str, with an implicit lifetime.
    debug_str(&a);
    { // 'b
        let string = String::from("world");
        let b = string.as_str();
        debug(&mut a, b);
    }
}
```
参数： `(&'b mut &'b str, &'b str)`

这是因为在没有手动指定生命周期的情况下，编译器会根据代码逻辑自动推断生命周期。

`a` 的生命周期被“压缩”到了 `'b`，使得它能匹配 `debug` 签名同时不违反 invariance。

而下面的代码：
```rust
fn debug<T: std::fmt::Debug>(a: &mut T, b: T) {
    println!("a={:?}, b={:?}", a, b);
}

fn debug_static<T: std::fmt::Debug>(a: &'static T) {
    println!("{:?}", a);
}

fn main() {
    let mut a = "hello"; // &str, with no explicit lifetime.
    debug_static(&a); // 1.
    { // 'b
        let string = String::from("world");
        let b = string.as_str();
        debug(&mut a, b);
    }
    println!("a={:?}", a); // 2.

    // 1. sets lifetime of `a` to 'static
    // 2. needs lifetime of `a` at least longer than 'b
}
```
上面的代码是不能编译通过的，因为编译器推断出的生命周期 
1. `'static` 以及
2. `at least longer than 'b`

`1.` 和 `2.` 任意存在都会导致违反 `&'a mut T` 对 `T` 的 invariance.

## Contravariance

最后，也是是唯一一个的 contra-variant。

`fn(T) -> U` over `T` is contravariant.

If `t1 <: t2`, then `fn(t2) <: fn(t1)`.

直接看例子：
```rust
thread_local! {
    pub static StaticVecs: RefCell<Vec<&'static str>> = RefCell::new(Vec::new());
}

/// saves the input given into a thread local `Vec<&'static str>`
fn store(input: &'static str) {
    StaticVecs.with_borrow_mut(|v| v.push(input));
}

/// Calls the function with it's input (must have the same lifetime!)
fn demo<'a>(input: &'a str, f: fn(&'a str)) {
    f(input);
}

fn main() {
    demo("hello", store); // "hello" is 'static. Can call `store` fine

    {
        let smuggle = String::from("smuggle");

        // `&smuggle` is not static. If we were to call `store` with `&smuggle`,
        // we would have pushed an invalid lifetime into the `StaticVecs`.
        // Therefore, `fn(&'static str)` cannot be a subtype of `fn(&'a str)`
        demo(&smuggle, store);
    }

    // use after free...
    StaticVecs.with_borrow(|v| println!("{v:?}"));
}
```

**References**

https://doc.rust-lang.org/nomicon/subtyping.html
https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)
