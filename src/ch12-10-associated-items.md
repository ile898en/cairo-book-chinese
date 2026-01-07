# 关联项 (Associated Items)

_关联项 (Associated Items)_ 是在 [traits] 中声明或在 [implementations] 中定义的项。具体来说，有 [关联函数][associated functions]（包括我们已经在 [第 {{#chap using-structs-to-structure-related-data}} 章][methods] 中介绍过的方法）、[关联类型][associated types]、[关联常量][associated constants] 和 [关联实现][associated implementations]。

[traits]: ./ch08-02-traits-in-cairo.md
[implementations]: ./ch08-02-traits-in-cairo.md#implementing-a-trait-on-a-type
[associated types]: ./ch12-10-associated-items.md#associated-types
[associated functions]: ./ch05-03-method-syntax.md#associated-functions
[methods]: ./ch05-03-method-syntax.md
[associated constants]: ./ch12-10-associated-items.md#associated-constants
[associated implementations]:
  ./ch12-10-associated-items.md#associated-implementations

当关联项与实现逻辑相关时，它们很有用。例如，`Option` 上的 `is_some` 方法与 Options 本质相关，因此应该是关联的。

每种关联项都有两种形式：包含实际实现的定义和声明定义的签名的声明。

## 关联类型 (Associated Types)

关联类型是 _类型别名_，允许你在 traits 中定义抽象类型占位符。关联类型允许 trait 实现者选择要使用的实际类型，而不是在 trait 定义中指定具体类型。

让我们考虑以下 `Pack` trait：

```cairo, noplayground
trait Pack<T> {
    type Result;

    fn pack(self: T, other: T) -> Self::Result;
}
```

我们的 `Pack` trait 中的 `Result` 类型充当将在稍后填充的类型的占位符。将关联类型视为在 trait 中留出空白空间，以便每个实现写入其所需的特定类型。这种方法使你的 trait 定义保持整洁和灵活。当你使用 trait 时，你不需要担心指定这些类型 —— 它们已经由实现为你选择了。在我们的 `Pack` trait 中，类型 `Result` 就是这样一个占位符。方法的定义表明它将返回 `Self::Result` 类型的值，但它没有指定 `Result` 实际上是什么。这留给了 `Pack` trait 的实现者，他们将为 `Result` 指定具体类型。当调用 `pack` 方法时，它将返回该所选具体类型的值，无论它可能是什么。

让我们看看关联类型与更传统的泛型方法相比如何。假设我们需要一个函数 `foo`，它可以打包两个 `T` 类型的变量。如果没有关联类型，我们要定义一个 `PackGeneric` trait 和一个打包两个 `u32` 的实现，如下所示：

```cairo, noplayground
trait PackGeneric<T, U> {
    fn pack_generic(self: T, other: T) -> U;
}

impl PackGenericU32 of PackGeneric<u32, u64> {
    fn pack_generic(self: u32, other: u32) -> u64 {
        let shift: u64 = 0x100000000; // 2^32
        self.into() * shift + other.into()
    }
}
```

使用这种方法，`foo` 将实现为：

```cairo, noplayground
fn foo<T, U, +PackGeneric<T, U>>(self: T, other: T) -> U {
    self.pack_generic(other)
}
```

注意 `foo` 如何需要同时指定 `T` 和 `U` 作为泛型参数。现在，让我们将其与具有关联类型的 `Pack` trait 进行比较：

```cairo, noplayground
impl PackU32Impl of Pack<u32> {
    type Result = u64;

    fn pack(self: u32, other: u32) -> Self::Result {
        let shift: u64 = 0x100000000; // 2^32
        self.into() * shift + other.into()
    }
}
```

使用关联类型，我们可以更简洁地定义 `bar`：

```cairo, noplayground
fn bar<T, impl PackImpl: Pack<T>>(self: T, b: T) -> PackImpl::Result {
    PackImpl::pack(self, b)
}
```

最后，让我们看看这两种方法的实际效果，证明最终结果是相同的：

```cairo
#[executable]
fn main() {
    let a: u32 = 1;
    let b: u32 = 1;

    let x = foo(a, b);
    let y = bar(a, b);

    // result is 2^32 + 1
    println!("x: {}", x);
    println!("y: {}", y);
}
```

如你所见，`bar` 不需要为打包结果指定第二个泛型类型。此信息隐藏在 `Pack` trait 的实现中，使函数签名更清晰、更灵活。关联类型允许我们以更少的冗长表达相同的功能，同时仍然保持泛型编程的灵活性。

## 关联常量 (Associated Constants)

关联常量是与类型关联的常量。它们使用 `const` 关键字在 trait 中声明，并在其实现中定义。在下一个示例中，我们要定义一个通用的 `Shape` trait，我们为 `Triangle` 和 `Square` 实现该 trait。此 trait 包含一个关联常量，定义实现该 trait 的类型的边数。

```cairo, noplayground
trait Shape<T> {
    const SIDES: u32;
    fn describe() -> ByteArray;
}

struct Triangle {}

impl TriangleShape of Shape<Triangle> {
    const SIDES: u32 = 3;
    fn describe() -> ByteArray {
        "I am a triangle."
    }
}

struct Square {}

impl SquareShape of Shape<Square> {
    const SIDES: u32 = 4;
    fn describe() -> ByteArray {
        "I am a square."
    }
}
```

之后，我们创建一个 `print_shape_info` 泛型函数，它要求泛型参数实现 `Shape` trait。此函数将使用关联常量来检索几何图形的边数，并将其与其描述一起打印。

```cairo, noplayground
fn print_shape_info<T, impl ShapeImpl: Shape<T>>() {
    println!("I have {} sides. {}", ShapeImpl::SIDES, ShapeImpl::describe());
}
```

关联常量允许我们将常量数字绑定到 `Shape` trait，而不是将其添加到结构体中或仅在实现中硬编码该值。这种方法提供了几个好处：

1. 它使常量与 trait 紧密联系，改善了代码组织。
2. 它允许进行编译时检查，以确保所有实现者都定义了所需的常量。
3. 它确保同一类型的两个实例具有相同的边数。

关联常量也可用于特定于类型的行为或配置，使其成为 trait 设计中的多功能工具。

我们要最终运行 `print_shape_info` 并查看 `Triangle` 和 `Square` 的输出：

```cairo
#[executable]
fn main() {
    print_shape_info::<Triangle>();
    print_shape_info::<Square>();
}
```

## 关联实现 (Associated Implementations)

关联实现允许你声明必须为关联类型存在 trait 实现。当你想要在 trait 级别强制执行类型和实现之间的关系时，此功能特别有用。它确保了跨不同 trait 实现的类型安全和一致性，这在泛型编程上下文中很重要。

为了理解关联实现的实用性，让我们检查 Cairo 核心库中的 `Iterator` 和 `IntoIterator` traits，以及它们使用 `ArrayIter<T>` 作为集合类型的各自实现：

```cairo, noplayground
// Collection type that contains a simple array
#[derive(Drop)]
pub struct ArrayIter<T> {
    array: Array<T>,
}

// T is the collection type
pub trait Iterator<T> {
    type Item;
    fn next(ref self: T) -> Option<Self::Item>;
}

impl ArrayIterator<T> of Iterator<ArrayIter<T>> {
    type Item = T;
    fn next(ref self: ArrayIter<T>) -> Option<T> {
        self.array.pop_front()
    }
}

/// Turns a collection of values into an iterator
pub trait IntoIterator<T> {
    /// The iterator type that will be created
    type IntoIter;
    impl Iterator: Iterator<Self::IntoIter>;

    fn into_iter(self: T) -> Self::IntoIter;
}

impl ArrayIntoIterator<T> of IntoIterator<Array<T>> {
    type IntoIter = ArrayIter<T>;
    fn into_iter(self: Array<T>) -> ArrayIter<T> {
        ArrayIter { array: self }
    }
}
```

1. `IntoIterator` trait 旨在将集合转换为迭代器。
2. `IntoIter` 关联类型表示将创建的特定迭代器类型。这允许不同的集合定义它们自己的高效迭代器类型。
3. 关联实现 `Iterator: Iterator<Self::IntoIter>`（我们正在讨论的关键特性）声明此 `IntoIter` 类型必须实现 `Iterator` trait。
4. 这种设计允许类型安全的迭代，而无需每次都显式指定迭代器类型，从而改善了代码的人体工程学。

关联实现创建了一个 trait 级别的绑定，保证：

- `into_iter` 方法将始终返回一个实现 `Iterator` 的类型。
- 这种关系对 `IntoIterator` 的所有实现强制执行，而不仅仅是逐个案例。

以下 `main` 函数演示了这在实践中如何用于 `Array<felt252>`：

```cairo
#[executable]
fn main() {
    let mut arr: Array<felt252> = array![1, 2, 3];

    // Converts the array into an iterator
    let mut iter = arr.into_iter();

    // Uses the iterator to print each element
    while let Some(item) = iter.next() {
        println!("Item: {}", item);
    }
}
```
