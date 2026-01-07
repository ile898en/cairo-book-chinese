# 宏 (Macros)

我们在本书中一直在使用像 `println!` 这样的宏，但我们还没有完全探索宏是什么以及它是如何工作的。术语 _宏 (macro)_ 指的是 Cairo 中的一组特性：使用 `macro` 的 _声明式 (declarative)_ 宏，以及在 [过程宏](./ch12-10-procedural-macros.md) 中介绍的三种 _过程 (procedural)_ 宏：

- 自定义 `#[derive]` 宏，指定在结构体和枚举上使用 `derive` 属性时添加的代码
- 属性类 (Attribute-like) 宏，定义可用于任何项的自定义属性
- 函数类 (Function-like) 宏，看起来像函数调用，但对作为其参数指定的标记 (tokens) 进行操作

我们将依次讨论每一个，但首先，让我们看看当我们已经有了函数时为什么还需要宏。

## 宏和函数之间的区别

从根本上说，宏是一种编写其他代码的代码的方式，这被称为 _元编程 (metaprogramming)_。在附录 C 中，我们讨论了可派生 traits 和 `derive` 属性，它为你生成各种 traits 的实现。我们在整本书中还使用了 `println!` 和 `array!` 宏。所有这些宏都会 _展开_ 以生成比你手动编写的代码更多的代码。

元编程对于减少你需要编写和维护的代码量很有用，这也是函数的作用之一。然而，宏具有一些函数所没有的额外能力。

函数签名必须声明函数具有的参数的数量和类型。另一方面，宏可以接受可变数量的参数：我们可以用一个参数调用 `println!("hello")`，或者用两个参数调用 `println!("hello {}", name)`。此外，宏在编译器解释代码含义之前展开，因此宏可以，例如，在给定类型上实现 trait。函数不能，因为它在运行时被调用，而 trait 需要在编译时实现。

实现宏而不是函数的缺点是，宏定义比函数定义更复杂，因为你正在编写 Cairo 代码 —— 或者更复杂的 Rust 代码 —— 来编写 Cairo 代码。由于这种间接性，宏定义通常比函数定义更难阅读、理解和维护。

宏和函数之间的另一个重要区别是，你必须在文件中调用宏 _之前_ 定义宏或将它们引入作用域，而函数可以在任何地方定义并在任何地方调用。

## 用于通用元编程的声明式内联宏

Cairo 中最简单的宏形式是 _声明式宏 (declarative macro)_，有时也简称为“宏”。在其核心，声明式宏允许你编写类似于 `match` 表达式的东西。如 [第 6 章](./ch06-00-enums-and-pattern-matching.md) 所述，`match` 表达式是控制结构，它接受一个表达式，将表达式的结果值与模式进行比较，然后运行与匹配模式关联的代码。宏也将值与关联特定代码的模式进行比较：在这种情况下，值是传递给宏的文字 Cairo 源代码；模式与该源代码的结构进行比较；当匹配时，与每个模式关联的代码替换传递给宏的代码。这一切都发生在编译期间。

要定义宏，你使用 `macro` 构造。让我们通过查看“数组构建”宏的工作原理来探索这种风格。之前，我们使用 Cairo 内置的 `array!` 宏来创建具有特定值的数组。例如，以下代码创建一个包含三个整数的新数组：

```cairo
let a = array![1, 2, 3];
```

我们也可以使用 `array!` 制作两个整数的数组或五个其他类型值的数组，因为宏可以接受可变数量的参数。我们无法使用常规函数做同样的事情，因为我们无法预先知道值的数量或类型。

下面是用 Cairo 编写的数组构建宏的稍微简化的定义。它不是核心库中的确切 `array!` 宏，但它展示了使用声明式内联宏的相同核心思想：

```cairo
macro make_array {
    ($($x:expr), *) => {
        {
            let mut arr = $defsite::ArrayTrait::new();
            $(arr.append($x);)*
            arr
        }
    };
}
```

> 注意：标准库中的内置 `array!` 宏可能包含优化（如预留容量），我们为了保持示例简单而没有包含这些优化。

宏主体的结构类似于 `match` 表达式。这里我们有一个带有模式 `($($x:expr), *)` 的分支，后面跟着 `=>` 和与此模式关联的代码块。如果模式匹配，则发出关联的代码块。更复杂的宏可以有多个分支，每个分支都有不同的模式。

宏定义中的模式语法与匹配值时使用的模式语法不同：宏模式与 Cairo 源代码结构进行匹配。让我们以此为例，逐步了解模式的各个部分：

- 我们使用括号来包含整个匹配器模式。
- 美元符号 (`$`) 引入一个宏变量，它将捕获与子模式匹配的代码。在 `$()` 内是 `$x:expr`，它匹配任何 Cairo 表达式并将该表达式命名为 `$x`。
- 跟随 `$()` 的逗号要求每个匹配表达式之间有文字逗号。
- `*` 量词指定子模式可以重复零次或多次。

当我们用 `make_array![1, 2, 3]` 调用此宏时，`$x` 模式匹配三次：表达式 `1`、`2` 和 `3`。

现在看展开方面：`$(arr.append($x);)*` 对模式中 `$()` 的每次匹配生成一次。`$x` 被替换为每个匹配的表达式。调用 `make_array![1, 2, 3]` 展开为如下代码：

> 注意：VSCode 扩展可以通过按 `Ctrl+Shift+P` 然后选择 `Cairo: Recursively expand macros for item at caret` 来帮助你检查展开的代码。

```cairo,ignore
{
    let mut arr = ArrayTrait::new();
    arr.append(1);
    arr.append(2);
    arr.append(3);
    arr
}
```

我们定义了一个宏，它可以接受任意数量、任意类型的参数，并生成代码来创建包含指定元素的数组。

用法如下所示：

```cairo
    let a = make_array![1, 2, 3];
```

要使用它们，请在你的 `Scarb.toml` 中启用实验性功能：

```toml
<!-- Warning: Anchor 'feature_flag' not found in Scarb.toml -->
```

内联宏是用 `macro name { ... }` 定义的，其中每个分支匹配一个代码模式并展开为替换代码。像 Rust 的 macros-by-example 一样，你可以用 `$var: kind` 捕获语法片段，并用 `$()*`、`$()+` 或 `$()?` 重复匹配。

### 卫生 (Hygiene)、`$defsite`/`$callsite` 和 `expose!`

Cairo 的内联宏是卫生的：宏定义中引入的名称不会泄漏到调用点，除非你显式地暴露它们。宏内的名称解析可以使用 `$defsite::` 和 `$callsite::` 引用宏定义点或调用点。

注意，与 Rust 类似，宏预期展开为单个表达式；因此，如果你的宏定义了多个语句，你应该将它们包装在一个额外的 `{}` 块中，该块返回最终表达式。

以下端到端示例说明了所有这些方面：

```cairo
mod hygiene_demo {
    // A helper available at the macro definition site
    fn def_bonus() -> u8 {
        10
    }

    // Adds the defsite bonus, regardless of what exists at the callsite
    pub macro add_defsite_bonus {
        ($x: expr) => { $x + $defsite::def_bonus() };
    }

    // Adds the callsite bonus, resolved where the macro is invoked
    pub macro add_callsite_bonus {
        ($x: expr) => { $x + $callsite::bonus() };
    }

    // Exposes a variable to the callsite using `expose!`.
    pub macro apply_and_expose_total {
        ($base: expr) => {
            let total = $base + 1;
            expose!(let exposed_total = total;);
        };
    }

    // A helper macro that reads a callsite-exposed variable
    pub macro read_exposed_total {
        () => { $callsite::exposed_total };
    }

    // Wraps apply_and_expose_total and then uses another inline macro
    // that accesses the exposed variable via `$callsite::...`.
    pub macro wrapper_uses_exposed {
        ($x: expr) => {
            {
                $defsite::apply_and_expose_total!($x);
                $defsite::read_exposed_total!()
            }
        };
    }
}
```

调用点的用法：

```cairo

    // Callsite defines its own `bonus` — used only by callsite-resolving macro
    let bonus = | | -> u8 {
        20
    };
    let price: u8 = 5;
    assert_eq!(add_defsite_bonus!(price), 15); // uses defsite::def_bonus() = 10
    assert_eq!(add_callsite_bonus!(price), 25); // uses callsite::bonus() = 20

    // Call in statement position; it exposes `exposed_total` at the callsite
    apply_and_expose_total!(3);
    assert_eq!(exposed_total, 4);

    // A macro invoked by another macro can access exposed values via `$callsite::...`
    let w = wrapper_uses_exposed!(7);
    assert_eq!(w, 8);
```

这演示了什么：

- `$defsite::...` 解析为宏定义旁边的项，并在调用点之间保持稳定。
- `$callsite::...` 解析为在调用宏的地方可见的项。
- 默认情况下名称不会泄漏；`expose!` 可以故意将新项引入调用点。
- 暴露的名称可以通过 `$callsite::name` 访问你的宏主体内调用的其他内联宏。

注意：

- 此功能是实验性的；语法和功能可能会发生变化。
- 尚未支持产生项的宏（结构体、枚举、函数等）；支持将在未来版本中添加。
- 对于属性、派生和 crate 范围的转换，请首选过程宏（下一节）。
