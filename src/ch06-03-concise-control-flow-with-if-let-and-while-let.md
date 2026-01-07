# 使用 `if let` 和 `while let` 的简洁控制流

## `if let`

`if let` 语法允许你将 `if` 和 `let` 组合成一种不太冗长的方式来处理匹配一个模式的值，同时忽略其余模式。考虑清单 {{#ref config_max}} 中的程序，它匹配 `config_max` 变量中的 `Some<u8>` 值，但只有当值是 `Some` 变体时才想执行代码。

```cairo
    let config_max = Some(5);
    match config_max {
        Some(max) => println!("The maximum is configured to be {}", max),
        _ => (),
    }
```

{{#label config_max}} <span class="caption">清单 {{#ref config_max}}: 一个只关心当值是 `Some` 时执行代码的 `match`</span>

如果值是 `Some`，我们通过在模式中将值绑定到变量 `max` 来打印 `Some` 变体中的值。我们不想对 `None` 值做任何事情。为了满足 `match` 表达式，我们在处理完一个变体后必须添加 `_ => ()`，这是必须添加的令人讨厌的样板代码。

相反，我们可以使用 `if let` 以更简短的方式编写此代码。下面的代码与清单 {{#ref config_max}} 中的 `match` 行为相同：

```cairo
    let number = Some(5);
    if let Some(max) = number {
        println!("The maximum is configured to be {}", max);
    }
```

语法 `if let` 接受一个模式和一个由等号分隔的表达式。它的工作方式与 `match` 相同，其中表达式提供给 `match`，模式是其第一个分支。在这种情况下，模式是 `Some(max)`，`max` 绑定到 `Some` 内部的值。然后我们可以在 `if let` 块的主体中使用 `max`，就像我们在相应的 `match` 分支中使用 `max` 一样。如果值不匹配模式，则不运行 `if let` 块中的代码。

使用 `if let` 意味着更少的输入、更少的缩进和更少的样板代码。然而，你失去了 `match` 强制执行的穷尽性检查。在 `match` 和 `if let` 之间进行选择取决于你在特定情况下的做法，以及获得简洁性是否是失去穷尽性检查的适当权衡。

换句话说，你可以把 `if let` 看作是 `match` 的语法糖，当值匹配某个模式时运行代码，然后忽略所有其他值。

我们可以在 `if let` 中包含一个 `else`。与 `else` 一起的代码块与 `match` 表达式中的 `_` 情况一起的代码块相同。回想一下清单 {{#ref match-pattern-bind}} 中的 `Coin` 枚举定义，其中 `Quarter` 变体也保存一个 `UsState` 值。如果我们想计数我们看到的所有非 25 美分硬币，同时还要宣布 25 美分硬币的州，我们可以用 `match` 表达式这样做，像这样：

```cairo
    let coin = Coin::Quarter;
    let mut count = 0;
    match coin {
        Coin::Quarter => println!("You got a quarter!"),
        _ => count += 1,
    }
```

或者我们可以使用 `if let` 和 `else` 表达式，像这样：

```cairo
    let coin = Coin::Quarter;
    let mut count = 0;
    if let Coin::Quarter = coin {
        println!("You got a quarter!");
    } else {
        count += 1;
    }
```

如果你遇到程序中的逻辑过于冗长而无法使用 `match` 表达的情况，请记住 `if let` 也在你的 Cairo 工具箱中。

## `while let`

`while let` 语法与 `if let` 语法类似，但它允许你遍历值的集合，并为每个匹配指定模式的值执行代码块。在下面的情况下，模式是 `Some(x)`，它匹配 `Option` 枚举的任何 `Some` 变体。

```cairo
#[executable]
fn main() {
    let mut arr = array![1, 2, 3, 4, 5, 6, 7, 8, 9];
    let mut sum = 0;
    while let Some(value) = arr.pop_front() {
        sum += value;
    }
    println!("{}", sum);
}
```

使用 `while let` 提供了编写此循环的一种更简洁和惯用的方式，与具有显式模式匹配或处理 `Option` 类型的传统 `while` 循环相比。然而，与 `if let` 一样，你失去了 `match` 表达式提供的穷尽性检查，因此在必要时你需要小心处理 `while let` 循环之外的任何剩余情况。

## Let 链 (Let Chains)

Cairo 支持 let 链，以结合涉及 `if let` 或 `while let` 的多个条件而无需嵌套。这让你可以在单个表达式中进行模式匹配并应用额外的布尔条件：

```cairo
fn get_x() -> Option<u32> {
    Some(5)
}

fn get_y() -> Option<u32> {
    Some(8)
}

#[executable]
fn main() {
    // Using a let chain to combine pattern matching and additional conditions
    if let Some(x) = get_x() && x > 0 && let Some(y) = get_y() && y > 0 {
        let sum: u32 = x + y;
        println!("sum: {}", sum);
        return;
    }
    println!("x or y is not positive");
    // else is not supported yet;
}
```

> 注意：let 链表达式尚不支持 `else`；这将在以后的版本中添加。

## `let else`

`let else` 启用 `let` 绑定中的可反驳模式匹配，并在模式不匹配时允许 `else` 块发散（例如使用 `return`、`break`、`continue` 或 `panic!`）：

```cairo
#[derive(Drop)]
enum MyEnum {
    A: u32,
    B: u32,
}

fn foo(a: MyEnum) {
    let MyEnum::A(x) = a else {
        println!("Called with B");
        return;
    };
    println!("Called with A({x})");
}

#[executable]
fn main() {
    foo(MyEnum::A(42));
    foo(MyEnum::B(7));
}

```
