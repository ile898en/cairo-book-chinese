# 附录 C - 可派生 Traits

在书中各个地方，我们讨论了 `derive` 属性，你可以将其应用于结构体或枚举定义。`derive` 属性会生成代码，为你用 `derive` 语法注释的类型实现默认 trait。

在本附录中，我们提供了一个综合参考，详细介绍了标准库中与 `derive` 属性兼容的所有 traits。

此处列出的这些 traits 是核心库定义的唯一可以使用 `derive` 在你的类型上实现的 traits。标准库中定义的其他 traits 没有合理的默认行为，因此由你来以对你试图实现的目标有意义的方式实现它们。

## Drop 和 Destruct

当移出作用域时，变量首先需要被移动。这就是 `Drop` trait 介入的地方。你可以 [在此](ch04-01-what-is-ownership.md#no-op-destruction-the-drop-trait) 找到有关其用法的更多详细信息。

此外，字典在离开作用域之前需要被 squash（压扁/压缩）。手动对每个字典调用 `squash` 方法很快就会变得多余。`Destruct` trait 允许字典在离开作用域时自动被 squash。你也可以 [在此](ch04-01-what-is-ownership.md#destruction-with-a-side-effect-the-destruct-trait) 找到有关 `Destruct` 的更多信息。

## 用于复制值的 `Clone` 和 `Copy`

`Clone` trait 提供了显式创建值深拷贝的功能。

派生 `Clone` 实现了 `clone` 方法，该方法反过来对类型的每个组件调用 clone。这意味着类型中的所有字段或值也必须实现 `Clone` 才能派生 `Clone`。

这是一个简单的示例：

```cairo
#[derive(Clone, Drop)]
struct A {
    item: felt252,
}

#[executable]
fn main() {
    let first_struct = A { item: 2 };
    let second_struct = first_struct.clone();
    assert!(second_struct.item == 2, "Not equal");
}
```

`Copy` trait 允许复制值。你可以在其部分都实现 `Copy` 的任何类型上派生 `Copy`。

示例：

```cairo
#[derive(Copy, Drop)]
struct A {
    item: felt252,
}

#[executable]
fn main() {
    let first_struct = A { item: 2 };
    let second_struct = first_struct;
    // Copy Trait prevents first_struct from moving into second_struct
    assert!(second_struct.item == 2, "Not equal");
    assert!(first_struct.item == 2, "Not Equal");
}
```

## 用于打印和调试的 `Debug`

`Debug` trait 启用格式字符串中的调试格式化，你可以通过在 `{}` 占位符内添加 `:?` 来指示。

它允许你为了调试目的打印类型的实例，因此你和使用此类型的其他程序员可以在程序执行的特定点检查实例。

例如，如果你想打印 `Point` 类型变量的值，你可以这样做：

```cairo
#[derive(Copy, Drop, Debug)]
struct Point {
    x: u8,
    y: u8,
}

#[executable]
fn main() {
    let p = Point { x: 1, y: 3 };
    println!("{:?}", p);
}
```

```shell
scarb execute
Point { x: 1, y: 3 }
```

例如，当在测试中使用 `assert_xx!` 宏时，需要 `Debug` trait。如果相等性或比较断言失败，这些宏将打印作为参数给出的实例的值，以便程序员可以看到为什么两个实例不相等。

## 用于默认值的 `Default`

`Default` trait 允许创建类型的默认值。最常见的默认值为零。标准库中的所有原始类型都实现了 `Default`。

如果你想在复合类型上派生 `Default`，它的每个元素必须已经实现 `Default`。如果你有一个 [`enum`](ch06-01-enums.md) 类型，你必须通过在其变体之一上使用 `#[default]` 属性来声明其默认值。

一个例子：

```cairo
#[derive(Default, Drop)]
struct A {
    item1: felt252,
    item2: u64,
}

#[derive(Default, Drop, PartialEq)]
enum CaseWithDefault {
    A: felt252,
    B: u128,
    #[default]
    C: u64,
}

#[executable]
fn main() {
    let defaulted: A = Default::default();
    assert!(defaulted.item1 == 0_felt252, "item1 mismatch");
    assert!(defaulted.item2 == 0_u64, "item2 mismatch");

    let default_case: CaseWithDefault = Default::default();
    assert!(default_case == CaseWithDefault::C(0_u64), "case mismatch");
}
```

## 用于相等比较的 `PartialEq`

`PartialEq` trait 允许比较类型的实例是否相等，从而启用 `==` 和 `!=` 运算符。

当在结构体上派生 `PartialEq` 时，仅当所有字段都相等时，两个实例才相等；如果任何字段不同，则它们不相等。当为枚举派生时，每个变体等于其自身，不等于其他变体。

如果你无法派生它或者你想实现自定义规则，你可以为你的类型编写自己的 `PartialEq` trait 实现。在以下示例中，我们为 `PartialEq` 编写了一个实现，其中我们认为如果两个矩形具有相同的面积，则它们相等：

```cairo
#[derive(Copy, Drop)]
struct Rectangle {
    width: u64,
    height: u64,
}

impl PartialEqImpl of PartialEq<Rectangle> {
    fn eq(lhs: @Rectangle, rhs: @Rectangle) -> bool {
        (*lhs.width) * (*lhs.height) == (*rhs.width) * (*rhs.height)
    }

    fn ne(lhs: @Rectangle, rhs: @Rectangle) -> bool {
        (*lhs.width) * (*lhs.height) != (*rhs.width) * (*rhs.height)
    }
}

#[executable]
fn main() {
    let rect1 = Rectangle { width: 30, height: 50 };
    let rect2 = Rectangle { width: 50, height: 30 };

    println!("Are rect1 and rect2 equal? {}", rect1 == rect2);
}
```

在测试中使用 `assert_eq!` 宏时需要 `PartialEq` trait，这需要能够比较两个类型实例的相等性。

这是一个例子：

```cairo
#[derive(PartialEq, Drop)]
struct A {
    item: felt252,
}

#[executable]
fn main() {
    let first_struct = A { item: 2 };
    let second_struct = A { item: 2 };
    assert!(first_struct == second_struct, "Structs are different");
}
```

## 使用 `Serde` 进行序列化

`Serde` 为你的 crate 中定义的数据结构提供 `serialize` 和 `deserialize` 函数的 trait 实现。它允许你将结构转换为数组（或反之）。

> **[序列化 (Serialization)](https://zh.wikipedia.org/wiki/序列化)** 是将数据结构转换为易于存储或传输的格式的过程。假设你正在运行一个程序，并希望持久化其状态以便稍后恢复。为此，你可以获取程序正在使用的每个对象并保存它们的信息，例如保存在文件中。这是序列化的简化版本。现在，如果你想使用此保存的状态恢复程序，你将执行 **反序列化 (deserialization)**，这意味着从保存的源加载对象的状态。

例如：

```cairo
// TAG: does_not_run

#[derive(Serde, Drop)]
struct A {
    item_one: felt252,
    item_two: felt252,
}

#[executable]
fn main() {
    let first_struct = A { item_one: 2, item_two: 99 };
    let mut output_array = array![];
    first_struct.serialize(ref output_array);
    panic(output_array);
}

```

如果你运行 `main` 函数，输出将是：

```shell
Run panicked with [2, 99 ('c'), ].
```

我们可以在这里看到我们的结构体 `A` 已被序列化到输出数组中。注意，`serialize` 函数将你要转换为数组的类型的快照作为参数。这就是为什么这里需要为 `A` 派生 `Drop`，因为 `main` 函数保留了 `first_struct` 结构体的所有权。

此外，我们可以使用 `deserialize` 函数将序列化的数组转换回我们的 `A` 结构体。

这是一个例子：

```cairo
#[derive(Serde, Drop)]
struct A {
    item_one: felt252,
    item_two: felt252,
}

#[executable]
fn main() {
    let first_struct = A { item_one: 2, item_two: 99 };
    let mut output_array = array![];
    first_struct.serialize(ref output_array);
    let mut span_array = output_array.span();
    let deserialized_struct: A = Serde::<A>::deserialize(ref span_array).unwrap();
}
```

在这里，我们将序列化的数组 span 转换回结构体 `A`。`deserialize` 返回一个 `Option`，所以我们需要 unwrap 它。这也是为什么这里需要为 `A` 派生 `Drop`，因为 `main` 函数保留了 `first_struct` 结构体的所有权。当使用 `deserialize` 时，我们还需要指定我们要反序列化成的类型。

## 使用 `Hash` 进行哈希

可以在结构体和枚举上派生 `Hash` trait。这允许使用任何可用的哈希函数轻松地对它们进行哈希。对于要派生 `Hash` 属性的结构体或枚举，所有字段或变体本身都需要是可哈希的。

你可以参考 [哈希章节](ch12-04-hash.md) 以获取有关如何对复杂数据类型进行哈希的更多信息。

## 使用 `starknet::Store` 进行 Starknet 存储

`starknet::Store` trait 仅在构建 [Starknet](ch100-00-introduction-to-smart-contracts.md) 合约时相关。通过自动实现必要的读写函数，它允许类型在智能合约存储中使用。

你可以在 [合约存储章节](ch101-01-00-contract-storage.md) 中找到有关 Starknet 存储内部工作原理的详细信息。
