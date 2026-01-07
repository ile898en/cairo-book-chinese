# 枚举

枚举 (Enums)，是 "enumerations" 的缩写，是一种通过定义一组固定的命名值（称为 _变体 (variants)_）来定义自定义数据类型的方法。枚举对于表示一组相关的值非常有用，其中每个值都是独特的并且具有特定的意义。

## 枚举变体和值

这是一个简单的枚举示例：

```cairo, noplayground
#[derive(Drop)]
enum Direction {
    North,
    East,
    South,
    West,
}
```

在这个例子中，我们定义了一个名为 `Direction` 的枚举，有四个变体：`North`、`East`、`South` 和 `West`。命名约定是对枚举变体使用 PascalCase。每个变体代表 `Direction` 类型的一个独特值。在这个特定例子中，变体没有任何关联值。可以使用这种语法实例化一个变体：

```cairo, noplayground
    let direction = Direction::North;
```

现在让我们想象我们的變体有关联值，存储方向的确切度数。我们可以定义一个新的 `Direction` 枚举：

```cairo, noplayground
#[derive(Drop)]
enum Direction {
    North: u128,
    East: u128,
    South: u128,
    West: u128,
}
```

并如下实例化它：

```cairo, noplayground
    let direction = Direction::North(10);
```

在这段代码中，每个变体都关联了一个 `u128` 值，表示以度为单位的方向。在下一个例子中，我们将看到也可以将不同的数据类型与每个变体关联。

编写根据枚举实例的变体不同而采取不同行动的代码很容易，在这个例子中是根据方向运行特定代码。你可以在 [Match 控制流结构][match] 部分了解更多相关信息。

[match]: ./ch06-02-the-match-control-flow-construct.md

## 结合自定义类型的枚举

枚举也可以用来存储与每个变体关联的更有趣的自定义数据。例如：

```cairo, noplayground
#[derive(Drop)]
enum Message {
    Quit,
    Echo: felt252,
    Move: (u128, u128),
}
```

在这个例子中，`Message` 枚举有三个变体：`Quit`、`Echo` 和 `Move`，都有不同的类型：

- `Quit` 没有任何关联值。
- `Echo` 是单个 `felt252`。
- `Move` 是两个 `u128` 值的元组。

你甚至可以在你的枚举变体中使用结构体或你定义的另一个枚举。

## 枚举的 Trait 实现

在 Cairo 中，你可以定义 trait 并为你的自定义枚举实现它们。这允许你定义与枚举关联的方法和行为。这是一个为前面的 `Message` 枚举定义 trait 并实现它的例子：

```cairo, noplayground
trait Processing {
    fn process(self: Message);
}

impl ProcessingImpl of Processing {
    fn process(self: Message) {
        match self {
            Message::Quit => { println!("quitting") },
            Message::Echo(value) => { println!("echoing {}", value) },
            Message::Move((x, y)) => { println!("moving from {} to {}", x, y) },
        }
    }
}
```

在这个例子中，我们为 `Message` 实现了 `Processing` trait。这是如何使用它来处理 `Quit` 消息的：

```cairo
    let msg: Message = Message::Quit;
    msg.process(); // prints "quitting"
```

## `Option` 枚举及其优势

`Option` 枚举是一个标准的 Cairo 枚举，表示可选值的概念。它有两个变体：`Some: T` 和 `None`。`Some: T` 表示有一个类型为 `T` 的值，而 `None` 表示没有值。

```cairo,noplayground
enum Option<T> {
    Some: T,
    None,
}
```

`Option` 枚举很有用，因为它允许你显式表示值不存在的可能性，使你的代码更具表现力且更易于推理。使用 `Option` 还可以帮助防止由使用未初始化或意外的 `null` 值引起的错误。

举个例子，这是一个返回数组中具有给定值的第一个元素的索引的函数，如果元素不存在则返回 `None`。

我们演示上述函数的两种方法：

- 使用 `find_value_recursive` 的递归方法。
- 使用 `find_value_iterative` 的迭代方法。

```cairo,noplayground
fn find_value_recursive(mut arr: Span<felt252>, value: felt252, index: usize) -> Option<usize> {
    match arr.pop_front() {
        Some(index_value) => { if (*index_value == value) {
            return Some(index);
        } },
        None => { return None; },
    }

    find_value_recursive(arr, value, index + 1)
}

fn find_value_iterative(mut arr: Span<felt252>, value: felt252) -> Option<usize> {
    for (idx, array_value) in arr.into_iter().enumerate() {
        if (*array_value == value) {
            return Some(idx);
        }
    }
    None
}
```

枚举在许多情况下都很有用，尤其是在使用我们刚刚使用的 `match` 流结构时。我们将在下一节描述它。

其他枚举非常常用，例如 `Result` 枚举，允许优雅地处理错误。我们将在 [“错误处理”][result enum] 章节详细解释 `Result` 枚举。

{{#quiz ../quizzes/ch06-01-enums.toml}}

[result enum]: ./ch09-02-recoverable-errors.md#the-result-enum
