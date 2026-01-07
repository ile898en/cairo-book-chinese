# 使用线性类型系统的所有权

Cairo 使用线性类型系统。在这种类型系统中，任何值（基本类型、结构体、枚举）必须被使用，而且只能被使用一次。这里的“使用”意味着该值要么 _被销毁 (destroyed)_，要么 _被移动 (moved)_。

_销毁_ 可以通过几种方式发生：

- 变量超出作用域。
- 结构体被解构。
- 使用 `destruct()` 显式销毁。

_移动_ 一个值仅仅意味着将该值传递给另一个函数。

这导致了与 Rust 所有权模型有些相似的约束，但也存在一些差异。特别是，Rust 所有权模型之所以存在，（部分）是为了避免数据竞争和对内存值的并发可变访问。这在 Cairo 中显然是不可能的，因为内存是不可变的。相反，Cairo 利用其线性类型系统主要用于两个目的：

- 确保所有代码都是可证明的，因此是可验证的。
- 抽象掉 Cairo VM 的不可变内存。

### 所有权 (Ownership)

在 Cairo 中，所有权适用于 _变量_ 而不适用于 _值_。一个值可以被许多不同的变量安全地引用（即使它们是可变变量），因为值本身总是不可变的。然而，变量可以是可变的，因此编译器必须确保常量变量不会被程序员意外修改。这使得谈论变量的所有权成为可能：所有者是可以读取（如果是可变的，则可以写入）该变量的代码。

这意味着变量（而非值）遵循与 Rust 值类似的规则：

- Cairo 中的每个变量都有一个所有者。
- 一次只能有一个所有者。
- 当所有者超出作用域时，变量将被销毁。

现在我们已经过了基础的 Cairo 语法阶段，我们不会在示例中包含所有的 `fn main() {` 代码，所以如果你在跟着做，请确保手动将以下示例放在 main 函数内。结果是，我们的例子将更加简洁，让我们专注于实际细节而不是样板代码。

## 变量作用域 (Variable Scope)

作为线性类型系统的第一个例子，我们将看一些变量的 _作用域_。作用域是程序中项目有效的范围。以这个变量为例：

```cairo,noplayground
let s = 'hello';
```

变量 `s` 引用一个短字符串。该变量从声明点开始直到当前 _作用域_ 结束都是有效的。清单 {{#ref variable-scope}} 展示了一个带有注释的程序，注释说明了变量 `s` 何时有效。

```cairo
    { // s is not valid here, it’s not yet declared
        let s = 'hello'; // s is valid from this point forward
        // do stuff with s
    } // this scope is now over, and s is no longer valid
```

{{#label variable-scope}} <span class="caption">清单 {{#ref variable-scope}}:
一个变量及其有效的作用域</span>

换句话说，这里有两个重要时间点：

- 当 `s` _进入_ 作用域时，它是有效的。
- 它保持有效，直到它 _超出_ 作用域。

在这一点上，作用域与变量何时有效之间的关系与其他编程语言中的类似。现在我们将基于这一理解，使用我们在之前的 [“数组”][array] 部分介绍的 `Array` 类型。

[array]: ./ch03-01-arrays.md

### 移动值 (Moving values)

如前所述，_移动_ 一个值仅仅意味着将该值传递给另一个函数。当这种情况发生时，原始作用域中引用该值的变量将被销毁并且不再可用，并且会创建一个新变量来持有相同的值。

数组是一个复杂类型的例子，当将其传递给另一个函数时会被移动。这是一个关于数组长什么样的简短提醒：

```cairo
<!-- Warning: Anchor '2' not found in lib.cairo -->
```

类型系统如何确保 Cairo 程序永远不会尝试两次写入同一个内存单元？考虑下面的代码，我们尝试两次移除数组的前端：

```cairo,does_not_compile
//TAG: does_not_run
fn foo(mut arr: Array<u128>) {
    arr.pop_front();
}

#[executable]
fn main() {
    let arr: Array<u128> = array![];
    foo(arr);
    foo(arr);
}
```

在这种情况下，我们试图将相同的值（`arr` 变量中的数组）传递给两个函数调用。这意味着我们的代码试图两次移除第一个元素，这将会尝试两次写入同一个内存单元——这是 Cairo VM 禁止的，会导致运行时错误。值得庆幸的是，这段代码实际上不能编译。一旦我们将数组传递给 `foo` 函数，变量 `arr` 就不再可用了。我们会得到这个编译时错误，告诉我们需要 Array 实现 Copy Trait：

```shell
$ scarb execute 
   Compiling no_listing_02_pass_array_by_value v0.1.0 (listings/ch04-understanding-ownership/no_listing_02_pass_array_by_value/Scarb.toml)
warn: Unhandled `#[must_use]` type `core::option::Option::<core::integer::u128>`
 --> listings/ch04-understanding-ownership/no_listing_02_pass_array_by_value/src/lib.cairo:3:5
    arr.pop_front();
    ^^^^^^^^^^^^^^^

error: Variable was previously moved.
 --> listings/ch04-understanding-ownership/no_listing_02_pass_array_by_value/src/lib.cairo:10:9
    foo(arr);
        ^^^
note: variable was previously used here:
  --> listings/ch04-understanding-ownership/no_listing_02_pass_array_by_value/src/lib.cairo:9:9
    foo(arr);
        ^^^
note: Trait has no implementation in context: core::traits::Copy::<core::array::Array::<core::integer::u128>>.

error: could not compile `no_listing_02_pass_array_by_value` due to previous error
error: `scarb` command exited with error

```

## `Copy` Trait

`Copy` trait 允许简单类型通过复制 felt 来复制，而无需分配新的内存段。这与 Cairo 默认的“移动”语义形成对比，后者转移值的所有权以确保内存安全并防止诸如对同一内存单元进行多次写入等问题。`Copy` 是为复制既安全又高效的类型实现的，绕过了移动语义的需要。像 `Array` 和 `Felt252Dict` 这样的类型不能实现 `Copy`，因为类型系统禁止在不同作用域中操作它们。

之前在 [“数据类型”][data types] 中描述的所有基本类型默认都实现了 `Copy` trait。

虽然数组和字典不能被复制，但不包含它们的自定义类型可以被复制。你可以通过在类型定义中添加 `#[derive(Copy)]` 注解来实现 `Copy` trait。然而，如果类型本身或其任何组件未实现 Copy trait，Cairo 将不允许该类型被注解为 Copy。

```cairo,ignore_format
#[derive(Copy, Drop)]
struct Point {
    x: u128,
    y: u128,
}

#[executable]
fn main() {
    let p1 = Point { x: 5, y: 10 };
    foo(p1);
    foo(p1);
}

fn foo(p: Point) { // do something with p
}
```

在这个例子中，我们可以将 `p1` 传递给 foo 函数两次，因为 `Point` 类型实现了 `Copy` trait。这意味着当我们把 `p1` 传递给 `foo` 时，我们实际上是在传递 `p1` 的副本，所以 `p1` 仍然有效。用所有权术语来说，这意味着 `p1` 的所有权仍然保留在 `main` 函数中。如果你从 `Point` 类型中移除 `Copy` trait 派生，你在尝试编译代码时会得到编译时错误。

_不用担心 `Struct` 关键字。我们将在 [第 {{#chap using-structs-to-structure-related-data}} 章][structs] 中介绍它。_

[data types]: ./ch02-02-data-types.md
[structs]: ./ch05-00-using-structs-to-structure-related-data.md

## 销毁值 - FeltDict 示例

线性类型被 _使用_ 的另一种方式是被销毁。销毁必须确保‘资源’现在被正确释放。例如，在 Rust 中，这可能是关闭文件访问或锁定互斥锁。在 Cairo 中，具有这种行为的一种类型是 `Felt252Dict`。为了可证明性，字典在销毁时必须被‘压缩 (squashed)’。这很容易被忘记，所以它由类型系统和编译器强制执行。

### 无操作销毁 (No-op Destruction): `Drop` Trait

你可能已经注意到前面例子中的 `Point` 类型也实现了 `Drop` trait。例如，下面的代码将无法编译，因为结构体 `A` 在超出作用域之前没有被移动或销毁：

```cairo,does_not_compile
//TAG: does_not_compile
struct A {}

#[executable]
fn main() {
    A {}; // error: Variable not dropped.
}
```

然而，实现了 `Drop` trait 的类型在超出作用域时会自动销毁。这种销毁什么也不做，它是无操作 (no-op)——仅仅是对编译器的提示，表明此类型一旦不再有用就可以安全销毁。我们称之为“丢弃 (dropping)”一个值。

目前，可以为所有类型派生 `Drop` 实现，允许它们在超出作用域时被丢弃，除了字典 (`Felt252Dict`) 和包含字典的类型。例如，下面的代码可以编译：

```cairo
#[derive(Drop)]
struct A {}

#[executable]
fn main() {
    A {}; // Now there is no error.
}
```

### 有副作用的销毁: `Destruct` Trait

当一个值被销毁时，编译器首先尝试在该类型上调用 `drop` 方法。如果它不存在，编译器则尝试调用 `destruct`。此方法由 `Destruct` trait 提供。

如前所述，Cairo 中的字典是在销毁时必须被“压缩”的类型，以便可以证明访问顺序。这对开发者来说很容易忘记，所以字典实现了 `Destruct` trait 以确保所有字典在超出作用域时都被 _压缩_。因此，以下示例将无法编译：

```cairo,does_not_compile
//TAG: does_not_compile
use core::dict::Felt252Dict;

struct A {
    dict: Felt252Dict<u128>,
}

#[executable]
fn main() {
    A { dict: Default::default() };
}
```

如果你尝试运行这段代码，你会得到一个编译时错误：

```shell
$ scarb execute 
   Compiling no_listing_06_no_destruct_compile_fails v0.1.0 (listings/ch04-understanding-ownership/no_listing_06_no_destruct_compile_fails/Scarb.toml)
error: Variable not dropped.
 --> listings/ch04-understanding-ownership/no_listing_06_no_destruct_compile_fails/src/lib.cairo:10:5
    A { dict: Default::default() };
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
note: Trait has no implementation in context: core::traits::Drop::<no_listing_06_no_destruct_compile_fails::A>.
note: Trait has no implementation in context: core::traits::Destruct::<no_listing_06_no_destruct_compile_fails::A>.

error: could not compile `no_listing_06_no_destruct_compile_fails` due to previous error
error: `scarb` command exited with error

```

当 `A` 超出作用域时，它不能被丢弃，因为它既没有实现 `Drop`（因为它包含一个字典且不能 `derive(Drop)`）也没有实现 `Destruct` trait。为了修复这个问题，我们可以为 `A` 类型派生 `Destruct` trait 实现：

```cairo
use core::dict::Felt252Dict;

#[derive(Destruct)]
struct A {
    dict: Felt252Dict<u128>,
}

#[executable]
fn main() {
    A { dict: Default::default() }; // No error here
}
```

现在，当 `A` 超出作用域时，其字典将自动被 `squashed`，程序将可以编译。

## 使用 `clone` 复制数组数据

如果我们 _确实_ 想要深度复制 `Array` 的数据，我们可以使用一个称为 `clone` 的常用方法。我们将在 [第 {{#chap using-structs-to-structure-related-data}} 章][method syntax] 的专门部分讨论方法语法，但因为方法是许多编程语言中的常见特性，你可能以前见过它们。

这是一个 `clone` 方法的实际例子。

```cairo
#[executable]
fn main() {
    let arr1: Array<u128> = array![];
    let arr2 = arr1.clone();
}
```

当你看到对 `clone` 的调用时，你知道正在执行一些任意代码，并且该代码可能很昂贵。这是一个视觉指示器，表明正在发生一些不同的事情。在这种情况下，`arr1` 引用的 _值_ 被复制，导致使用了新的内存单元，并且创建了一个新的 _变量_ `arr2`，引用新复制的值。

[method syntax]: ./ch05-03-method-syntax.md

## 返回值和作用域

返回值等同于 _移动_ 它们。清单 {{#ref move-return-values}} 展示了一个返回某个值的函数示例，其注释与清单 {{#ref variable-scope}} 中的类似。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
//TAG: ignore_fmt
#[derive(Drop)]
struct A {}

#[executable]
fn main() {
    let a1 = gives_ownership();           // gives_ownership moves its return
                                          // value into a1

    let a2 = A {};                        // a2 comes into scope

    let a3 = takes_and_gives_back(a2);    // a2 is moved into
                                          // takes_and_gives_back, which also
                                          // moves its return value into a3

} // Here, a3 goes out of scope and is dropped. a2 was moved, so nothing
  // happens. a1 goes out of scope and is dropped.

fn gives_ownership() -> A {               // gives_ownership will move its
                                          // return value into the function
                                          // that calls it

    let some_a = A {};                    // some_a comes into scope

    some_a                                // some_a is returned and
                                          // moves ownership to the calling
                                          // function
}

// This function takes an instance some_a of A and returns it
fn takes_and_gives_back(some_a: A) -> A { // some_a comes into scope

    some_a                                // some_a is returned and
                                          // moves ownership to the calling
                                          // function
}
```

{{#label move-return-values}} <span class="caption">清单
{{#ref move-return-values}}: 移动返回值</span>

虽然这行得通，但在每个函数中移入和移出都有些繁琐。如果我们想让一个函数使用一个值但不移动该值怎么办？如果我们想再次使用它，除了我们可能想作为返回值返回的函数体产生的任何数据之外，我们传入的任何东西也需要被传回，这非常烦人。

Cairo 确实允许我们使用元组返回多个值，如清单 {{#ref return-multiple-values}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[executable]
fn main() {
    let arr1: Array<u128> = array![];

    let (arr2, len) = calculate_length(arr1);
}

fn calculate_length(arr: Array<u128>) -> (Array<u128>, usize) {
    let length = arr.len(); // len() returns the length of an array

    (arr, length)
}
```

{{#label return-multiple-values}} <span class="caption">清单
{{#ref return-multiple-values}}: 返回多个值</span>

但这对于一个应该是通用的概念来说太繁琐了，而且要做很多工作。对我们来说幸运的是，Cairo 有两个特性可以通过不销毁或移动值来传递值，称为 _引用_ 和 _快照_。
