# 方法语法

_方法 (Methods)_ 与函数类似：我们使用 `fn` 关键字和名称声明它们，它们可以有参数和返回值，并且包含在从别处调用该方法时运行的代码。与函数不同，方法是在结构体（或枚举，我们将在 [第 {{#chap enums-and-pattern-matching}} 章][enums] 中介绍）的上下文中定义的，并且它们的第一个参数总是 `self`，它代表调用该方法的类型的实例。

## 定义方法

让我们改变以 `Rectangle` 实例为参数的 `area` 函数，改为在 `Rectangle` 结构体上定义一个 `area` 方法，如清单 {{#ref area-method}} 所示。

```cairo, noplayground
#[derive(Copy, Drop)]
struct Rectangle {
    width: u64,
    height: u64,
}

//ANCHOR: trait_definition
trait RectangleTrait {
    fn area(self: @Rectangle) -> u64;
}
//ANCHOR_END: trait_definition

//ANCHOR: trait_implementation
impl RectangleImpl of RectangleTrait {
    fn area(self: @Rectangle) -> u64 {
        (*self.width) * (*self.height)
    }
}
//ANCHOR_END: trait_implementation

//ANCHOR: main
#[executable]
fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    println!("Area is {}", rect1.area());
}
//ANCHOR_END: main
```

{{#label area-method}} 清单 {{#ref area-method}}: 在 `Rectangle` 结构体上定义 `area` 方法。

要在 `Rectangle` 的上下文中定义函数，我们需要为 `RectangleTrait` trait 启动一个 `impl`（实现）块，该 trait 定义了可以在 `Rectangle` 实例上调用的方法。由于只能为 trait 而不能为类型定义 impl 块，我们需要首先定义这个 trait - 但它不打算用于任何其他事情。

此 `impl` 块内的所有内容都将与 `Rectangle` 类型相关联。然后我们将 `area` 函数移到 `impl` 花括号内，并将签名中和主体内各处的第一个（在这种情况下是唯一的）参数更改为 `self`。在 `main` 中，我们之前调用 `area` 函数并将 `rect1` 作为参数传递的地方，我们现在可以使用 _方法语法_ 在我们的 `Rectangle` 实例上调用 `area` 方法。方法语法在实例之后：我们添加一个点，后跟方法名称、括号和任何参数。

在 `area` 的签名中，我们使用 `self: @Rectangle` 而不是 `rectangle: @Rectangle`。方法必须有一个名为 `self` 的参数作为它们的第一个参数，`self` 的类型指示了可以调用该方法的类型。方法可以获取 `self` 的所有权，但 `self` 也可以像任何其他参数一样通过快照或引用传递。

> 类型和 trait 之间没有直接联系。只有方法的 `self` 参数的类型定义了可以调用此方法的类型。这意味着，从技术上讲，可以在同一个 trait 中定义多个类型的方法（例如混合 `Rectangle` 和 `Circle` 方法）。但 **这不是推荐的做法**，因为它会导致混淆。

使用方法而不是函数的主要原因，除了提供方法语法外，还在于组织。我们将所有可以用一个类型的实例做的事情放在一个 `impl` 块中，而不是让我们代码的未来用户在我们提供的库中的各个地方搜索 `Rectangle` 的功能。

## `generate_trait` 属性

如果你熟悉 Rust，你可能会觉得 Cairo 的方法令人困惑，因为方法不能直接在类型上定义。相反，你必须定义一个 [trait](./ch08-02-traits-in-cairo.md)，并实现与该方法预期的类型相关联的此 trait。然而，定义一个 trait 然后实现它以在特定类型上定义方法是冗长的，并且不必要的：trait 本身不会被重用。

因此，为了避免定义无用的 trait，Cairo 提供了 `#[generate_trait]` 属性添加到 trait 实现上方，这告诉编译器为你生成相应的 trait 定义，并让你只关注实现。这两种方法是等效的，但在这种情况下不显式定义 trait 被认为是最佳实践。

前一个例子也可以写成如下形式：

```cairo
#[derive(Copy, Drop)]
struct Rectangle {
    width: u64,
    height: u64,
}

#[generate_trait]
impl RectangleImpl of RectangleTrait {
    fn area(self: @Rectangle) -> u64 {
        (*self.width) * (*self.height)
    }
}

#[executable]
fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    println!("Area is {}", rect1.area());
}
```

让我们在接下来的章节中使用这个 `#[generate_trait]` 来使我们的代码更清晰。

## 快照和引用

由于 `area` 方法不修改调用实例，`self` 被声明为使用 `@` 快照运算符的 `Rectangle` 实例的快照。但是，当然，我们也可以定义一些接收此实例的可变引用的方法，以便能够修改它。

让我们编写一个新方法 `scale`，它使用作为参数给出的 `factor` 来调整矩形的大小：

```cairo
#[generate_trait]
impl RectangleImpl of RectangleTrait {
    fn area(self: @Rectangle) -> u64 {
        (*self.width) * (*self.height)
    }
    fn scale(ref self: Rectangle, factor: u64) {
        self.width *= factor;
        self.height *= factor;
    }
}

#[executable]
fn main() {
    let mut rect2 = Rectangle { width: 10, height: 20 };
    rect2.scale(2);
    println!("The new size is (width: {}, height: {})", rect2.width, rect2.height);
}
```

也可以通过仅使用 `self` 作为第一个参数来定义获取实例所有权的方法，但这很少见。这种技术通常在方法将 `self` 转换为其他东西并且你想防止调用者在转换后使用原始实例时使用。

查看 [理解所有权](ch04-00-understanding-ownership.md) 章节以获取有关这些重要概念的更多详细信息。

## 多个参数的方法

让我们通过在 `Rectangle` 结构体上实现另一个方法来练习使用方法。这次我们要编写方法 `can_hold`，它接受另一个 `Rectangle` 实例，如果此矩形可以完全容纳在 self 内，则返回 `true`；否则，它应该返回 false。

```cairo
#[generate_trait]
impl RectangleImpl of RectangleTrait {
    fn area(self: @Rectangle) -> u64 {
        *self.width * *self.height
    }

    fn scale(ref self: Rectangle, factor: u64) {
        self.width *= factor;
        self.height *= factor;
    }

    fn can_hold(self: @Rectangle, other: @Rectangle) -> bool {
        *self.width > *other.width && *self.height > *other.height
    }
}

#[executable]
fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    let rect2 = Rectangle { width: 10, height: 40 };
    let rect3 = Rectangle { width: 60, height: 45 };

    println!("Can rect1 hold rect2? {}", rect1.can_hold(@rect2));
    println!("Can rect1 hold rect3? {}", rect1.can_hold(@rect3));
}
```

在这里，我们期望 `rect1` 可以容纳 `rect2` 但不能容纳 `rect3`。

## 关联函数

我们称在与特定类型关联的 `impl` 块内定义的所有函数为 _关联函数 (Associated functions)_。虽然编译器不强制执行此操作，但将与同一类型相关的关联函数保留在同一 `impl` 块中是一个好习惯 - 例如，所有与 `Rectangle` 相关的函数都将分组在 `RectangleTrait` 的同一 `impl` 块中。

方法是一种特殊类型的关联函数，但我们也可以定义没有 `self` 作为其第一个参数的关联函数（因此不是方法），因为它们不需要该类型的实例来工作，但仍与该类型相关联。

不是方法的关联函数通常用于将返回类型新实例的构造函数。这些通常被称为 `new`，但 `new` 不是一个特殊的名称，也不是语言内置的。例如，我们可以选择提供一个名为 `square` 的关联函数，该函数将具有一个维度参数并将其用作宽度和高度，从而使创建正方形 `Rectangle` 更容易，而不必指定相同的值两次：

让我们创建函数 `new`，它从 `width` 和 `height` 创建一个 `Rectangle`；`square`，它从 `size` 创建一个正方形 `Rectangle`；以及 `avg`，它计算两个 `Rectangle` 实例的平均值：

```cairo
#[generate_trait]
impl RectangleImpl of RectangleTrait {
    fn area(self: @Rectangle) -> u64 {
        (*self.width) * (*self.height)
    }

    fn new(width: u64, height: u64) -> Rectangle {
        Rectangle { width, height }
    }

    fn square(size: u64) -> Rectangle {
        Rectangle { width: size, height: size }
    }

    fn avg(lhs: @Rectangle, rhs: @Rectangle) -> Rectangle {
        Rectangle {
            width: ((*lhs.width) + (*rhs.width)) / 2, height: ((*lhs.height) + (*rhs.height)) / 2,
        }
    }
}

#[executable]
fn main() {
    let rect1 = RectangleTrait::new(30, 50);
    let rect2 = RectangleTrait::square(10);

    println!(
        "The average Rectangle of {:?} and {:?} is {:?}",
        @rect1,
        @rect2,
        RectangleTrait::avg(@rect1, @rect2),
    );
}
```

要调用 `square` 关联函数，我们使用带有结构体名称的 `::` 语法；`let sq = RectangleTrait::square(3);` 就是一个例子。此函数由 trait 命名空间化：`::` 语法用于关联函数和模块创建的命名空间。我们将在 [第 7 章][modules] 讨论模块。

注意 `avg` 函数也可以写成以第一个矩形为 `self` 的方法等。在这种情况下，它将不使用 `RectangleTrait::avg(@rect1, @rect2)`，而是通过 `rect1.avg(rect2)` 调用。

## 多个 Traits 和 `impl` 块

每个结构体允许有多个 `trait` 和 `impl` 块。例如，以下代码等同于 _多个参数的方法_ 部分中显示的代码，该部分将每个方法放在自己的 `trait` 和 `impl` 块中。

```cairo
#[generate_trait]
impl RectangleCalcImpl of RectangleCalc {
    fn area(self: @Rectangle) -> u64 {
        (*self.width) * (*self.height)
    }
}

#[generate_trait]
impl RectangleCmpImpl of RectangleCmp {
    fn can_hold(self: @Rectangle, other: @Rectangle) -> bool {
        *self.width > *other.width && *self.height > *other.height
    }
}
```

这里没有强烈的理由将这些方法分成多个 `trait` 和 `impl` 块，但这是有效的语法。

{{#quiz ../quizzes/ch05-03-method-syntax.toml}}

[enums]: ./ch06-01-enums.md
[modules]: ./ch07-02-defining-modules-to-control-scope.md
