# 数组

数组是相同类型元素的集合。你可以通过使用核心库中的 `ArrayTrait` trait 来创建和使用数组方法。

需要注意的一件重要事情是，数组的修改选项有限。实际上，数组是其值不能被修改的队列。这与以下事实有关：内存槽一被写入，就不能被覆盖，只能从中读取。你只能将项目追加到数组的末尾，并从前面移除项目。

## 创建一个数组

创建数组是通过 `ArrayTrait::new()` 调用完成的。这是一个创建数组并向其追加 3 个元素的示例：

```cairo
#[executable]
fn main() {
    let mut a = ArrayTrait::new();
    a.append(0);
    a.append(1);
    a.append(2);
}
```

当需要时，你可以在像这样实例化数组时传递数组内部预期的项目类型，或者显式定义变量的类型。

```cairo, noplayground
let mut arr = ArrayTrait::<u128>::new();
```

```cairo, noplayground
let mut arr:Array<u128> = ArrayTrait::new();
```

## 更新数组

### 添加元素

要将元素添加到数组的末尾，可以使用 `append()` 方法：

```cairo
<!-- Warning: Anchor '5' not found in lib.cairo -->
```

### 移除元素

你只能通过使用 `pop_front()` 方法从数组的前面移除元素。此方法返回一个可以解包的 `Option`，其中包含被移除的元素，或者如果数组为空则返回 `None`。

```cairo
#[executable]
fn main() {
    let mut a = ArrayTrait::new();
    a.append(10);
    a.append(1);
    a.append(2);

    let first_value = a.pop_front().unwrap();
    println!("The first value is {}", first_value);
}
```

上面的代码将打印 `The first value is 10`，因为我们移除了添加的第一个元素。

在 Cairo 中，内存是不可变的，这意味着一旦数组元素被添加，就不可能修改它们。你只能将元素添加到数组的末尾，并从数组的前面移除元素。这些操作不需要内存突变，因为它们涉及更新指针而不是直接修改内存单元。

## 从数组读取元素

要访问数组元素，你可以使用返回不同类型的 `get()` 或 `at()` 数组方法。使用 `arr.at(index)` 等同于使用下标运算符 `arr[index]`。

### `get()` 方法

`get` 函数返回一个 `Option<Box<@T>>`，这意味着如果指定的索引处的元素存在于数组中，它返回一个指向该元素的快照的 Box 类型（Cairo 的智能指针类型）的 option。如果元素不存在，`get` 返回 `None`。当你期望访问可能不在数组边界内的索引，并且想要优雅地处理此类情况而不会引起 panic 时，此方法很有用。快照将在 ["引用和快照"][snapshots] 章节中更详细地解释。

这是一个使用 `get()` 方法的示例：

```cairo
// TAG: does_not_run

#[executable]
fn main() -> u128 {
    let mut arr = ArrayTrait::<u128>::new();
    arr.append(100);
    let index_to_access =
        1; // Change this value to see different results, what would happen if the index doesn't exist?
    match arr.get(index_to_access) {
        Some(x) => {
            *x
                .unbox() // Don't worry about * for now, if you are curious see Chapter 4.2 #desnap operator
            // It basically means "transform what get(idx) returned into a real value"
        },
        None => { panic!("out of bounds") },
    }
}
```

[snapshots]: ./ch04-02-references-and-snapshots.md#snapshots

### `at()` 方法

另一方面，`at` 函数及其等效的下标运算符直接返回指向指定索引处元素的快照，使用 `unbox()` 运算符提取存储在 box 中的值。如果索引越界，则会发生 panic 错误。你应该只在希望程序在提供的索引超出数组边界时 panic 时使用 `at`，这可以防止意外行为。

```cairo
#[executable]
fn main() {
    let mut a = ArrayTrait::new();
    a.append(0);
    a.append(1);

    // using the `at()` method
    let first = *a.at(0);
    assert!(first == 0);
    // using the subscripting operator
    let second = *a[1];
    assert!(second == 1);
}
```

在这个例子中，名为 `first` 的变量将获得值 `0`，因为那是数组中索引 `0` 处的值。名为 `second` 的变量将从数组中的索引 `1` 获得值 `1`。

总之，当你想要对越界访问尝试进行 panic 时使用 `at`，当你更喜欢优雅地处理此类情况而不 panic 时使用 `get`。

## 大小相关方法

要确定数组中的元素数量，请使用 `len()` 方法。返回值是 `usize` 类型。

如果你想检查数组是否为空，可以使用 `is_empty()` 方法，如果数组为空则返回 `true`，否则返回 `false`。

## `array!` 宏

有时，我们需要创建值在编译时已知的数组。执行此操作的基本方法是多余的。你会先声明数组，然后逐个追加每个值。`array!` 是通过组合这两个步骤来完成此任务的一种更简单的方法。在编译时，宏扩展为按顺序追加项目的代码。关于声明性宏如何匹配模式和扩展的更深入解释，请参阅 [宏 → 声明性内联宏](./ch12-05-macros.md#declarative-inline-macros-for-general-metaprogramming)。

不使用 `array!`:

```cairo
    let mut arr = ArrayTrait::new();
    arr.append(1);
    arr.append(2);
    arr.append(3);
    arr.append(4);
    arr.append(5);
```

使用 `array!`:

```cairo
    let arr = array![1, 2, 3, 4, 5];
```

## 使用枚举存储多种类型

如果你想在数组中存储不同类型的元素，可以使用 `Enum` 定义一个可以保存多种类型的自定义数据类型。枚举将在 ["枚举和模式匹配"][enums] 章节中更详细地解释。

```cairo
#[derive(Copy, Drop)]
enum Data {
    Integer: u128,
    Felt: felt252,
    Tuple: (u32, u32),
}

#[executable]
fn main() {
    let mut messages: Array<Data> = array![];
    messages.append(Data::Integer(100));
    messages.append(Data::Felt('hello world'));
    messages.append(Data::Tuple((10, 30)));
}
```

[enums]: ./ch06-00-enums-and-pattern-matching.md

## Span

`Span` 是一个表示 `Array` 快照的结构体。它旨在提供对数组元素的安全且受控的访问，而不修改原始数组。Span 对于确保数据完整性和避免在函数之间传递数组或执行只读操作时的借用问题特别有用，如 ["引用和快照"][references] 中介绍的那样。

`Array` 提供的除 `append()` 方法外的所有方法也可以与 `Span` 一起使用。

[references]: ./ch04-02-references-and-snapshots.md

### 将数组转换为 Span

要创建 `Array` 的 `Span`，请调用 `span()` 方法：

```cairo
#[executable]
fn main() {
    let mut array: Array<u8> = ArrayTrait::new();
    array.span();
}
```

{{#quiz ../quizzes/ch03-01-arrays.toml}}
