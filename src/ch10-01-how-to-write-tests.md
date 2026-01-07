# 如何编写测试

## 测试函数的解剖结构

测试是 Cairo 函数，用于验证非测试代码是否按预期方式运行。测试函数的主体通常执行以下三个操作：

- 设置任何所需的数据或状态。
- 运行你想要测试的代码。
- 断言结果是否符合你的预期。

让我们看看 Cairo 提供的用于编写执行这些操作的测试的功能，其中包括：

- `#[test]` 属性。
- `assert!` 宏。
- `assert_eq!`, `assert_ne!`, `assert_lt!`, `assert_le!`, `assert_gt!` 和 `assert_ge!` 宏。为了使用它们，如果你使用的是旧版本，可能需要添加 `assert_macros` 依赖，但在最新版本中通常已包含（注：原文提到 dev dependency，视具体 Scarb 版本而定）。
- `#[should_panic]` 属性。

> 注意：确在创建项目时选择 Starknet Foundry 作为测试运行器。

### 测试函数的解剖结构

最简单地说，Cairo 中的测试是一个带有 `#[test]` 属性注释的函数。属性是关于 Cairo 代码片段的元数据；一个例子是我们在 [第 {{#chap using-structs-to-structure-related-data}} 章][structs] 中对结构体使用的 `#[derive()]` 属性。要将函数更改为测试函数，请在 `fn` 之前的一行添加 `#[test]`。当你使用 `scarb test` 命令运行测试时，Scarb 会运行 Starknet Foundry 的测试运行器二进制文件，该文件运行带注释的函数并报告每个测试函数是通过还是失败。

让我们使用 Scarb 创建一个名为 _adder_ 的新项目，命令为 `scarb new adder`。删除 _tests_ 文件夹。

```shell
adder
├── Scarb.toml
└── src
    └── lib.cairo
```

在 _lib.cairo_ 中，让我们删除现有内容并添加一个包含第一个测试的 `tests` 模块，如清单 {{#ref first-test}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

{{#label first-test}} <span class="caption">清单 {{#ref first-test}}: 一个简单的测试函数</span>

注意 `#[test]` 注释：此属性指示这是一个测试函数，因此测试运行器知道将此函数视为测试。我们可能还有非测试函数来帮助设置常见场景或执行常见操作，因此我们总是需要指明哪些函数是测试。

我们需要对 `tests` 模块使用 `#[cfg(test)]` 属性，以便编译器知道它包含的代码只需在运行测试时编译。这实际上不是一个选项：如果你在 _lib.cairo_ 文件中放置一个带有 `#[test]` 属性的简单测试，它将不会编译。我们将在下一节 [测试组织][test organization] 中更多地讨论 `#[cfg(test)]` 属性。

示例函数体使用了 `assert_eq!` 宏，它断言 2 加 2 的结果等于 4。此断言用作典型测试格式的示例。我们稍后将在本章中更详细地解释 `assert_eq!` 的工作原理。让我们运行它看看这个测试是否通过。

`scarb test` 命令运行在我们的项目中找到的所有测试，并显示以下输出：

```shell
$ scarb test 
     Running test listing_10_01 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_01_unittest) listing_10_01 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_01/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from listing_10_01 package
Running 2 test(s) from src/
[PASS] listing_10_01::other_tests::exploration (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
[PASS] listing_10_01::tests::it_works (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

`scarb test` 编译并运行了测试。我们看到行 `Collected 1 test(s) from adder package` 后面是行 `Running 1 test(s) from src/`。下一行显示了称为 `it_works` 的测试函数的名称，并且该测试的运行结果是 `ok`。测试运行器还提供了 Gas 消耗的估算。总体摘要显示所有测试都已通过，`1 passed; 0 failed` 部分统计了通过或失败的测试数量。

可以将测试标记为忽略，以便它不会在特定实例中运行；我们将在本章稍后的 [除非特别请求，否则忽略某些测试](#ignoring-some-tests-unless-specifically-requested) 部分中介绍这一点。因为我们在这里没有这样做，所以摘要显示 `0 ignored`。我们还可以向 `scarb test` 命令传递参数，以仅运行名称与字符串匹配的测试；这称为过滤，我们将在 [运行单个测试](#running-single-tests) 部分中介绍这一点。由于我们没有过滤正在运行的测试，因此摘要末尾显示 `0 filtered out`。

让我们开始根据我们自己的需求自定义测试。首先将 `it_works` 函数的名称更改为不同的名称，例如 `exploration`，如下所示：

```cairo, noplayground
    #[test]
    fn exploration() {
        let result = 2 + 2;
        assert_eq!(result, 4);
    }
```

然后再次运行 `scarb test`。现在的输出显示 `exploration` 而不是 `it_works`：

```shell
$ scarb test 
     Running test listing_10_01 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_01_unittest) listing_10_01 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_01/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from listing_10_01 package
Running 2 test(s) from src/
[PASS] listing_10_01::other_tests::exploration (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
[PASS] listing_10_01::tests::it_works (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

现在我们将添加另一个测试，但这次我们将制作一个失败的测试！当测试函数中的某些内容 panic 时，测试失败。每个测试都在新线程中运行，当主线程看到测试线程已死亡时，测试被标记为失败。输入名为 `another` 的新测试作为函数，以便你的 _src/lib.cairo_ 文件看起来像清单 {{#ref second-test}} 中那样。

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
#[cfg(test)]
mod tests {
    #[test]
    fn exploration() {
        let result = 2 + 2;
        assert_eq!(result, 4);
    }

    #[test]
    fn another() {
        let result = 2 + 2;
        assert!(result == 6, "Make this test fail");
    }
}
```

{{#label second-test}} <span class="caption">清单 {{#ref second-test}}:
在 _lib.cairo_ 中添加第二个将失败的测试</span>

运行 `scarb test`，你将看到以下输出：

```shell
Collected 2 test(s) from adder package
Running 2 test(s) from src/
[FAIL] adder::tests::another

Failure data:
    "Make this test fail"

[PASS] adder::tests::exploration (gas: ~1)
Tests: 1 passed, 1 failed, 0 skipped, 0 ignored, 0 filtered out

Failures:
    adder::tests::another
```

`adder::tests::another` 行显示的不是 `[PASS]`，而是 `[FAIL]`。一个新的部分出现在单个结果和摘要之间。它显示了每个测试失败的详细原因。在这种情况下，我们得到的细节是 `another` 失败了，因为它使用 `"Make this test fail"` 错误 panic 了。

之后，显示摘要行：我们有一个测试通过，一个测试失败。最后，我们看到失败测试的列表。

现在你已经看到了测试结果在不同场景下的样子，让我们看看一些在测试中有用的函数。

[structs]: ./ch05-01-defining-and-instantiating-structs.md
[test organization]: ./ch10-02-test-organization.md

## 使用 `assert!` 宏检查结果

Cairo 提供的 `assert!` 宏在你想要确保测试中的某些条件评估为 `true` 时很有用。我们给 `assert!` 宏的第一个参数评估为布尔值。如果值为 `true`，则不发生任何事情，测试通过。如果值为 `false`，则 `assert!` 宏调用 `panic()` 导致测试失败，并带有我们定义为第二个参数的消息。使用 `assert!` 宏有助于我们检查代码是否按我们预期的方式运行。

还记得在 [第 {{#chap using-structs-to-structure-related-data}} 章][method syntax] 中，我们使用了 `Rectangle` 结构体和 `can_hold` 方法，这里在清单 {{#ref rectangle}} 中重复。让我们将此代码放在 _src/lib.cairo_ 文件中，然后使用 `assert!` 宏为它编写一些测试。

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
#[derive(Drop)]
struct Rectangle {
    width: u64,
    height: u64,
}

trait RectangleTrait {
    fn can_hold(self: @Rectangle, other: @Rectangle) -> bool;
}

impl RectangleImpl of RectangleTrait {
    fn can_hold(self: @Rectangle, other: @Rectangle) -> bool {
        *self.width > *other.width && *self.height > *other.height
    }
}
```

{{#label rectangle}} <span class="caption">清单 {{#ref rectangle}}: 使用 [第 {{#chap using-structs-to-structure-related-data}} 章][structs] 中的 `Rectangle` 结构体及其 `can_hold` 方法</span>

`can_hold` 方法返回一个 `bool`，这意味着它是 `assert!` 宏的完美用例。我们可以通过创建一个宽度为 `8` 高度为 `7` 的 `Rectangle` 实例，并断言它可以容纳另一个宽度为 `5` 高度为 `1` 的 `Rectangle` 实例来编写一个测试来练习 `can_hold` 方法。

```cairo, noplayground
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle { height: 7, width: 8 };
        let smaller = Rectangle { height: 1, width: 5 };

        assert!(larger.can_hold(@smaller), "rectangle cannot hold");
    }
}
```

注意 `tests` 模块内的 `use super::*;` 行。`tests` 模块是一个常规模块，它遵循我们在第 {{#chap paths-for-referring-to-an-item-in-the-module-tree}} 章的 [“引用模块树中项目的路径”][paths-for-referring-to-an-item-in-the-module-tree]<!-- ignore --> 部分中介绍的常规可见性规则。因为 `tests` 模块是一个内部模块，我们需要将外部模块中的待测试代码引入内部模块的范围。我们在这里使用了一个 glob，所以我们在外部模块中定义的任何东西都对此 `tests` 模块可用。

如果你没有将测试模块命名为 `tests`，你可能不需要 `use super::*;`，但这取决于你的模块结构。

我们将测试命名为 `larger_can_hold_smaller`，并创建了我们需要的两个 `Rectangle` 实例。然后我们调用 `assert!` 宏并传递调用 `larger.can_hold(@smaller)` 的结果。此表达式应该返回 `true`，所以我们的测试应该通过。让我们来看看！

```shell
$ scarb test 
     Running test listing_10_03 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_03_unittest) listing_10_03 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_03/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from listing_10_03 package
Running 2 test(s) from src/
[PASS] listing_10_03::tests2::smaller_cannot_hold_larger (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
[PASS] listing_10_03::tests::larger_can_hold_smaller (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

它确实通过了！让我们添加另一个测试，这次断言一个较小的矩形不能容纳一个较大的矩形：

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
    #[test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle { height: 7, width: 8 };
        let smaller = Rectangle { height: 1, width: 5 };

        assert!(!smaller.can_hold(@larger), "rectangle cannot hold");
    }
```

{{#label another-test}} <span class="caption">清单 {{#ref another-test}}:
在 _lib.cairo_ 中添加另一个将通过的测试</span>

因为在这种情况下 `can_hold` 方法的正确结果是 `false`，我们需要在将其传递给 `assert!` 宏之前否定该结果。因此，如果 `can_hold` 返回 `false`，我们的测试将通过：

```shell
$ scarb test 
     Running test listing_10_03 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_03_unittest) listing_10_03 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_03/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from listing_10_03 package
Running 2 test(s) from src/
[PASS] listing_10_03::tests2::smaller_cannot_hold_larger (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
[PASS] listing_10_03::tests::larger_can_hold_smaller (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

两个测试都通过了！现在让我们看看当我们在代码中引入错误时，我们的测试记过会发生什么。我们将通过在比较宽度时将 `>` 符号替换为 `<` 符号来更改 `can_hold` 方法的实现：

```cairo, noplayground
impl RectangleImpl of RectangleTrait {
    fn can_hold(self: @Rectangle, other: @Rectangle) -> bool {
        *self.width < *other.width && *self.height > *other.height
    }
}
```

现在运行测试会产生以下结果：

```shell
$ scarb test 
     Running test no_listing_01_wrong_can_hold_impl (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(no_listing_01_wrong_can_hold_impl_unittest) no_listing_01_wrong_can_hold_impl v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_01_wrong_can_hold_impl/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from no_listing_01_wrong_can_hold_impl package
Running 2 test(s) from src/
[FAIL] no_listing_01_wrong_can_hold_impl::tests::larger_can_hold_smaller

Failure data:
    "rectangle cannot hold"

[PASS] no_listing_01_wrong_can_hold_impl::tests2::smaller_cannot_hold_larger (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 1 passed, 1 failed, 0 ignored, 0 filtered out

Failures:
    no_listing_01_wrong_can_hold_impl::tests::larger_can_hold_smaller

```

我们的测试捕获了错误！因为 `larger.width` 是 `8` 且 `smaller.width` 是 `5`，在 `larger_can_hold_smaller` 测试中，`can_hold` 中的宽度比较现在返回 `false`（`8` 不小于 `5`）。注意 `smaller_cannot_hold_larger` 测试仍然通过：要使此测试失败，高度比较也应该在 `can_hold` 方法中修改，将 `>` 符号替换为 `<` 符号。

[method syntax]: ./ch05-03-method-syntax.md

## 使用 `assert_xx!` 宏测试相等性和比较

### `assert_eq!` 和 `assert_ne!` 宏

验证功能的常用方法是测试被测代码的结果与你期望代码返回的值之间的相等性。你可以使用 `assert!` 宏并传递一个使用 `==` 运算符的表达式来做到这一点。然而，这是一个如此常见的测试，以至于标准库提供了一对宏 —— `assert_eq!` 和 `assert_ne!` —— 以更方便地执行此测试。这些宏分别比较两个参数的相等性或不相等性。如果断言失败，它们还会打印这两个值，这使得更容易看出 _为什么_ 测试失败；相反，`assert!` 宏仅指示它得到了 `==` 表达式的 `false` 值，而不打印导致 `false` 值的值。

在清单 {{#ref add_two}} 中，我们编写了一个名为 `add_two` 的函数，它将 `2` 添加到其参数，然后我们使用 `assert_eq!` 和 `assert_ne!` 宏测试此函数。

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
pub fn add_two(a: u32) -> u32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        assert_eq!(4, add_two(2));
    }

    #[test]
    fn wrong_check() {
        assert_ne!(0, add_two(2));
    }
}
```

{{#label add_two}} <span class="caption">清单 {{#ref add_two}}: 使用 `assert_eq!` 和 `assert_ne!` 宏测试函数 `add_two`</span>

让我们检查它是否通过！

```shell
$ scarb test 
     Running test listing_10_04 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_04_unittest) listing_10_04 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_04/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from listing_10_04 package
Running 2 test(s) from src/
[PASS] listing_10_04::tests::wrong_check (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
[PASS] listing_10_04::tests::it_adds_two (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

在 `it_adds_two` 测试中，我们将 `4` 作为参数传递给 `assert_eq!` 宏，它等于调用 `add_two(2)` 的结果。此测试的行是 `[PASS] adder::tests::it_adds_two (gas: ~1)`。

在 `wrong_check` 测试中，我们将 `0` 作为参数传递给 `assert_ne!` 宏，它不等于调用 `add_two(2)` 的结果。如果我们给出的两个值 _不_ 相等，使用 `assert_ne!` 宏的测试将通过，如果它们相等则失败。当我们不确定值 _将_ 是什么，但我们知道值绝对 _不应该_ 是什么时，此宏最有用。例如，如果我们正在测试一个保证以某种方式更改其输入的函数，但输入的更改方式取决于我们运行测试的星期几，那么最好的断言可能是函数的输出不等于输入。

让我们在代码中引入一个错误，看看 `assert_eq!` 在失败时是什么样子的。将 `add_two` 函数的实现更改为加 `3`：

```cairo, noplayground
pub fn add_two(a: u32) -> u32 {
    a + 3
}
```

再次运行测试：

```shell
$ scarb test 
     Running test listing_10_04_wong_add (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_04_wong_add_unittest) listing_10_04_wong_add v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_04_wong_add/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 1 test(s) from listing_10_04_wong_add package
Running 1 test(s) from src/
[FAIL] listing_10_04_wong_add::tests::it_adds_two

Failure data:
    "assertion `4 == add_two(2)` failed.
    4: 4
    add_two(2): 5"

Tests: 0 passed, 1 failed, 0 ignored, 0 filtered out

Failures:
    listing_10_04_wong_add::tests::it_adds_two

```

我们的测试捕获了错误！`it_adds_two` 测试失败，并显示以下消息：``"assertion `4 == add_two(2)` failed``。它告诉我们失败的断言是 `` "assertion `left == right` failed``，并且 `left` 和 `right` 值打印在下一行，为 `left: left_value` 和 `right: right_value`。这有助于我们开始调试：`left` 参数是 `4`，但 `right` 参数（我们有 `add_two(2)` 的地方）是 `5`。你可以想象，当我们进行大量测试时，这将特别有帮助。

注意，在某些语言和测试框架中，相等断言函数的参数称为 `expected` 和 `actual`，并且我们指定参数的顺序很重要。然而，在 Cairo 中，它们被称为 `left` 和 `right`，并且我们指定我们期望的值和代码产生的值的顺序并不重要。我们可以将此测试中的断言写为 `assert_eq!(add_two(2), 4)`，这将导致显示相同的失败消息 `` assertion failed: `(left == right)` ``。

这是一个比较两个结构体的简单示例，展示了如何使用 `assert_eq!` 和 `assert_ne!` 宏：

```cairo, noplayground
#[derive(Drop, Debug, PartialEq)]
struct MyStruct {
    var1: u8,
    var2: u8,
}

#[cfg(test)]
#[test]
fn test_struct_equality() {
    let first = MyStruct { var1: 1, var2: 2 };
    let second = MyStruct { var1: 1, var2: 2 };
    let third = MyStruct { var1: 1, var2: 3 };

    assert_eq!(first, second);
    assert_eq!(first, second, "{:?},{:?} should be equal", first, second);
    assert_ne!(first, third);
    assert_ne!(first, third, "{:?},{:?} should not be equal", first, third);
}
```

在表面之下，`assert_eq!` 和 `assert_ne!` 宏分别使用运算符 `==` 和 `!=`。它们都接受值的快照作为参数。当断言失败时，这些宏使用调试格式化（`{:?}` 语法）打印它们的参数，这意味着被比较的值必须实现 `PartialEq` 和 `Debug` traits。所有原始类型和大多数核心库类型都实现了这些 traits。对于你自己定义的结构体和枚举，你需要实现 `PartialEq` 来断言那些类型的相等性。你还需要实现 `Debug` 以在断言失败时打印值。因为这两个 traits 都是可派生的，所以这通常就像将 `#[derive(Drop, Debug, PartialEq)]` 注释添加到你的结构体或枚举定义一样简单。有关这些和其他可派生 traits 的更多详细信息，请参阅 [附录 C][derivable traits]。

[derivable traits]: ./appendix-03-derivable-traits.md

### `assert_lt!`, `assert_le!`, `assert_gt!` 和 `assert_ge!` 宏

测试中的比较可以使用 `assert_xx!` 宏完成：

- `assert_lt!` 检查给定值是否小于另一个值，否则回滚。
- `assert_le!` 检查给定值是否小于或等于另一个值，否则回滚。
- `assert_gt!` 检查给定值是否大于另一个值，否则回滚。
- `assert_ge!` 检查给定值是否大于或等于另一个值，否则回滚。

清单 {{#ref assert_macros}} 演示了如何使用这些宏：

```cairo, noplayground
#[derive(Drop, Copy, Debug, PartialEq)]
struct Dice {
    number: u8,
}

impl DicePartialOrd of PartialOrd<Dice> {
    fn lt(lhs: Dice, rhs: Dice) -> bool {
        lhs.number < rhs.number
    }

    fn le(lhs: Dice, rhs: Dice) -> bool {
        lhs.number <= rhs.number
    }

    fn gt(lhs: Dice, rhs: Dice) -> bool {
        lhs.number > rhs.number
    }

    fn ge(lhs: Dice, rhs: Dice) -> bool {
        lhs.number >= rhs.number
    }
}

#[cfg(test)]
#[test]
fn test_struct_equality() {
    let first_throw = Dice { number: 5 };
    let second_throw = Dice { number: 2 };
    let third_throw = Dice { number: 6 };
    let fourth_throw = Dice { number: 5 };

    assert_gt!(first_throw, second_throw);
    assert_ge!(first_throw, fourth_throw);
    assert_lt!(second_throw, third_throw);
    assert_le!(
        first_throw, fourth_throw, "{:?},{:?} should be lower or equal", first_throw, fourth_throw,
    );
}
```

{{#label assert_macros}} <span class="caption">清单 {{#ref assert_macros}}:
使用 `assert_xx!` 宏进行比较的测试示例</span>

在这个例子中，我们多次掷 `Dice` 结构体并比较结果。我们需要为我们的结构体手动实现 `PartialOrd` trait，以便我们可以使用 `lt`, `le`, `gt` 和 `ge` 函数比较 `Dice` 实例，这些函数分别被 `assert_lt!`, `assert_le!`, `assert_gt!` 和 `assert_ge!` 宏使用。我们还需要在我们的 `Dice` 结构体上派生 `Copy` trait，以便多次使用实例化的结构体，因为比较函数会获取变量的所有权。

## 添加自定义失败消息

你还可以作为 `assert!`, `assert_eq!` 和 `assert_ne!` 宏的可选参数，添加要与失败消息一起打印的自定义消息。在所需参数之后指定的任何参数都会传递给 `format!` 宏（在 [打印][formatting] 一章中讨论），因此你可以传递一个包含 `{}` 占位符的格式字符串以及要放入这些占位符的值。自定义消息对于记录断言的含义很有用；当测试失败时，你将更好地了解代码的问题所在。

让我们添加一个自定义失败消息，该消息由一个格式字符串组成，其中包含一个占位符，该占位符填充了我们从上一个 `add_two` 函数获得的实际值：

```cairo, noplayground
    #[test]
    fn it_adds_two() {
        assert_eq!(4, add_two(2), "Expected {}, got add_two(2)={}", 4, add_two(2));
    }
```

现在当我们运行测试时，我们将得到更具信息量的错误消息：

```shell
$ scarb test 
     Running test no_listing_02_custom_messages (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(no_listing_02_custom_messages_unittest) no_listing_02_custom_messages v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_02_custom_messages/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 1 test(s) from no_listing_02_custom_messages package
Running 1 test(s) from src/
[FAIL] no_listing_02_custom_messages::tests::it_adds_two

Failure data:
    "assertion `4 == add_two(2)` failed: Expected 4, got add_two(2)=5
    4: 4
    add_two(2): 5"

Tests: 0 passed, 1 failed, 0 ignored, 0 filtered out

Failures:
    no_listing_02_custom_messages::tests::it_adds_two

```

我们可以在测试输出中看到我们实际得到的值，这将有助于我们调试发生的事情，而不是我们期望发生的事情。

[formatting]: ./ch12-08-printing.md#formatting

## 使用 `should_panic` 检查 panics

除了检查返回值外，检查我们的代码是否按预期处理错误条件也很重要。例如，考虑清单 {{#ref guess}} 中的 `Guess` 类型：

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
#[derive(Drop)]
struct Guess {
    value: u64,
}

pub trait GuessTrait {
    fn new(value: u64) -> Guess;
}

impl GuessImpl of GuessTrait {
    fn new(value: u64) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess must be >= 1 and <= 100");
        }

        Guess { value }
    }
}
```

{{#label guess}} <span class="caption">清单 {{#ref guess}}: `Guess` 结构体及其 `new` 方法</span>

其他使用 `Guess` 的代码依赖于 `Guess` 实例将仅包含 `1` 和 `100` 之间的值这一保证。我们可以编写一个测试，通过调用 `Guess::new` 来确保尝试创建一个值超出该范围的 `Guess` 实例会 panic。

我们通过向我们的测试函数添加属性 `should_panic` 来做到这一点。如果函数内的代码 panic，则测试通过；如果函数内的代码不 panic，则测试失败。

```cairo, noplayground
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        GuessTrait::new(200);
    }
}
```

我们将 `#[should_panic]` 属性放在 `#[test]` 属性之后和它应用的测试函数之前。让我们看看结果，看看这个测试是否通过：

```shell
$ scarb test 
     Running test listing_09_08 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_09_08_unittest) listing_09_08 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_05/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 1 test(s) from listing_09_08 package
Running 1 test(s) from src/
[PASS] listing_09_08::tests::greater_than_100 (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 1 passed, 0 failed, 0 ignored, 0 filtered out


```

看起来不错！现在让我们通过删除 `new` 函数在值大于 `100` 时会 panic 的条件，在我们的代码中引入一个错误：

```cairo, noplayground
impl GuessImpl of GuessTrait {
    fn new(value: u64) -> Guess {
        if value < 1 {
            panic!("Guess must be >= 1 and <= 100");
        }

        Guess { value }
    }
}
```

当我们运行测试时，它将失败：

```shell
$ scarb test 
     Running test no_listing_03_wrong_new_impl (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(no_listing_03_wrong_new_impl_unittest) no_listing_03_wrong_new_impl v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_03_wrong_new_impl/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 1 test(s) from no_listing_03_wrong_new_impl package
Running 1 test(s) from src/
[FAIL] no_listing_03_wrong_new_impl::tests::greater_than_100

Failure data:
    Expected to panic, but no panic occurred

Tests: 0 passed, 1 failed, 0 ignored, 0 filtered out

Failures:
    no_listing_03_wrong_new_impl::tests::greater_than_100

```

在这种情况下，我们没有得到非常有用的消息，但是当我们查看测试函数时，我们看到它用 `#[should_panic]` 属性进行了注释。我们得到的失败意味着测试函数中的代码没有导致 panic。

使用 `should_panic` 的测试可能不精确。如果测试由于与我们预期的原因不同的原因而 panic，`should_panic` 测试也会通过。为了使 `should_panic` 测试更精确，我们可以向 `#[should_panic]` 属性添加一个可选的 `expected` 参数。测试工具将确保失败消息包含提供的文本。例如，考虑清单 {{#ref guess-2}} 中 `GuessImpl` 的修改后的代码，其中 `new` 函数根据值是太小还是太大而使用不同的消息 panic：

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
impl GuessImpl of GuessTrait {
    fn new(value: u64) -> Guess {
        if value < 1 {
            panic!("Guess must be >= 1");
        } else if value > 100 {
            panic!("Guess must be <= 100");
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected: "Guess must be <= 100")]
    fn greater_than_100() {
        GuessTrait::new(200);
    }
}
```

{{#label guess-2}} <span class="caption">清单 {{#ref guess-2}}: `new`
使用不同错误消息 panic 的实现</span>

测试将通过，因为我们在 `should_panic` 属性的 `expected` 参数中放置的值是 `Guess::new` 方法 panic 所用的字符串。我们需要指定我们期望的整个 panic 消息。

为了看看当带有预期消息的 `should_panic` 测试失败时会发生什么，让我们再次通过交换 `if value < 1` 和 `else if value > 100` 块的主体来在我们的代码中引入错误：

```cairo, noplayground
impl GuessImpl of GuessTrait {
    fn new(value: u64) -> Guess {
        if value < 1 {
            panic!("Guess must be <= 100");
        } else if value > 100 {
            panic!("Guess must be >= 1");
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected: "Guess must be <= 100")]
    fn greater_than_100() {
        GuessTrait::new(200);
    }
}
```

这次当我们运行 `should_panic` 测试时，它将失败：

```shell
$ scarb test 
     Running test no_listing_04_new_bug (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(no_listing_04_new_bug_unittest) no_listing_04_new_bug v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_04_new_bug/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 1 test(s) from no_listing_04_new_bug package
Running 1 test(s) from src/
[FAIL] no_listing_04_new_bug::tests::greater_than_100

Failure data:
    Incorrect panic data
    Actual:    [0x46a6158a16a947e5916b2a2ca68501a45e93d7110e81aa2d6438b1c57c879a3, 0x0, 0x4775657373206d757374206265203e3d2031, 0x12] (Guess must be >= 1)
    Expected:  [0x46a6158a16a947e5916b2a2ca68501a45e93d7110e81aa2d6438b1c57c879a3, 0x0, 0x4775657373206d757374206265203c3d20313030, 0x14] (Guess must be <= 100)

Tests: 0 passed, 1 failed, 0 ignored, 0 filtered out

Failures:
    no_listing_04_new_bug::tests::greater_than_100

```

失败消息表明此测试确实如我们预期的那样 panic 了，但 panic 消息不包含预期的字符串。我们在这种情况下得到的 panic 消息是 `Guess must be >= 1`。现在我们可以开始找出我们的错误在哪里了！

## 运行单个测试

有时，运行完整的测试套件可能需要很长时间。如果你正在处理特定区域的代码，你可能只想运行属于该代码的测试。你可以通过将你想运行的测试的名称作为参数传递给 `scarb test` 来选择要运行的测试。

为了演示如何运行单个测试，我们将首先创建两个测试函数，如清单 {{#ref two-tests}} 所示，并选择要运行的测试。

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
#[cfg(test)]
mod tests {
    #[test]
    fn add_two_and_two() {
        let result = 2 + 2;
        assert_eq!(result, 4);
    }

    #[test]
    fn add_three_and_two() {
        let result = 3 + 2;
        assert!(result == 5, "result is not 5");
    }
}
```

{{#label two-tests}} <span class="caption">清单 {{#ref two-tests}}: 两个具有不同名称的测试</span>

我们可以将任何测试函数的名称传递给 `scarb test` 以仅运行该测试：

```shell
$ scarb test add_two_and_two
     Running test listing_10_07 (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_10_07_unittest) listing_10_07 v0.1.0 (listings/ch10-testing-cairo-programs/listing_10_07/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 1 test(s) from listing_10_07 package
Running 1 test(s) from src/
[PASS] listing_10_07::tests::add_two_and_two (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 1 passed, 0 failed, 0 ignored, 1 filtered out


```

只有名为 `add_two_and_two` 的测试运行了；另一个测试不匹配该名称。测试输出让我们知道我们还有一个未运行的测试，在末尾显示 `1 filtered out;`。

我们还可以指定测试名称的一部分，任何名称包含该值的测试都将运行。

## 除非特别请求，否则忽略某些测试

有时一些特定的测试执行起来非常耗时，因此你可能希望在大多数 `scarb test` 运行期间排除它们。除了将所有你想要运行的测试列为参数外，你还可以使用 `#[ignore]` 属性注释耗时的测试以排除它们，如下所示：

```cairo, noplayground
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }

    #[test]
    #[ignore]
    fn expensive_test() { // code that takes an hour to run
    }
}
```

在 `#[test]` 之后，我们将 `#[ignore]` 行添加到我们想要排除的测试中。现在当我们运行测试时，`it_works` 运行，但 `expensive_test` 不运行：

```shell
$ scarb test 
     Running test no_listing_05_ignore_tests (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(no_listing_05_ignore_tests_unittest) no_listing_05_ignore_tests v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_05_ignore_tests/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from no_listing_05_ignore_tests package
Running 2 test(s) from src/
[IGNORE] no_listing_05_ignore_tests::tests::expensive_test
[PASS] no_listing_05_ignore_tests::tests::it_works (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 1 passed, 0 failed, 1 ignored, 0 filtered out


```

`expensive_test` 函数被列为已忽略。

当你只想检查被忽略测试的结果并且有时间等待结果时，你可以运行 `scarb test --include-ignored` 来运行所有测试，无论它们是否被忽略。

## 测试递归函数或循环

在测试递归函数或循环时，测试默认使用它可以消耗的最大 Gas 量实例化。这可以防止运行无限循环或消耗过多的 Gas，并可以帮助你对实现的效率进行基准测试。假设这个值足够大，但你可以通过向测试函数添加 `#[available_gas(<Number>)]` 属性来覆盖它。以下示例显示了如何使用它：

```cairo, noplayground
fn sum_n(n: usize) -> usize {
    let mut i = 0;
    let mut sum = 0;
    while i <= n {
        sum += i;
        i += 1;
    }
    sum
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[available_gas(l2_gas: 2000000)]
    fn test_sum_n() {
        let result = sum_n(10);
        assert!(result == 55, "result is not 55");
    }
}
```

## 对 Cairo 程序进行基准测试

Starknet Foundry 包含一个性能分析功能，可用于分析和优化 Cairo 程序的性能。

[profiling][profiling] 功能为成功的测试生成执行跟踪，用于创建配置文件输出。这允许你对代码的特定部分进行基准测试。

要使用分析器，你需要：

1. 从 Software Mansion 安装 [Cairo Profiler][cairo profiler]。
2. 安装 [Go][go], [Graphviz][graphviz] 和 [pprof][pprof]，它们都需要用来可视化生成的配置文件输出。
3. 运行 `snforge test --build-profile` 命令，该命令为每个通过的测试生成一个跟踪文件，存储在你项目的 _snfoundry_trace_ 目录中。此命令还在 _profile_ 目录中生成相应的输出文件。
4. 运行 `go tool pprof -http=":8000" path/to/profile/output.pb.gz` 来分析配置文件。这将在指定端口启动 Web 服务器。

让我们重用上面研究的 `sum_n` 函数：

```cairo, noplayground
fn sum_n(n: usize) -> usize {
    let mut i = 0;
    let mut sum = 0;
    while i <= n {
        sum += i;
        i += 1;
    }
    sum
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[available_gas(l2_gas: 2000000)]
    fn test_sum_n() {
        let result = sum_n(10);
        assert!(result == 55, "result is not 55");
    }
}
```

在生成跟踪文件和配置文件输出后，在你的项目中运行 `go tool pprof` 将启动 Web 服务器，你可以在其中找到有关你运行的测试的许多有用信息：

- 测试包括一个函数调用，对应于对测试函数的调用。在测试函数中多次调用 `sum_n` 仍将返回 1 次调用。这是因为 `snforge` 在执行测试时模拟合约调用。

- `sum_n` 函数执行使用 256 个 Cairo 步骤：

<div align="center">
    <img src="pprof-steps.png" alt="pprof number of steps" width="800px"/>
</div>

其他信息也可用，例如内存空洞（即未使用的内存单元）或内置函数的使用。Cairo Profiler 正在积极开发中，未来将提供许多其他功能。

[hello world]: ./ch01-02-hello-world.md#creating-a-project-with-scarb
[profiling]:
  https://foundry-rs.github.io/starknet-foundry/snforge-advanced-features/profiling.html
[cairo profiler]: https://github.com/software-mansion/cairo-profiler
[go]: https://go.dev/doc/install
[Graphviz]: https://www.graphviz.org/download/
[pprof]: https://github.com/google/pprof?tab=readme-ov-file#building-pprof
[paths-for-referring-to-an-item-in-the-module-tree]:
  ./ch07-03-paths-for-referring-to-an-item-in-the-module-tree.md

{{#quiz ../quizzes/ch10-01-how_to_write_tests.toml}}
