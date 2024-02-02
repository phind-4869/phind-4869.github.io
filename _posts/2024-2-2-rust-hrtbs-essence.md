---
title: 从另一个视角看 Rust HRTBs
date: 2024-2-2 16:58:04 +0800
categories: [杂记, Rust]
tags: [rust, 编程语言]     # TAG names should always be lowercase
---

HRTBs，即**高阶 Trait 约束（Higher-Rank Trait Bounds）**，在 Rust 中是令很多初学者感到莫名其妙的概念，一手 `for<'a> S<'a>`{:.language-rust} 的语法更是使得原本就复杂的生命周期更加吓人。

但是，如果从另一个角度对 HRTBs 进行解剖，或许我们能看到不一样的东西。

首先，让我们考虑一个泛型和闭包的应用：

```rust
use std::fmt::Display;

fn print<T, U>(x: T, y: U)
where
    T: Display,
    U: Display,
{
    println!("{x}, {y}")
}

fn curry<F, T, U, R>(f: F, a: T) -> impl Fn(U) -> R
where
    F: Fn(T, U) -> R,
    T: Copy,
{
    move |b| f(a, b)
}

fn main() {
    let print_one = curry(print, 1);
    print_one(1);
}
```
{: run="rust" }

这段代码可能稍显复杂，简单来说，我们首先定义了一个泛型 `print` 函数，可以接受任意两个可打印的类型的值；然后我们定义了一个柯里化函数 `curry`，它可以将一个函数 `f` 和它所需的第一个参数进行打包，产生一个新的闭包，而这个闭包以函数 `f` 所需的第二个参数进行调用时，则会将两个参数同时传递给它进行调用。

在 `main` 函数中，我们将函数 `print` 和参数 `1`{:.language-rust} 进行打包，得到一个闭包 `print_one`，该 `print_one` 可以接收另一个参数，然后将之前打包的 `1`{:.language-rust} 和新的参数一起传递给 `print` 调用。因此，上述代码最终会在终端打印 `1, 1`。

但是，如果我们想在下面再继续调用 `print_one(1.0)`{:.language-rust} 或 `print_one("hello")`{:.language-rust} 或其他实现了 `Display` trait 的类型的值时，rust 则会严厉地拒绝我们：

```text
error[E0308]: mismatched types
  --> src/main.rs:22:15
   |
22 |     print_one(1.0);
   |     --------- ^^^ expected integer, found floating-point number
   |     |
   |     arguments to this function are incorrect
```

这是因为 Rust 是静态类型语言，在我们调用 `print_one(1)`{:.language-rust} 时，就**把 `print_one` 推导并固定为 `impl Fn(i32) -> i32`{:.language-rust}**，从而不能再接收并不是 `i32`{:.language-rust} 类型的参数。因此，我们需要注释掉 `print_one(1)`{:.language-rust}，才能以其他类型的值调用 `print_one`。

然而，在另一门同样是静态类型的语言 Haskell 中，一切又似乎是那么的不同：

```haskell
ghci> print x y = (show x) ++ (show y) -- 定义 print 函数，haskell 默认此函数是泛型
ghci> print_one = print 1 -- 函数式语言原生支持柯里化，只需以数量不足的参数调用函数即可
ghci> print_one 1 -- 以整数类型调用 print_one
"1, 1"
ghci> print_one 1.0 -- 以小数类型调用 print_one
"1, 1.0"
ghci> print_one "hello" -- 以字符串类型调用 print_one
"1, \"hello\""
```

令人惊讶，同为静态类型语言，Haskell 却能做到这种神奇的事情。不难发现，在上述例子中，显然 **`print_one` 是一个泛型函数**，这意味着，**`print 1`{:.language-haskell} 这个表达式，返回的竟然是一个泛型类型！**

如果，假设我们是 Rust 的设计者，我们想要在 Rust 中支持这一功能，我们应当怎么实现它？

首先，返回值类型 `impl Fn(U) -> R`{:.language-rust} 的泛型参数 `U` 和 `R` 都不能再依赖 `curry` 的泛型参数，否则 `curry` 被实例化时，这两个类型都会被固定，`F` 类型也是如此。那么，我们需要**为这些与 `curry` 函数不相干的泛型参数定义一个新的语法来声明这些泛型参数**，我们应该如何设计？

在这里，我们假设我们从来都不知道什么 HRTBs，于是，我们误打误撞选了 **`for<T>`{:.language-rust}** 这个语法，`curry` 函数就变成了：

```rust
fn curry<F, T>(f: F, a: T) -> for<U, R> impl Fn(U) -> R
where
    F: for<U, R> Fn(T, U) -> R,
    T: Copy,
{
    move |b| f(a, b)
}
```

在这个新的 `curry` 函数中，**泛型 `U` 和 `R` 不再和函数 `curry` 绑定，因此，实例化 `curry` 并不会固定 `U` 和 `R` 的类型**。当然，Rust 到目前为止并没有支持 `for<T>`{:.language-rust} 这种语法，一切都是我们的一厢情愿罢了。

然而，在 Rust 中，确切地有一个类似的东西，**他就是 HRTBs，它的语法是 `for<'a>`{:.language-rust}**。到了这里，我相信不少看客内心多少都有了一些新的想法。

让我们来看一个生命周期的例子：

```rust
trait DoSth<T> {
    fn do_sth(&self, _t: T) {
        todo!()
    }
}

fn do_sth_for_i32(r: &dyn DoSth<&i32>) {
    let num = 1;
    r.do_sth(&num);
}
```

上述例子无法通过编译。Rust 向我们抱怨 num 的生命周期不够长：

```text
error[E0597]: `num` does not live long enough
  --> src/main.rs:9:14
   |
7  | fn do_sth_for_i32(r: &dyn DoSth<&i32>) {
   |                   - has type `&dyn DoSth<&'1 i32>`
8  |     let num = 1;
   |         --- binding `num` declared here
9  |     r.do_sth(&num);
   |     ---------^^^^-
   |     |        |
   |     |        borrowed value does not live long enough
   |     argument requires that `num` is borrowed for `'1`
10 | }
   | - `num` dropped here while still borrowed
```

这是为什么呢？事实上，原因和之前类似，我们知道，`&i32`{:.language-rust} 隐含了一个生命周期参数 `&'a i32`{:.language-rust}，Rust 在实例化 `dyn DoSth<&i32>`{:.language-rust} 这一泛型时，已经将 `&'a i32`{:.language-rust} 的生命周期 `'a` 实例化为了一个固定的生命周期。Rust 无从知晓我们究竟会以什么样的方式使用该生命周期，因此，Rust 采用了最保险的办法，它认为该生命周期至少要大于 `dyn DoSth<&i32>`{:.language-rust} 的生命周期，自然，我们无法将 `num` 的引用传入。

而想要解决这个问题，我们所采取的手段也是类似的：我们需要引入一个新的语法，用以声明该生命周期是与 `dyn DoSth<&i32>`{:.language-rust} 的实例化是无关的，该类型仍然是一个关于生命周期的泛型类型。这个语法就是 HRTBs，即 `for<'a>`{:.language-rust}，**我们将它改成 `dyn for<'a> DoSth<&'a i32>`{:.language-rust}，这个例子就可以通过编译**：

```rust
trait DoSth<T> {
    fn do_sth(&self, _t: T) {
        todo!()
    }
}

fn do_sth_for_i32(r: &dyn for<'a> DoSth<&'a i32>) {
    let num = 1;
    r.do_sth(&num);
}
```

最后，仍然是惯例提醒一下，在使用形如 `Fn(&T) -> &U`{:.language-rust} 的语法时，Rust **隐含地为其添加了 HRTBs**，也就是说它实际是 `for<'a> Fn(&'a T) -> &'a U`{:.language-rust}，并不需要我们显式地写出 HRTBs 语法；而当使用形如 `Fn(&T, &U) -> &W`{:.language-rust} 的语法时，Rust 不知道返回值的生命周期受到哪个参数的约束，因此，仍然需要我们添加 HRTBs 才能通过编译。
