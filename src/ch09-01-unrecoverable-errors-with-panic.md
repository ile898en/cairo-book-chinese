# 使用 `panic` 处理不可恢复错误

在 Cairo 中，程序执行期间可能会出现意外问题，从而导致运行时错误。虽然核心库中的 `panic` 函数无法解决这些错误，但它确实承认了它们的发生并终止程序。在 Cairo 中触发 panic 主要有两种方式：无意中，通过导致代码 panic 的操作（例如，访问超出其边界的数组）；或者故意，通过调用 `panic` 函数。

当 panic 发生时，它会导致程序突然终止。`panic` 函数接受一个数组作为参数，该数组可用于提供错误消息并执行展开 (unwind) 过程，其中所有变量都被删除，字典被压缩 (squashed)，以确保程序的安全性以安全地终止执行。

这是我们如何从程序内部调用 `panic` 并返回错误代码 `2` 的方法：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
// TAG: does_not_run

#[executable]
fn main() {
    let mut data = array![2];

    if true {
        panic(data);
    }
    println!("This line isn't reached");
}
```

运行该程序将产生以下输出：

```shell
$ scarb execute 
   Compiling no_listing_01_panic v0.1.0 (listings/ch09-error-handling/no_listing_01_panic/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_01_panic
error: Panicked with 0x2.

```

正如你在输出中注意到的那样，永远不会到达对 `println!` 宏的调用，因为程序在遇到 `panic` 语句后终止。

在 Cairo 中处理 panic 的另一种更惯用的方法是使用 `panic_with_felt252` 函数。此函数充当数组定义过程的抽象，并且由于其更清晰、更简洁的意图表达而通常是首选。通过使用 `panic_with_felt252`，开发人员可以通过提供 `felt252` 错误消息作为参数在一行代码中 panic，从而使代码更具可读性和可维护性。

让我们看一个例子：

```cairo
// TAG: does_not_run

use core::panic_with_felt252;

#[executable]
fn main() {
    panic_with_felt252(2);
}
```

执行此程序将产生与之前相同的错误消息。在那种情况下，如果不需要在错误中返回数组和多个值，`panic_with_felt252` 是一个更简洁的替代方案。

## `panic!` 宏

`panic!` 宏真的很有帮助。之前返回错误代码 `2` 的示例展示了 `panic!` 宏是多么方便。不需要像使用 `panic` 函数那样创建一个数组并将其作为参数传递。

```cairo
// TAG: does_not_run

#[executable]
fn main() {
    if true {
        panic!("2");
    }
    println!("This line isn't reached");
}
```

与 `panic_with_felt252` 函数不同，使用 `panic!` 允许输入（最终是 panic 错误）是长度超过 31 字节的字面量。这是因为 `panic!` 接受一个字符串作为参数。例如，以下代码行将成功编译：

```cairo, noplayground
panic!("the error for panic! macro is not limited to 31 characters anymore");
```

## `nopanic` 标记

你可以使用 `nopanic` 标记来指示一个函数永远不会 panic。只有 `nopanic` 函数才能在被注释为 `nopanic` 的函数中调用。

这是一个例子：

```cairo,noplayground
fn function_never_panic() -> felt252 nopanic {
    42
}
```

此函数将始终返回 `42`，并且保证永远不会 panic。
相反，以下函数不能保证永远不会 panic：

```cairo,noplayground
fn function_never_panic() nopanic {
    assert!(1 == 1, "what");
}
```

如果你尝试编译此包含可能 panic 的代码的函数，你将收到以下错误：

```shell
$ scarb execute 
   Compiling no_listing_04_nopanic_wrong v0.1.0 (listings/ch09-error-handling/no_listing_05_nopanic_wrong/Scarb.toml)
error: Function is declared as nopanic but calls a function that may panic.
 --> listings/ch09-error-handling/no_listing_05_nopanic_wrong/src/lib.cairo:4:13
    assert!(1 == 1, "what");
            ^^^^^^

error: Function is declared as nopanic but calls a function that may panic.
 --> listings/ch09-error-handling/no_listing_05_nopanic_wrong/src/lib.cairo:4:5
    assert!(1 == 1, "what");
    ^^^^^^^^^^^^^^^^^^^^^^^

error: could not compile `no_listing_04_nopanic_wrong` due to previous error
error: `scarb` command exited with error

```

注意这里有两个可能 panic 的函数：`assert` 和使用 `==` 的相等性检查。我们在实践中通常不使用 `assert` 函数，而是使用 `assert!` 宏。我们将在 [测试 Cairo 程序][assert macro] 一章中更详细地讨论 `assert!` 宏。

[assert macro]:
  ./ch10-01-how-to-write-tests.md#checking-results-with-the-assert-macro

## `panic_with` 属性

你可以使用 `panic_with` 属性来标记返回 `Option` 或 `Result` 的函数。此属性接受两个参数，即作为 panic 原因传递的数据以及包装函数的名称。如果函数返回 `None` 或 `Err`，它将为你的带注释函数创建一个包装器，该包装器将使用给定的数据作为 panic 错误进行 panic。

例子：

```cairo
// TAG: does_not_run

#[panic_with('value is 0', wrap_not_zero)]
fn wrap_if_not_zero(value: u128) -> Option<u128> {
    if value == 0 {
        None
    } else {
        Some(value)
    }
}

#[executable]
fn main() {
    wrap_if_not_zero(0); // this returns None
    wrap_not_zero(0); // this panics with 'value is 0'
}
```

{{#quiz ../quizzes/ch09-01-unrecoverable-errors-with-panic.toml}}
