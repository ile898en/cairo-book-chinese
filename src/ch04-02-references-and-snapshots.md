# 引用和快照 (References and Snapshots)

前一个清单 {{#ref return-multiple-values}} 中的元组代码的问题在于，我们必须将 `Array` 返回给调用函数，以便我们在调用 `calculate_length` 之后仍然可以使用 `Array`，因为 `Array` 被移动到了 `calculate_length` 中。

## 快照 (Snapshots)

在上一章中，我们讨论了 Cairo 的所有权系统如何阻止我们在移动变量后使用它，从而保护我们免受可能对同一内存单元进行两次写入的影响。然而，这不是很方便。让我们看看如何使用快照在调用函数中保留变量的所有权。

在 Cairo 中，快照是程序执行在某一时刻值的不可变视图。回想一下，内存是不可变的，所以修改变量实际上填充了一个新的内存单元。旧的内存单元仍然存在，快照是引用那个“旧”值的变量。在这个意义上，快照是“对过去的”视图。

这是你如何定义和使用一个 `calculate_area` 函数，它接受 `Rectangle` 结构体的快照作为参数，而不是获取底层值的所有权。在这个例子中，`calculate_area` 函数返回作为快照传递的 `Rectangle` 的面积。由于我们将它作为不可变视图传递，我们可以确定 `calculate_area` 不会改变 `Rectangle`，并且所有权保留在 `main` 函数中。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[derive(Drop)]
struct Rectangle {
    height: u64,
    width: u64,
}

#[executable]
fn main() {
    let mut rec = Rectangle { height: 3, width: 10 };
    let first_snapshot = @rec; // Take a snapshot of `rec` at this point in time
    rec.height = 5; // Mutate `rec` by changing its height
    let first_area = calculate_area(first_snapshot); // Calculate the area of the snapshot
    let second_area = calculate_area(@rec); // Calculate the current area
    println!("The area of the rectangle when the snapshot was taken is {}", first_area);
    println!("The current area of the rectangle is {}", second_area);
}

fn calculate_area(rec: @Rectangle) -> u64 {
    *rec.height * *rec.width
}
```

> 注意：访问快照的字段（例如 `rec.height`）会产生这些字段的快照，我们使用 `*` 对其进行去快照 (desnap) 以获取值。这在这里之所以有效是因为 `u64` 实现了 `Copy`。你将在下一节了解更多关于去快照的内容。

这个程序的输出是：

```shell
$ scarb execute 
   Compiling no_listing_09_snapshots v0.1.0 (listings/ch04-understanding-ownership/no_listing_09_snapshots/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_09_snapshots
The area of the rectangle when the snapshot was taken is 30
The current area of the rectangle is 50


```

首先，注意变量声明和函数返回值中的所有元组代码都没了。其次，注意我们将 `@rec` 传入 `calculate_area`，并且在其定义中，我们接受 `@Rectangle` 而不是 `Rectangle`。

让我们仔细看看这里的函数调用：

```cairo
let second_length = calculate_length(@arr1); // 计算当前数组的长度
```

`@rec` 语法允许我们创建 `rec` 中值的快照。因为快照是执行在特定时刻值的不可变视图，所以不强制执行线性类型系统的通常规则。特别是，快照变量总是实现 `Drop` trait，从不实现 `Destruct` trait，即使是字典快照。

值得注意的是，`@T` 不是指针——快照是按值传递给函数的，就像普通变量一样。这意味着 `@T` 的大小与 `T` 的大小相同，当你将 `@rec` 传递给 `calculate_area` 时，整个结构体（在这种情况下，是具有两个 `u64` 字段的 `Rectangle`）被复制到了函数的堆栈中。对于大型数据结构，可以通过使用 `Box<T>` 来避免这种复制——这就提供了无需改变值的需求，我们将在 [第 12 章][chap-smart-pointers] 中探讨这一点，但现在，请理解快照依赖于这种按值传递机制。

同样，函数签名使用 `@` 来指示参数 `arr` 的类型是快照。让我们添加一些解释性注释：

```cairo, noplayground
fn calculate_area(
    rec_snapshot: @Rectangle // rec_snapshot 是 Rectangle 的快照
) -> u64 {
    *rec_snapshot.height * *rec_snapshot.width
} // 这里，rec_snapshot 超出作用域并被丢弃。
// 然而，因为它只是原始 `rec` 包含内容的视图，原始 `rec` 仍然可以使用。
```

变量 `rec_snapshot` 有效的作用域与任何函数参数的作用域相同，但是当 `rec_snapshot` 停止使用时，快照的底层值不会被丢弃。当函数拥有快照作为参数而不是实际值时，我们不需要返回值来归还原始值的所有权，因为我们从未拥有过它。

### 去快照运算符 (Desnap Operator)

要将快照转换回普通变量，可以使用 `desnap` 运算符 `*`，它是 `@` 运算符的反义词。

只有 `Copy` 类型可以被去快照。然而，在一般情况下，因为值没有被修改，`desnap` 运算符创建的新变量复用了旧值，所以去快照是一个完全免费的操作，就像 `Copy` 一样。

在下面的例子中，我们想计算一个矩形的面积，但我们不想在 `calculate_area` 函数中获取矩形的所有权，因为我们可能想在函数调用后再次使用矩形。由于我们的函数不改变矩形实例，我们可以将矩形的快照传递给函数，然后使用 `desnap` 运算符 `*` 将快照转换回值。

```cairo
#[derive(Drop)]
struct Rectangle {
    height: u64,
    width: u64,
}

#[executable]
fn main() {
    let rec = Rectangle { height: 3, width: 10 };
    let area = calculate_area(@rec);
    println!("Area: {}", area);
}

fn calculate_area(rec: @Rectangle) -> u64 {
    // As rec is a snapshot to a Rectangle, its fields are also snapshots of the fields types.
    // We need to transform the snapshots back into values using the desnap operator `*`.
    // This is only possible if the type is copyable, which is the case for u64.
    // Here, `*` is used for both multiplying the height and width and for desnapping the snapshots.
    *rec.height * *rec.width
}
```

但是，如果我们尝试修改作为快照传递的东西会发生什么？尝试清单 {{#ref modify-snapshot}} 中的代码。剧透警告：它不起作用！

<span class="filename">文件名: src/lib.cairo</span>

```cairo,does_not_compile
//TAG: does_not_compile
#[derive(Copy, Drop)]
struct Rectangle {
    height: u64,
    width: u64,
}

#[executable]
fn main() {
    let rec = Rectangle { height: 3, width: 10 };
    flip(@rec);
}

fn flip(rec: @Rectangle) {
    let temp = rec.height;
    rec.height = rec.width;
    rec.width = temp;
}
```

{{#label modify-snapshot}}

<span class="caption">清单 {{#ref modify-snapshot}}: 尝试修改快照值</span>

这是错误：

```shell
$ scarb execute 
   Compiling listing_04_04 v0.1.0 (listings/ch04-understanding-ownership/listing_04_attempt_modifying_snapshot/Scarb.toml)
error: Invalid left-hand side of assignment.
 --> listings/ch04-understanding-ownership/listing_04_attempt_modifying_snapshot/src/lib.cairo:16:5
    rec.height = rec.width;
    ^^^^^^^^^^

error: Invalid left-hand side of assignment.
 --> listings/ch04-understanding-ownership/listing_04_attempt_modifying_snapshot/src/lib.cairo:17:5
    rec.width = temp;
    ^^^^^^^^^

error: could not compile `listing_04_04` due to previous error
error: `scarb` command exited with error

```

编译器阻止我们修改与快照关联的值。

## 可变引用 (Mutable References)

我们可以通过使用 _可变引用_ 而不是快照来实现我们在清单 {{#ref modify-snapshot}} 中想要的行为。可变引用实际上是传递给函数的可变值，它们在函数结束时隐式返回，将所有权归还给调用上下文。通过这样做，它们允许你改变传递的值，同时通过在执行结束时自动返回它来保持对它的所有权。在 Cairo 中，可以使用 `ref` 修饰符将参数作为 _可变引用_ 传递。

> **注意**: 在 Cairo 中，只有当变量用 `mut` 声明为可变时，参数才能使用 `ref` 修饰符作为 _可变引用_ 传递。

在清单 4-5 中，我们在 `flip` 函数中使用可变引用来修改 `Rectangle` 实例的 `height` 和 `width` 字段的值。

```cairo
#[derive(Drop)]
struct Rectangle {
    height: u64,
    width: u64,
}

#[executable]
fn main() {
    let mut rec = Rectangle { height: 3, width: 10 };
    flip(ref rec);
    println!("height: {}, width: {}", rec.height, rec.width);
}

fn flip(ref rec: Rectangle) {
    let temp = rec.height;
    rec.height = rec.width;
    rec.width = temp;
}
```

<span class="caption">清单 4-5: 使用可变引用修改值</span>

首先，我们将 `rec` 更改为 `mut`。然后我们用 `ref rec` 将 `rec` 的可变引用传入 `flip`，并更新函数签名以接受带有 `ref rec: Rectangle` 的可变引用。这非常清楚地表明 `flip` 函数将改变作为参数传递的 `Rectangle` 实例的值。

与快照不同，可变引用允许突变，但像快照一样，`ref` 参数不是指针——它们也是按值传递的。当你传递 `ref rec` 时，无论是否实现 `Copy`，整个 `Rectangle` 类型都会被复制到函数的堆栈中。这确保了函数在其自己的数据本地版本上操作，然后隐式返回给调用者。为了避免对大型类型的这种复制，Cairo 提供了在 [第 {{#chap smart-pointers}} 章][chap-smart-pointers] 中介绍的 `Box<T>` 类型作为替代方案，但在本例中，`ref` 修饰符完美地满足了我们的需求。

程序的输出是：

```shell
$ scarb execute 
   Compiling listing_04_05 v0.1.0 (listings/ch04-understanding-ownership/listing_05_mutable_reference/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing listing_04_05
height: 10, width: 3


```

正如预期的那样，`rec` 变量的 `height` 和 `width` 字段已被交换。

{{#quiz ../quizzes/ch04-02-references-and-snapshots.toml}}

## 小结

让我们回顾一下我们讨论过的关于线性类型系统、所有权、快照和引用的内容：

- 在任何给定时间，一个变量只能有一个所有者。
- 你可以通过按值 (by-value)、按快照 (by-snapshot) 或按引用 (by-reference) 将变量传递给函数。
- 如果你按值传递，变量的所有权将转移给函数。
- 如果你想保留变量的所有权并且知道你的函数不会改变它，你可以用 `@` 将其作为快照传递。
- 如果你想保留变量的所有权并且知道你的函数会改变它，你可以用 `ref` 将其作为可变引用传递。

[chap-smart-pointers]: ./ch12-02-smart-pointers.md
