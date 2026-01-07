# 注释

所有程序员都努力使他们的代码易于理解，但有时额外的解释是必要的。在这些情况下，程序员在源代码中留下注释，编译器会忽略这些注释，但阅读源代码的人可能会发现它们很有用。

这是一个简单的注释：

```cairo,noplayground
// hello, world
```

在 Cairo 中，惯用的注释风格是以两个斜杠开始注释，注释一直持续到行尾。对于超过一行的注释，你需要在每一行包含 `//`，像这样：

```cairo,noplayground
// So we’re doing something complicated here, long enough that we need
// multiple lines of comments to do it! Whew! Hopefully, this comment will
// explain what’s going on.
```

注释也可以放在包含代码的行的末尾：

```cairo
#[executable]
fn main() -> felt252 {
    1 + 4 // return the sum of 1 and 4
}
```

但你会更频繁地看到这种格式，注释位于它所注视的代码上方的单独一行：

```cairo
#[executable]
fn main() -> felt252 {
    // this function performs a simple addition
    1 + 4
}
```

## 项目级文档 (Item-level Documentation)

项目级文档注释指的是特定的项目，如函数、实现、Trait 等。它们以前缀三个斜杠 (`///`) 开头。这些注释提供了对项目的详细描述、使用示例以及可能导致 panic 的条件。在函数的情况下，注释还可以包括参数和返回值描述的单独部分。

```cairo,noplayground
/// Returns the sum of `arg1` and `arg2`.
/// `arg1` cannot be zero.
///
/// # Panics
///
/// This function will panic if `arg1` is `0`.
///
/// # Examples
///
/// ```
/// let a: felt252 = 2;
/// let b: felt252 = 3;
/// let c: felt252 = add(a, b);
/// assert!(c == a + b, "Should equal a + b");
/// ```
fn add(arg1: felt252, arg2: felt252) -> felt252 {
    assert!(arg1 != 0, "Cannot be zero");
    arg1 + arg2
}
```

## 模块文档 (Module Documentation)

模块文档注释提供整个模块的概述，包括其目的和使用示例。这些注释旨在放在它们描述的模块上方，并以前缀 `//!` 开头。这种类型的文档提供了对模块作用以及如何使用的广泛理解。

```cairo,noplayground
//! # my_module and implementation
//!
//! This is an example description of my_module and some of its features.
//!
//! # Examples
//!
//! ```
//! mod my_other_module {
//!   use path::to::my_module;
//!
//!   fn foo() {
//!     my_module.bar();
//!   }
//! }
//! ```
mod my_module { // rest of implementation...
}
```

{{#quiz ../quizzes/ch02-04-comments.toml}}
