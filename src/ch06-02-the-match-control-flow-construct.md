# Match 控制流结构

Cairo 有一个极其强大的控制流结构叫 `match`，它允许你将一个值与一系列模式进行比较，然后根据匹配的模式执行代码。模式可以由字面值、变量名、通配符和许多其他东西组成。`match` 的力量来自于模式的表现力以及编译器确认所有可能的情况都得到了处理。

把 `match` 表达式想象成一台硬币分类机：硬币沿着一条有不同大小孔的轨道滑下，每枚硬币都会掉进它遇到的第一个适合它的孔里。同样地，值通过 match 中的每一个模式，在第一个“适合”的模式处，值就会进入关联的代码块在执行期间被使用。

说到硬币，让我们用它们作为使用 `match` 的例子！我们可以写一个函数，它接受一个未知的美国硬币，并以类似于计数机的方式，确定它是哪种硬币并以美分返回其值，如清单 {{#ref match-enum}} 所示。

```cairo,noplayground
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

// ANCHOR: function
fn value_in_cents(coin: Coin) -> felt252 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
// ANCHOR_END: function
```

{{#label match-enum}} <span class="caption">清单 {{#ref match-enum}}: 一个枚举和一个以枚举变体作为其模式的 `match` 表达式</span>

让我们分解 `value_in_cents` 函数中的 `match` 表达式。首先，我们列出 `match` 关键字后跟一个表达式，在这种情况下是值 `coin`。这看起来与用于 `if` 语句的条件表达式非常相似，但有一个很大的不同：对于 `if`，条件需要评估为布尔值，但这里它可以是任何类型。在这个例子中，`coin` 的类型是我们在第一行定义的 `Coin` 枚举。

接下来是 `match` 分支 (arms)。一个分支有两个部分：一个模式和一些代码。这里的第一个分支有一个模式，即值 `Coin::Penny`，然后是 `=>` 运算符，将模式和要运行的代码分开。这种情况下的代码只是值 `1`。每个分支与下一个分支用逗号分隔。

当 `match` 表达式执行时，它将结果值与每个分支的模式按给定的顺序进行比较。如果模式与值匹配，则执行与该模式关联的代码。如果该模式不匹配值，则执行继续到下一个分支，就像硬币分类机一样。我们可以有任意数量的分支：在上面的例子中，我们的 `match` 有四个分支。

与每个分支关联的代码是一个表达式，匹配分支中表达式的结果值是整个 match 表达式返回的值。

如果 `match` 分支代码很短，我们通常不使用花括号，就像在我们的例子中，每个分支只是返回一个值。如果你想在 `match` 分支中运行多行代码，你必须使用花括号，并在分支后跟一个逗号。例如，以下代码每次用 `Coin::Penny` 调用该方法时都会打印“Lucky penny!”，但仍返回块的最后一个值 `1`：

```cairo,noplayground
fn value_in_cents(coin: Coin) -> felt252 {
    match coin {
        Coin::Penny => {
            println!("Lucky penny!");
            1
        },
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

## 绑定到值的模式

`match` 分支的另一个有用特性是它们可以绑定到匹配模式的值的部分。这就是我们如何从枚举变体中提取值。

举个例子，让我们改变我们的一个枚举变体以在其中保存数据。从 1999 年到 2008 年，美国铸造了 25 美分硬币，一面是 50 个州中每个州的不同设计。没有其他硬币有州设计，所以只有 25 美分硬币有这个额外的值。我们可以通过更改 `Quarter` 变体以包含存储在其中的 `UsState` 值来将此信息添加到我们的 `enum` 中，我们在清单 {{#ref match-pattern-bind}} 中已经这样做了。

```cairo,noplayground

#[derive(Drop, Debug)] // Debug so we can inspect the state in a minute
enum UsState {
    Alabama,
    Alaska,
}

#[derive(Drop)]
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter: UsState,
}
```

{{#label match-pattern-bind}} <span class="caption">清单
{{#ref match-pattern-bind}}: 一个 `Coin` 枚举，其中 `Quarter` 变体也保存一个 `UsState` 值</span>

让我们想象一个朋友正在试图收集所有 50 个州的 25 美分硬币。当我们按硬币类型分类我们的零钱时，我们也会喊出每个 25 美分硬币关联的州名，这样如果它是我们朋友没有的，他们就可以把它加到他们的收藏中。

在这段代码的 `match` 表达式中，我们向匹配 `Coin::Quarter` 变体值的模式添加了一个名为 `state` 的变量。当 `Coin::Quarter` 匹配时，`state` 变量将绑定到那个 25 美分硬币的州的值。然后我们可以在该分支的代码中使用 `state`，像这样：

```cairo,noplayground
fn value_in_cents(coin: Coin) -> felt252 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        },
    }
}
```

因为 `state` 是一个实现了 `Debug` trait 的 `UsState` 枚举，我们可以用 `println!` 宏打印 `state` 值。

> 注意：`{:?}` 是一种特殊的格式化语法，允许打印传递给 `println!` 宏的参数的调试形式。你可以在 [附录 C][debug trait] 中找到关于它的更多信息。

如果我们调用 `value_in_cents(Coin::Quarter(UsState::Alaska))`，`coin` 将是 `Coin::Quarter(UsState::Alaska)`。当我们把那个值与每个匹配分支比较时，直到我们到达 `Coin::Quarter(state)` 都没有匹配的。在那一点，`state` 的绑定将是值 `UsState::Alaska`。然后我们可以在 `println!` 宏中使用该绑定，从而从 `Quarter` 的 `Coin` 枚举变体中获取内部状态值。

[debug trait]: ./appendix-03-derivable-traits.md#debug-for-printing-and-debugging

## 匹配 `Option<T>`

在上一节中，我们想在使用 `Option<T>` 时从 `Some` 情况中获取内部 `T` 值；我们也可以使用 `match` 处理 `Option<T>`，就像我们处理 `Coin` 枚举一样！我们将比较 `Option<T>` 的变体，而不是比较硬币，但 `match` 表达式的工作方式保持不变。

假设我们要写一个函数，它接受一个 `Option<u8>`，如果里面有值，就给那个值加 `1`。如果里面没有值，函数应该返回 `None` 值，不尝试执行任何操作。

这个函数很容易编写，这就多亏了 `match`，看起来像清单 {{#ref match-option}}。

```cairo
fn plus_one(x: Option<u8>) -> Option<u8> {
    match x {
        //ANCHOR: option_some
        Some(val) => Some(val + 1),
        //ANCHOR_END: option_some
        // ANCHOR: option_none
        None => None,
        //ANCHOR_END: option_none
    }
}

#[executable]
fn main() {
    let five: Option<u8> = Some(5);
    let six: Option<u8> = plus_one(five);
    let none = plus_one(None);
}
```

{{#label match-option}} <span class="caption">清单 {{#ref match-option}}: 一个在 `Option<u8>` 上使用 `match` 表达式的函数</span>

让我们更详细地检查 `plus_one` 的第一次执行。当我们调用 `plus_one(five)` 时，`plus_one` 主体中的变量 `x` 将具有值 `Some(5)`。然后我们将其与每个 `match` 分支进行比较：

```cairo,noplayground
        Some(val) => Some(val + 1),
```

`Some(5)` 值匹配模式 `Some(val)` 吗？是的！我们要么有相同的变体。`val` 绑定到包含在 `Some` 中的值，所以 `val` 取值 `5`。然后执行 `match` 分支中的代码，所以我们将 `val` 的值加 `1` 并创建一个包含我们总数 `6` 的新 `Some` 值。因为第一个分支匹配了，所以不再比较其他分支。

现在让我们考虑我们的 main 函数中 `plus_one` 的第二次调用，其中 `x` 是 `None`。我们进入 `match` 并与第一个分支比较：

```cairo,noplayground
        Some(val) => Some(val + 1),
```

`Some(val)` 值不匹配模式 `None`，所以我们继续到下一个分支：

```cairo
        None => None,
```

匹配了！没有值可以加，所以匹配结构结束并在 `=>` 右侧返回 `None` 值。

结合 `match` 和枚举在各种情况下都很有用。你会在 Cairo 代码中经常看到这种模式：针对枚举进行 `match`，将变量绑定到内部数据，然后基于此执行代码。起初这有点棘手，但一旦你习惯了它，你会希望所有语言都有它。它始终是用户的最爱。

## 匹配是穷尽的 (Exhaustive)

我们需要讨论 `match` 的另一个方面：分支的模式必须覆盖所有可能性。考虑这个版本的 `plus_one` 函数，它有一个错误并且无法编译：

```cairo,noplayground
fn plus_one(x: Option<u8>) -> Option<u8> {
    match x {
        Some(val) => Some(val + 1),
    }
}
```

我们没有处理 `None` 的情况，所以这段代码会导致错误。幸运的是，这是一个 Cairo 知道如何捕获的错误。如果我们尝试编译这段代码，我们会得到这个错误：

```shell
$ scarb execute 
   Compiling no_listing_08_missing_match_arm v0.1.0 (listings/ch06-enums-and-pattern-matching/no_listing_09_missing_match_arm/Scarb.toml)
error: Missing match arm: `None` not covered.
 --> listings/ch06-enums-and-pattern-matching/no_listing_09_missing_match_arm/src/lib.cairo:5:5-7:5
      match x {
 _____^
|         Some(val) => Some(val + 1),
|     }
|_____^

error: could not compile `no_listing_08_missing_match_arm` due to previous error
error: `scarb` command exited with error

```

Cairo 知道我们没有覆盖所有可能的情况，甚至知道我们忘记了哪个模式！Cairo 中的匹配是穷尽的：为了使代码有效，我们必须穷尽每一种可能性。特别是在 `Option<T>` 的情况下，当 Cairo 阻止我们忘记显式处理 `None` 情况时，它保护我们免受假设我们在可能有 null 时有一个值的影响，从而使之前讨论的 [十亿美元错误][null pointer] 成为不可能。

[null pointer]: https://en.wikipedia.org/wiki/Null_pointer#History

## 使用 `_` 通配符的 Catch-all

使用枚举，我们还可以对一些特定值采取特殊操作，但对所有其他值采取一个默认操作。`_` 是一个特殊模式，它可以匹配任何值并且不绑定到该值。你可以通过简单地添加一个新分支，并将 `_` 作为 `match` 表达式最后一个分支的模式来使用它。

想象我们有一台只接受 10 美分硬币 (Dime) 的自动售货机。我们想要一个处理投入硬币的函数，只有当硬币被接受时才返回 `true`。

这是一个实现此逻辑的 `vending_machine_accept` 函数：

```cairo,noplayground
fn vending_machine_accept(coin: Coin) -> bool {
    match coin {
        Coin::Dime => true,
        _ => false,
    }
}
```

这个例子也满足穷尽性要求，因为我们在最后一个分支中显式忽略了所有其他值；我们没有忘记任何东西。

> Cairo 中没有允许你使用模式值的 catch-all 模式。

<!--
  TODO move the following in a separate chapter when there's more pattern matching features in upcoming Cairo versions. cf rust book chapter 18
-->

## 使用 `|` 运算符的多个模式

在 `match` 表达式中，你可以使用 `|` 语法匹配多个模式，这是模式 _或_ 运算符。

例如，在下面的代码中，我们修改了 `vending_machine_accept` 函数，在一个分支中同时接受 `Dime` 和 `Quarter` 硬币：

```cairo,noplayground
fn vending_machine_accept(coin: Coin) -> bool {
    match coin {
        Coin::Dime | Coin::Quarter => true,
        _ => false,
    }
}
```

## 匹配元组

可以匹配元组。让我们引入一个新的 `DayType` 枚举：

```cairo,noplayground
#[derive(Drop)]
enum DayType {
    Week,
    Weekend,
    Holiday,
}
```

现在，假设我们的自动售货机在工作日接受任何硬币，但在周末和节假日只接受 25 美分和 10 美分硬币。我们可以修改 `vending_machine_accept` 函数以接受 `Coin` 和 `Weekday` 的元组，并且仅当给定的硬币在指定的日子被接受时才返回 `true`：

```cairo,noplayground
fn vending_machine_accept(c: (DayType, Coin)) -> bool {
    match c {
        (DayType::Week, _) => true,
        (_, Coin::Dime) | (_, Coin::Quarter) => true,
        (_, _) => false,
    }
}
```

为元组匹配模式的最后一个分支写 `(_, _)` 可能感觉多余。因此，如果我们想要，例如，我们的自动售货机只在工作日接受 25 美分硬币，我们可以使用 `_ =>` 语法：

```cairo,noplayground
fn vending_week_machine(c: (DayType, Coin)) -> bool {
    match c {
        (DayType::Week, Coin::Quarter) => true,
        _ => false,
    }
}
```

## 匹配 `felt252` 和整数变量

你也可以匹配 `felt252` 和整数变量。当你想要针对一系列值进行匹配时，这很有用。但是，有一些限制：

- 仅支持适合单个 `felt252` 的整数（即不支持 `u256`）。
- 第一个分支必须是 0。
- 每个分支必须覆盖一个连续的段，与其他分支连续。

想象我们正在实现一个游戏，你掷一个六面的骰子得到一个 0 到 5 之间的数字。如果你有 0、1 或 2，你赢了。如果你有 3，你可以再掷一次。对于所有其他值，你输了。

这是一个实现该逻辑的 match：

```cairo,noplayground
fn roll(value: u8) {
    match value {
        0 | 1 | 2 => println!("you won!"),
        3 => println!("you can roll again!"),
        _ => println!("you lost..."),
    }
}
```

{{#quiz ../quizzes/ch06-02-match.toml}}

> 这些限制计划在 Cairo 的未来版本中放宽。
