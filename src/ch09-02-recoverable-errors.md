# 使用 `Result` 处理可恢复错误

大多数错误并没有严重到需要程序完全停止。有时，当函数失败时，这是出于你可以轻松解释和响应的原因。例如，如果你尝试将两个大整数相加，并且由于总和超过了最大可表示值而导致操作溢出，你可能希望返回错误或包装的结果，而不是导致未定义的行为或终止进程。

## `Result` 枚举

回忆一下 [第 {{#chap generic-types-and-traits}} 章中的泛型数据类型][generic enums] 部分，`Result` 枚举定义为具有两个变体，`Ok` 和 `Err`，如下所示：

```cairo,noplayground
enum Result<T, E> {
    Ok: T,
    Err: E,
}
```

`Result<T, E>` 枚举有两个泛型类型 `T` 和 `E`，以及两个变体：`Ok` 持有类型 `T` 的值，`Err` 持有类型 `E` 的值。这个定义使得在任何我们有一个可能成功（通过返回类型 `T` 的值）或失败（通过返回类型 `E` 的值）的操作的地方使用 `Result` 枚举变得方便。

[generic enums]: ./ch08-01-generic-data-types.md#enums

## `ResultTrait`

`ResultTrait` trait 提供了用于处理 `Result<T, E>` 枚举的方法，例如解包值、检查 `Result` 是 `Ok` 还是 `Err` 以及使用自定义消息 panic。`ResultTraitImpl` 实现定义了这些方法的逻辑。

```cairo,noplayground
trait ResultTrait<T, E> {
    fn expect<+Drop<E>>(self: Result<T, E>, err: felt252) -> T;

    fn unwrap<+Drop<E>>(self: Result<T, E>) -> T;

    fn expect_err<+Drop<T>>(self: Result<T, E>, err: felt252) -> E;

    fn unwrap_err<+Drop<T>>(self: Result<T, E>) -> E;

    fn is_ok(self: @Result<T, E>) -> bool;

    fn is_err(self: @Result<T, E>) -> bool;
}
```

`expect` 和 `unwrap` 方法类似，它们都试图在 `Result<T, E>` 处于 `Ok` 变体时从中提取类型 `T` 的值。如果 `Result` 是 `Ok(x)`，这两种方法都返回以此值 `x`。然而，这两种方法之间的关键区别在于当 `Result` 处于 `Err` 变体时的行为。`expect` 方法允许你提供自定义错误消息（作为 `felt252` 值），该消息将在 panic 时使用，从而让你对 panic 有更多的控制和上下文。另一方面，`unwrap` 方法使用默认错误消息 panic，提供的关于 panic 原因的信息较少。

`expect_err` 和 `unwrap_err` 方法具有完全相反的行为。如果 `Result` 是 `Err(x)`，这两种方法都返回以此值 `x`。然而，这两种方法之间的关键区别在于 `Ok()` 的情况。`expect_err` 方法允许你提供自定义错误消息（作为 `felt252` 值），该消息将在 panic 时使用，从而让你对 panic 有更多的控制和上下文。另一方面，`unwrap_err` 方法使用默认错误消息 panic，提供的关于 panic 原因的信息较少。

细心的读者可能已经注意到前四个方法签名中的 `<+Drop<T>>` 和 `<+Drop<E>>`。这种语法表示 Cairo 语言中的泛型类型约束，如上一章所示。这些约束表明，关联函数分别需要泛型类型 `T` 和 `E` 的 `Drop` trait 实现。

最后，`is_ok` 和 `is_err` 方法是 `ResultTrait` trait 提供的实用函数，用于检查 `Result` 枚举值的变体。

- `is_ok` 接受 `Result<T, E>` 值的快照，如果 `Result` 是 `Ok` 变体（意味着操作成功），则返回 `true`。如果 `Result` 是 `Err` 变体，则返回 `false`。
- `is_err` 接受 `Result<T, E>` 值的快照，如果 `Result` 是 `Err` 变体（意味着操作遇到错误），则返回 `true`。如果 `Result` 是 `Ok` 变体，则返回 `false`。

当你想要检查操作的成功或失败而不消耗 `Result` 值时，这些方法很有帮助，允许你在不解包它的情况下执行其他操作或根据变体做出决定。

你可以在 [这里][result corelib] 找到 `ResultTrait` 的实现。

用例子总是更容易理解。看看这个函数签名：

```cairo,noplayground
fn u128_overflowing_add(a: u128, b: u128) -> Result<u128, u128>;
```

它接受两个 `u128` 整数 `a` 和 `b`，并返回一个 `Result<u128, u128>`，如果加法不溢出，`Ok` 变体持有总和，如果加法溢出，则 `Err` 变体持有溢出的值。

现在，我们可以在其他地方使用这个函数。例如：

```cairo,noplayground
fn u128_checked_add(a: u128, b: u128) -> Option<u128> {
    match u128_overflowing_add(a, b) {
        Ok(r) => Some(r),
        Err(r) => None,
    }
}

```

在这里，它接受两个 `u128` 整数 `a` 和 `b`，并返回一个 `Option<u128>`。它使用 `u128_overflowing_add` 返回的 `Result` 来确定加法操作的成功或失败。`match` 表达式检查来自 `u128_overflowing_add` 的 `Result`。如果结果是 `Ok(r)`，它返回包含总和的 `Some(r)`。如果结果是 `Err(r)`，它返回 `None` 以指示由于溢出而导致操作失败。该函数在溢出的情况下不会 panic。

让我们再举一个例子：

```cairo,noplayground
fn parse_u8(s: felt252) -> Result<u8, felt252> {
    match s.try_into() {
        Some(value) => Ok(value),
        None => Err('Invalid integer'),
    }
}
```

在这个例子中，`parse_u8` 函数接受一个 `felt252` 并尝试使用 `try_into` 方法将其转换为 `u8` 整数。如果成功，它返回 `Ok(value)`，否则返回 `Err('Invalid integer')`。

我们的两个测试用例是：

```cairo,noplayground
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_felt252_to_u8() {
        let number: felt252 = 5;
        // should not panic
        let res = parse_u8(number).unwrap();
    }

    #[test]
    #[should_panic]
    fn test_felt252_to_u8_panic() {
        let number: felt252 = 256;
        // should panic
        let res = parse_u8(number).unwrap();
    }
}
```

现在不要担心 `#[cfg(test)]` 属性。我们将在下一章 [测试 Cairo 程序][tests] 中更详细地解释它的含义。

`#[test]` 属性意味着该函数是一个测试函数，`#[should_panic]` 属性意味着如果测试执行 panic，则该测试将通过。

第一个测试从 `felt252` 到 `u8` 的有效转换，期望 `unwrap` 方法不 panic。第二个测试函数尝试转换超出 `u8` 范围的值，期望 `unwrap` 方法使用错误消息 `Invalid integer` panic。

[result corelib]:
  https://github.com/starkware-libs/cairo/blob/main/corelib/src/result.cairo#L20
[tests]: ./ch10-01-how-to-write-tests.md

## 传播错误

当函数的实现调用可能会失败的东西时，你可以将错误返回给调用代码，以便它可以决定要做什么，而不是在函数本身内部处理错误。这被称为 _传播 (propagating)_ 错误，并赋予调用代码更多的控制权，因为调用代码中可能有比你在代码上下文中可用的更多的信息或逻辑来决定应该如何处理错误。

例如，清单 {{#ref match-example}} 展示了一个函数的实现，它尝试将数字解析为 `u8` 并使用 match 表达式来处理潜在的错误。

```cairo, noplayground
// A hypothetical function that might fail
fn parse_u8(input: felt252) -> Result<u8, felt252> {
    let input_u256: u256 = input.into();
    if input_u256 < 256 {
        Result::Ok(input.try_into().unwrap())
    } else {
        Result::Err('Invalid Integer')
    }
}

fn mutate_byte(input: felt252) -> Result<u8, felt252> {
    let input_to_u8 = match parse_u8(input) {
        Result::Ok(num) => num,
        Result::Err(err) => { return Result::Err(err); },
    };
    let res = input_to_u8 - 1;
    Result::Ok(res)
}
```

{{#label match-example}} <span class="caption">清单 {{#ref match-example}}:
使用 `match` 表达式将错误返回给调用代码的函数。</span>

调用此 `parse_u8` 的代码将处理获取包含数字的 `Ok` 值或包含错误消息的 `Err` 值。由调用代码决定如何处理这些值。如果调用代码获得 `Err` 值，它可以调用 `panic!` 并使程序崩溃，或者使用默认值。我们就调用代码实际上试图做什么没有足够的信息，所以我们将所有成功或错误信息向上传播，以便它可以适当地处理。

这种传播错误的模式在 Cairo 中非常常见，以至于 Cairo 提供了问号运算符 `?` 来使其更容易。

## 传播错误的快捷方式：`?` 运算符

清单 {{#ref question-operator}} 展示了 `mutate_byte` 的实现，它具有与清单 {{#ref match-example}} 中的功能相同的功能，但使用 `?` 运算符来优雅地处理错误。

```cairo, noplayground
fn mutate_byte(input: felt252) -> Result<u8, felt252> {
    let input_to_u8: u8 = parse_u8(input)?;
    let res = input_to_u8 - 1;
    Ok(res)
}
```

{{#label question-operator}} <span class="caption">清单
{{#ref question-operator}}: 使用 `?` 运算符将错误返回给调用代码的函数。</span>

放置在 `Result` 值之后的 `?` 被定义为与我们在清单 1 中定义的用于处理 `Result` 值的 `match` 表达式几乎相同的方式工作。如果 `Result` 的值是 `Ok`，则 `Ok` 内部的值将从该表达式返回，并且程序将继续。如果值是 `Err`，则 `Err` 将从整个函数返回，就像我们使用了 `return` 关键字一样，以便错误值传播到调用代码。

在清单 2 的上下文中，`parse_u8` 调用末尾的 `?` 将把 `Ok` 内部的值返回给变量 `input_to_u8`。如果发生错误，`?` 运算符将提前退出整个函数并将任何 `Err` 值提供给调用代码。

`?` 运算符消除了大量的样板代码，并使该函数的实现更简单且更符合人体工程学。

### 可以在哪里使用 `?` 运算符

`?` 运算符只能在返回类型与 `?` 作用的值兼容的函数中使用。这是因为 `?` 运算符被定义为从函数中提前返回一个值，这与我们在清单 {{#ref match-example}} 中定义的 `match` 表达式的方式相同。在清单 {{#ref match-example}} 中，`match` 使用的是 `Result` 值，并且提前返回分支返回了一个 `Err(e)` 值。函数的返回类型必须是 `Result`，以便它与此返回兼容。

在清单 {{#ref question-operator-wrong-return}} 中，让我们看看如果我们在这个返回类型与我们使用 `?` 的值的类型不兼容的函数中使用 `?` 运算符会得到什么错误。

```cairo
#[executable]
fn main() {
    let some_num = parse_u8(258)?;
}
```

{{#label question-operator-wrong-return}} <span class="caption">清单
{{#ref question-operator-wrong-return}}: 尝试在返回 `()` 的 `main` 函数中使用 `?` 将无法编译。</span>

这段代码调用了一个可能会失败的函数。`?` 运算符跟在 `parse_u8` 返回的 `Result` 值之后，但这 `main` 函数的返回类型是 `()`，而不是 `Result`。当我们编译这段代码时，我们会得到类似这样的错误消息：

```text
$ scarb build 
   Compiling listing_invalid_qmark v0.1.0 (listings/ch09-error-handling/listing_invalid_qmark/Scarb.toml)
error: `?` can only be used in a function with `Option` or `Result` return type.
 --> listings/ch09-error-handling/listing_invalid_qmark/src/lib.cairo:6:20
    let some_num = parse_u8(258)?;
                   ^^^^^^^^^^^^^^

warn[E0001]: Unused variable. Consider ignoring by prefixing with `_`.
 --> listings/ch09-error-handling/listing_invalid_qmark/src/lib.cairo:6:9
    let some_num = parse_u8(258)?;
        ^^^^^^^^

error: could not compile `listing_invalid_qmark` due to previous error

```

此错误指出我们只允许在返回 `Result` 或 `Option` 的函数中使用 `?` 运算符。

要修复此错误，你有两个选择。一种选择是将函数的返回类型更改为与你使用 `?` 运算符的值兼容，只要你没有阻止这样做的限制。另一种选择是使用 `match` 以任何合适的方式处理 `Result<T, E>`。

错误消息还提到 `?` 也可以与 `Option<T>` 值一起使用。与在 `Result` 上使用 `?` 一样，你只能在返回 `Option` 的函数中在 `Option` 上使用 `?`。在 `Option<T>` 上调用时，`?` 运算符的行为与其在 `Result<T, E>` 上调用时的行为类似：如果值是 `None`，则 `None` 将在那时从函数中提前返回。如果值是 `Some`，则 `Some` 内部的值是表达式的结果值，并且函数继续。

### 总结

我们看到可以使用 `Result` 枚举在 Cairo 中处理可恢复错误，该枚举有两个变体：`Ok` 和 `Err`。`Result<T, E>` 枚举是泛型的，类型 `T` 和 `E` 分别代表成功和错误值。`ResultTrait` 提供了用于处理 `Result<T, E>` 的方法，例如解包值、检查结果是 `Ok` 还是 `Err` 以及使用自定义消息 panic。

为了处理可恢复错误，函数可以返回 `Result` 类型并使用模式匹配来处理操作的成功或失败。`?` 运算符可用于通过传播错误或解包成功值来隐式处理错误。这允许更简洁和清晰的错误处理，其中调用者负责管理被调用函数引发的错误。

{{#quiz ../quizzes/ch09-02-error-handling-result.toml}}
