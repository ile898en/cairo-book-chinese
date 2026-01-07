# 控制流

根据条件是否为真来运行某些代码，以及在条件为真时重复运行某些代码的能力，是大多数编程语言的基本构建块。让你控制 Cairo 代码执行流程的最常见结构是 `if` 表达式和循环。

## `if` 表达式

`if` 表达式允许你根据条件分支你的代码。你提供一个条件，然后声明，“如果满足这个条件，运行这段代码块。如果不满足这个条件，不要运行这段代码块。”

在你的 _cairo_projects_ 目录中创建一个名为 _branches_ 的新项目来探索 `if` 表达式。在 _src/lib.cairo_ 文件中，输入以下内容：

```cairo
#[executable]
fn main() {
    let number = 3;

    if number == 5 {
        println!("condition was true and number = {}", number);
    } else {
        println!("condition was false and number = {}", number);
    }
}
```

所有的 `if` 表达式都以关键字 `if` 开头，后跟一个条件。在这种情况下，条件检查变量 `number` 的值是否等于 5。我们将如果条件为 `true` 时要执行的代码块放在条件之后的在大括号内。

或者，我们也可以包含一个 `else` 表达式（我们在这里选择这样做），以便在条件计算为 `false` 时给程序一个替代的代码块来执行。如果你不提供 `else` 表达式且条件为 `false`，程序将直接跳过 `if` 块并继续下一段代码。

尝试运行这段代码；你应该看到以下输出：

```shell
$ scarb execute 
   Compiling no_listing_24_if v0.1.0 (listings/ch02-common-programming-concepts/no_listing_27_if/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_24_if
condition was false and number = 3


```

让我们尝试将 `number` 的值更改为一个使条件为 `true` 的值，看看会发生什么：

```cairo, noplayground
    let number = 5;
```

```shell
$ scarb execute
condition was true and number = 5
Run completed successfully, returning []
```

同样值得注意的是，此代码中的条件必须是 `bool`。如果条件不是 `bool`，我们会得到一个错误。例如，尝试运行以下代码：

```cairo
//TAG: does_not_compile

#[executable]
fn main() {
    let number = 3;

    if number {
        println!("number was three");
    }
}
```

这次 `if` 条件计算为值 3，Cairo 抛出一个错误：

```shell
$ scarb build 
   Compiling no_listing_28_bis_if_not_bool v0.1.0 (listings/ch02-common-programming-concepts/no_listing_28_bis_if_not_bool/Scarb.toml)
error: Mismatched types. The type `core::bool` cannot be created from a numeric literal.
 --> listings/ch02-common-programming-concepts/no_listing_28_bis_if_not_bool/src/lib.cairo:5:18
    let number = 3;
                 ^

error: could not compile `no_listing_28_bis_if_not_bool` due to previous error

```

错误表明 Cairo 根据 `number` 稍后作为 `if` 语句条件的用法推断出它的类型是 `bool`。它尝试从值 `3` 创建一个 `bool`，但 Cairo 反正不支持从数字文字实例化 `bool` —— 你只能使用 `true` 或 `false` 来创建 `bool`。与 Ruby 和 JavaScript 等语言不同，Cairo 不会自动尝试将非布尔类型转换为布尔类型。如果你希望 `if` 代码块仅在数字不等于 0 时运行，例如，我们可以将 if 表达式更改为以下内容：

```cairo
#[executable]
fn main() {
    let number = 3;

    if number != 0 {
        println!("number was something other than zero");
    }
}

```

运行这段代码将打印 `number was something other than zero`。

## 使用 `else if` 处理多个条件

你可以通过在 `else if` 表达式中组合 `if` 和 `else` 来使用多个条件。例如：

```cairo
#[executable]
fn main() {
    let number = 3;

    if number == 12 {
        println!("number is 12");
    } else if number == 3 {
        println!("number is 3");
    } else if number - 2 == 1 {
        println!("number minus 2 is 1");
    } else {
        println!("number not found");
    }
}
```

这个程序有四条可能的路径可以走。运行后，你应该看到以下输出：

```shell
$ scarb execute 
   Compiling no_listing_25_else_if v0.1.0 (listings/ch02-common-programming-concepts/no_listing_30_else_if/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_25_else_if
number is 3


```

当这个程序执行时，它依次检查每个 `if` 表达式，并执行条件计算为 `true` 的第一个主体。注意，即使 `number - 2 == 1` 是 `true`，我们也没有看到输出 `number minus 2 is 1`，也没有看到来自 `else` 块的 `number not found` 文本。这是因为 Cairo 只执行第一个真条件的代码块，一旦找到一个，它甚至不会检查其余的。使用过多的 `else if` 表达式会使你的代码杂乱无章，所以如果你有多个，你可能想要重构你的代码。[第 {{#chap enums-and-pattern-matching}} 章][match] 描述了一个强大的 Cairo 分支结构，称为 `match`，专门处理这些情况。

[match]: ./ch06-02-the-match-control-flow-construct.md

## 在 `let` 语句中使用 `if`

因为 `if` 是一个表达式，我们可以在 `let` 语句的右侧使用它来讲结果赋值给一个变量。

```cairo
#[executable]
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };

    if number == 5 {
        println!("condition was true and number is {}", number);
    }
}
```

```shell
$ scarb execute 
   Compiling no_listing_26_if_let v0.1.0 (listings/ch02-common-programming-concepts/no_listing_31_if_let/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_26_if_let
condition was true and number is 5


```

`number` 变量将绑定到一个基于 `if` 表达式结果的值，在这里是 5。

## 使用循环重复执行

多次执行一块代码通常很有用。为了这个任务，Cairo 提供了一个简单的循环语法，它将运行循环体内的代码直到结束，然后立即从头开始。为了试验循环，让我们创建一个名为 _loops_ 的新项目。

Cairo 有三种类型的循环：`loop`，`while` 和 `for`。让我们尝试每一个。

### 使用 `loop` 重复代码

`loop` 关键字告诉 Cairo 一遍又一遍地执行一块代码，直到永远或直到你明确告诉它停止。

作为一个例子，将 _loops_ 目录中的 _src/lib.cairo_ 文件更改为如下所示：

```cairo
// TAG: does_not_run

#[executable]
fn main() {
    loop {
        println!("again!");
    }
}
```

当我们运行这个程序时，我们将看到 `again!` 被连续不断地打印，直到程序耗尽 gas 或者我们手动停止程序。大多数终端支持键盘快捷键 ctrl-c 来中断陷入持续循环的程序。试一试：

```shell
$ scarb execute --available-gas=20000000
   Compiling loops v0.1.0 (file:///projects/loops)
    Finished release target(s) in 0 seconds
     Running loops
again!
again!
again!
^Cagain!
```

符号 `^C` 代表你按下 ctrl-c 的位置。你可能会也可能不会在 ^C 之后看到单词 `again!` 打印出来，这取决于接收到中断信号时代码在循环中的位置。

> 注意：Cairo 通过包含 gas 计来防止我们运行无限循环的程序。Gas 计是一种限制程序中可以完成的计算量的机制。通过设置 `--available-gas` 标志的值，我们可以设置程序可用的最大 gas 量。Gas 是表示指令计算成本的计量单位。当 gas 计耗尽时，程序将停止。在前一种情况下，我们将 gas 限制设置得足够高，以便程序运行相当长的时间。

> 这在部署在 Starknet 上的智能合约的背景下尤为重要，因为它可以防止在网络上运行无限循环。如果你正在编写一个需要运行循环的程序，你将需要使用设置了足够大值的 `--available-gas` 标志来执行它，以便运行程序。

现在，尝试再次运行相同的程序，但这次将 `--available-gas` 标志设置为 `200000` 而不是 `2000000000000`。你会看到程序只打印了 3 次 `again!` 就停止了，因为它耗尽了 gas 无法继续执行循环。

幸运的是，Cairo 也提供了一种使用代码跳出循环的方法。你可以把 `break` 关键字放在循环内，告诉程序何时停止执行循环。

```cairo
#[executable]
fn main() {
    let mut i: usize = 0;
    loop {
        if i > 10 {
            break;
        }
        println!("i = {}", i);
        i += 1;
    }
}
```

`continue` 关键字告诉程序转到循环的下一次迭代，并跳过本次迭代中的其余代码。让我们在循环中添加一个 `continue` 语句，当 `i` 等于 `5` 时跳过 `println!` 语句。

```cairo
#[executable]
fn main() {
    let mut i: usize = 0;
    loop {
        if i > 10 {
            break;
        }
        if i == 5 {
            i += 1;
            continue;
        }
        println!("i = {}", i);
        i += 1;
    }
}
```

执行此程序将在 `i` 等于 `5` 时不打印 `i` 的值。

### 从循环返回值

`loop` 的用途之一是重试你明知可能失败的操作，例如检查操作是否成功。你可能还需要将该操作的结果从循环中传递给代码的其余部分。为此，你可以在用于停止循环的 `break` 表达式之后添加你想要返回的值；该值将从循环中返回，以便你可以使用它，如下所示：

```cairo
#[executable]
fn main() {
    let mut counter = 0;

    let result = loop {
        if counter == 10 {
            break counter * 2;
        }
        counter += 1;
    };

    println!("The result is {}", result);
}
```

在循环之前，我们声明一个名为 `counter` 的变量并将其初始化为 `0`。然后我们声明一个名为 `result` 的变量来保存从循环返回的值。在循环的每次迭代中，我们检查 `counter` 是否等于 `10`，然后将 `counter` 变量加 `1`。当条件满足时，我们使用 `break` 关键字和值 `counter * 2`。在循环之后，我们使用分号结束将值赋值给 `result` 的语句。最后，我们打印 `result` 中的值，在这种情况下是 `20`。

### 使用 `while` 的条件循环

程序通常需要在循环内评估条件。当条件为 `true` 时，循环运行。当条件不再为 `true` 时，程序调用 `break`，停止循环。可以使用 `loop`、`if`、`else` 和 `break` 的组合来实现像这样的行为；如果你愿意，你现在可以在程序中尝试一下。然而，这种模式非常常见，以至于 Cairo 有一个内置的语言结构，称为 `while` 循环。

在清单 {{#ref while-true}} 中，我们使用 `while` 循环程序三次，每次打印 `number` 的值后倒数，然后在循环之后，打印一条消息并退出。

```cairo
#[executable]
fn main() {
    let mut number = 3;

    while number != 0 {
        println!("{number}!");
        number -= 1;
    }

    println!("LIFTOFF!!!");
}
```

{{#label while-true}} <span class="caption">清单 {{#ref while-true}}: 使用 `while` 循环在条件保持 `true` 时运行代码。</span>

如果你使用 `loop`、`if`、`else` 和 `break`，这种结构消除了许多必要的嵌套，并且更清晰。只要条件计算为 `true`，代码就会运行；否则，它退出循环。

### 使用 `for` 遍历集合

你也可以使用 while 结构来遍历集合的元素，例如数组。例如，清单 {{#ref iter-while}} 中的循环打印数组 `a` 中的每个元素。

```cairo
#[executable]
fn main() {
    let a = [10, 20, 30, 40, 50].span();
    let mut index = 0;

    while index < 5 {
        println!("the value is: {}", a[index]);
        index += 1;
    }
}
```

{{#label iter-while}} <span class="caption">清单 {{#ref iter-while}}: 使用 `while` 循环遍历集合的每个元素</span>

这里，代码通过数组中的元素向上计数。它从索引 `0` 开始，然后循环直到达到数组中的最后一个索引（即，当 `index < 5` 不再为 `true` 时）。运行这段代码将打印数组中的每个元素：

```shell
$ scarb execute 
   Compiling no_listing_45_iter_loop_while v0.1.0 (listings/ch02-common-programming-concepts/no_listing_45_iter_loop_while/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_45_iter_loop_while
the value is: 10
the value is: 20
the value is: 30
the value is: 40
the value is: 50


```

所有五个数组值都如预期出现在终端中。即使 `index` 在某个时刻会达到值 `5`，循环在尝试从数组中获取第六个值之前就停止执行了。

然而，这种方法容易出错；如果索引值或测试条件不正确，我们可能会导致程序 panic。例如，如果你将 `a` 数组的定义更改为有四个元素，但忘记将条件更新为 `while index < 4`，代码将会 panic。它也很慢，因为编译器会添加运行时代码，以便在每次循环迭代中执行索引是否在数组边界内的条件检查。

作为一个更简洁的替代方案，你可以使用 `for` 循环并为集合中的每个项目执行一些代码。`for` 循环看起来像清单 {{#ref iter-for}} 中的代码。

```cairo
#[executable]
fn main() {
    let a = [10, 20, 30, 40, 50].span();

    for element in a {
        println!("the value is: {element}");
    }
}
```

{{#label iter-for}} <span class="caption">清单 {{#ref iter-for}}: 使用 `for` 循环遍历集合的每个元素</span>

当我们运行这段代码时，我们将看到与清单 {{#ref iter-while}} 相同的输出。更重要的是，我们现在提高了代码的安全性，并消除了因超出数组末尾或没有走得足够远而遗漏某些项目而导致 bug 的机会。

使用 `for` 循环，如果你更改数组中值的数量，你不需要记得更改任何其他代码，正如你在清单 {{#ref iter-while}} 中使用的方法那样。

`for` 循环的安全性和简洁性使它们成为 Cairo 中最常用的循环结构。即使在你想运行某些代码一定次数的情况下，如清单 {{#ref while-true}} 中使用 while 循环的倒计时示例。另一种运行代码一定次数的方法是使用核心库提供的 `Range`，它按顺序生成从一个数字开始并在另一个数字之前结束的所有数字。

这是你如何使用 `Range` 从 1 数到 3：

```cairo
#[executable]
fn main() {
    for number in 1..4_u8 {
        println!("{number}!");
    }
    println!("Go!!!");
}
```

这段代码好一点，不是吗？

## 循环和递归函数的等价性

循环和递归函数是多次重复代码块的两种常用方法。`loop` 关键字用于创建一个无限循环，可以使用 `break` 关键字中断。

```cairo
#[executable]
fn main() -> felt252 {
    let mut x: felt252 = 0;
    loop {
        if x == 2 {
            break;
        } else {
            x += 1;
        }
    }
    x
}
```

循环可以通过在函数内部调用函数本身转换为递归函数。这是模仿上面 `loop` 示例行为的递归函数的示例。

```cairo
#[executable]
fn main() -> felt252 {
    recursive_function(0)
}

fn recursive_function(mut x: felt252) -> felt252 {
    if x == 2 {
        x
    } else {
        recursive_function(x + 1)
    }
}
```

在这两种情况下，代码块都将无限期运行，直到满足条件 `x == 2`，此时将显示 x 的值。

在 Cairo 中，循环和递归不仅在概念上等效：它们还被编译成类似的低级表示。为了理解这一点，我们可以将两个示例都编译为 Sierra，并分析 Cairo 编译器为这两个示例生成的 Sierra 代码。在你的 `Scarb.toml` 文件中添加以下内容：

```toml
[lib]
sierra-text = true
```

然后，运行 `scarb build` 编译两个示例。你会发现为这两个示例生成的 Sierra 代码非常相似，因为循环在 Sierra 语句中被编译成了递归函数。

> 注意：对于我们的示例，我们的发现来自于理解 Sierra 中显示两个程序执行轨迹的 **statements** 部分。如果你对学习更多关于 Sierra 的知识感到好奇，请查看 [Exploring Sierra](https://medium.com/nethermind-eth/under-the-hood-of-cairo-1-0-exploring-sierra-7f32808421f5)。

{{#quiz ../quizzes/ch02-05-control-flow.toml}}

## 总结

你做到了！这是一个相当大的章节：你学习了变量、数据类型、函数、注释、`if` 表达式和循环！为了练习本章讨论的概念，尝试构建程序来做以下事情：

- 生成第 _n_ 个斐波那契数。
- 计算数字 _n_ 的阶乘。

现在，我们将在下一章回顾 Cairo 中的常见集合类型。
