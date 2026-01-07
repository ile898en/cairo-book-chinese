# 变量与可变性

Cairo 使用不可变内存模型，这意味着一旦一个内存单元被写入，它就不能被覆盖，只能被读取。为了反映这种不可变内存模型，Cairo 中的变量默认是不可变的。然而，语言抽象了这个模型，让你有选择地使变量可变。让我们探索一下 Cairo 是如何以及为什么强制实施不可变性的，以及你如何能让你的变量变得通过可变。

当一个变量是不可变的，一旦一个值被绑定到一个名字上，你就不能改变那个值。为了说明这一点，使用 `scarb new variables` 在你的 _cairo_projects_ 目录下生成一个名为 _variables_ 的新项目。

然后，在你的新 _variables_ 目录下，打开 _src/lib.cairo_ 并将其代码替换为以下代码，这些代码目前还不能编译：

<span class="filename">文件名: src/lib.cairo</span>

```cairo,does_not_compile
//TAG: does_not_compile

#[executable]
fn main() {
    let x = 5;
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}

```

使用 `scarb execute` 保存并运行程序。你应该会收到一条关于不可变性错误的错误消息，如下面的输出所示：

```shell
$ scarb execute 
   Compiling no_listing_01_variables_are_immutable v0.1.0 (listings/ch02-common-programming-concepts/no_listing_01_variables_are_immutable/Scarb.toml)
error: Cannot assign to an immutable variable.
 --> listings/ch02-common-programming-concepts/no_listing_01_variables_are_immutable/src/lib.cairo:7:5
    x = 6;
    ^^^^^

error: could not compile `no_listing_01_variables_are_immutable` due to previous error
error: `scarb` command exited with error

```

这个例子展示了编译器如何帮助你在程序中发现错误。编译器错误可能令人沮丧，但它们只意味着你的程序还没有安全地做你想让它做的事情；它们 _不_ 意味着你不是一个好程序员！经验丰富的 Cairo 宇航员（Caironautes）仍然会收到编译器错误。

你收到了错误消息 `Cannot assign to an immutable variable.`（不能赋值给不可变变量），因为你试图给不可变的 `x` 变量赋第二个值。

当我们试图改变一个被指定为不可变的值时，获得编译时错误是很重要的，因为这种特定情况可能导致 bug。如果我们的代码的一部分假设一个值永远不会改变，而代码的另一部分改变了那个值，那么代码的第一部分可能就不会做它被设计要做的事情。这种 bug 的原因在事后可能很难追踪，特别是当第二段代码只是 _有时_ 改变值的时候。

Cairo 与大多数其他语言不同，它拥有不可变内存。这使得整类 bug 变得不可能，因为值永远不会意外地改变。这使得代码更容易推理。

但是可变性可能非常有用，并且可以使代码编写起来更方便。虽然变量默认是不可变的，但你可以通过在变量名前添加 `mut` 来使它们可变。添加 `mut` 也向代码的未来读者传达了意图，表明代码的其他部分将改变与此变量关联的值。

<!-- TODO: add an illustration of this -->

然而，你此刻可能会想，当一个变量被声明为 `mut` 时究竟发生了什么，因为我们之前提到 Cairo 的内存是不可变的。答案是 _值_ 是不可变的，但 _变量_ 不是。与变量关联的值可以更改。在 Cairo 中赋值给一个可变变量本质上等同于重新声明它以引用另一个内存单元中的另一个值，但编译器为你处理了这个过程，而关键字 `mut` 使其显式化。在检查低级 Cairo 汇编代码时，很明显变量突变是作为语法糖实现的，它将突变操作转换为一系列等同于变量遮蔽（shadowing）的步骤。唯一的区别是，在 Cairo 层面，变量没有被重新声明，所以它的类型不能改变。

例如，让我们将 _src/lib.cairo_ 更改为以下内容：

```cairo
#[executable]
fn main() {
    let mut x = 5;
    println!("The value of x is: {}", x);
    x = 6;
    println!("The value of x is: {}", x);
}
```

当我们现在运行程序时，我们会得到这个：

```shell
$ scarb execute 
   Compiling no_listing_02_adding_mut v0.1.0 (listings/ch02-common-programming-concepts/no_listing_02_adding_mut/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_02_adding_mut
The value of x is: 5
The value of x is: 6


```

当使用 `mut` 时，我们被允许将绑定到 `x` 的值从 `5` 更改为 `6`。最终，决定是否使用可变性取决于你，并且取决于你认为在特定情况下哪种方式最清晰。

## 常量 (Constants)

像不可变变量一样，_常量_ 是绑定到名称且不允许更改的值，但常量和变量之间有一些区别。

首先，你不允许使用 `mut` 与常量一起使用。常量不仅默认是不可变的——它们总是不可变的。你使用 `const` 关键字而不是 `let` 关键字来声明常量，并且值的类型 _必须_ 被注解。我们将在下一节 [“数据类型”][data-types] 中介绍类型和类型注解，所以现在不用担心细节。只要知道你必须始终注解类型即可。

常量变量可以使用任何通常的数据类型声明，包括结构体、枚举和固定大小的数组。

常量只能在全局作用域中声明，这使得它们对于代码的许多部分都需要知道的值非常有用。

最后一个区别是，常量本身只能被设置为一个常量表达式，而不能是只能在运行时计算出的数值结果。

这是一个常量声明的例子：

```cairo,noplayground
struct AnyStruct {
    a: u256,
    b: u32,
}

enum AnyEnum {
    A: felt252,
    B: (usize, u256),
}

const ONE_HOUR_IN_SECONDS: u32 = 3600;
const ONE_HOUR_IN_SECONDS_2: u32 = 60 * 60;
const STRUCT_INSTANCE: AnyStruct = AnyStruct { a: 0, b: 1 };
const ENUM_INSTANCE: AnyEnum = AnyEnum::A('any enum');
const BOOL_FIXED_SIZE_ARRAY: [bool; 2] = [true, false];
```

Cairo 的常量命名约定是使用全大写字母并在单词之间加下划线。

常量在程序运行的整个期间内，在声明它们的作用域内都是有效的。这个属性使得常量对于你的应用程序领域中多个部分可能都需要知道的值有用，例如游戏玩家允许赚取的最大积分数，或者光速。

将程序中各处使用的硬编码值命名为常量，对于向代码的未来维护者传达该值的含义非常有用。如果在将来如果需要更新硬编码值，你也只需要在代码中的一个地方进行更改。

[data-types]: ./ch02-02-data-types.md

## 遮蔽 (Shadowing)

变量遮蔽是指声明一个与前一个变量同名的新变量。Cairo 宇航员（Caironautes）说第一个变量被第二个变量 _遮蔽_ 了，这意味着当你使用变量的名称时，编译器看到的是第二个变量。实际上，第二个变量掩盖了第一个变量，将该变量名的任何使用都归于自身，直到它自己被遮蔽或作用域结束。我们可以通过使用相同的变量名并重复使用 `let` 关键字来遮蔽一个变量，如下所示：

```cairo
#[executable]
fn main() {
    let x = 5;
    let x = x + 1;
    {
        let x = x * 2;
        println!("Inner scope x value is: {}", x);
    }
    println!("Outer scope x value is: {}", x);
}
```

这个程序首先将 `x` 绑定到值 `5`。然后它通过重复 `let x =` 创建了一个新变量 `x`，取原始值并加 `1`，所以 `x` 的值随后变成了 `6`。然后，在使用花括号创建的内部作用域内，第三个 `let` 语句也遮蔽了 `x` 并创建了一个新变量，将之前的值乘以 `2`，使 `x` 的值为 `12`。当该作用域结束时，内部遮蔽结束，`x` 回到 `6`。当我们运行这个程序时，它将输出以下内容：

```shell
$ scarb execute 
   Compiling no_listing_03_shadowing v0.1.0 (listings/ch02-common-programming-concepts/no_listing_03_shadowing/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_03_shadowing
Inner scope x value is: 12
Outer scope x value is: 6


```

遮蔽与将变量标记为 `mut` 不同，因为如果我们不小心尝试在不使用 `let` 关键字的情况下重新赋值给这个变量，我们会得到一个编译时错误。通过使用 `let`，我们可以对一个值进行一些转换，但在这些转换完成后，变量仍然是不可变的。

`mut` 和遮蔽之间的另一个区别是，当我们再次使用 `let` 关键字时，我们实际上是创建了一个新变量，这允许我们在复用相同名称的同时更改值的类型。如前所述，变量遮蔽和可变变量在底层是等价的。唯一的区别是，通过遮蔽变量，如果你改变它的类型，编译器不会抱怨。例如，假设我们的程序在 `u64` 和 `felt252` 类型之间进行类型转换。

```cairo
#[executable]
fn main() {
    let x: u64 = 2;
    println!("The value of x is {} of type u64", x);
    let x: felt252 = x.into(); // converts x to a felt, type annotation is required.
    println!("The value of x is {} of type felt252", x);
}
```

第一个 `x` 变量具有 `u64` 类型，而第二个 `x` 变量具有 `felt252` 类型。因此，遮蔽使我们不必想出不同的名称，如 `x_u64` 和 `x_felt252`；相反，我们可以复用更简单的 `x` 名称。然而，如果我们尝试使用 `mut` 来做这件事，如下所示，我们将得到一个编译时错误：

```cairo,does_not_compile
//TAG: does_not_compile

#[executable]
fn main() {
    let mut x: u64 = 2;
    println!("The value of x is: {}", x);
    x = 5_u8;
    println!("The value of x is: {}", x);
}
```

错误说我们期望一个 `u64`（原始类型），但我们得到了一个不同的类型：

```shell
$ scarb execute 
   Compiling no_listing_05_mut_cant_change_type v0.1.0 (listings/ch02-common-programming-concepts/no_listing_05_mut_cant_change_type/Scarb.toml)
error: Unexpected argument type. Expected: "core::integer::u64", found: "core::integer::u8".
 --> listings/ch02-common-programming-concepts/no_listing_05_mut_cant_change_type/src/lib.cairo:7:9
    x = 5_u8;
        ^^^^

error: could not compile `no_listing_05_mut_cant_change_type` due to previous error
error: `scarb` command exited with error

```

{{#quiz ../quizzes/ch02-01-variables-and-mutability.toml}}

既然我们已经探索了变量是如何工作的，让我们看看它们可以拥有的更多数据类型。
