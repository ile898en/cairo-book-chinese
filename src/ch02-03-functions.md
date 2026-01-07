# 函数

函数在 Cairo 代码中随处可见。你已经见过语言中最重要的函数之一：`main` 函数，它是许多程序的入口点。你也见过 `fn` 关键字，它允许你声明新函数。

Cairo 代码使用 _蛇形命名法_ (snake case) 作为函数和变量名的常规风格，即所有字母都是小写的，下划线分隔单词。这里有一个包含示例函数定义的程序：

```cairo
fn another_function() {
    println!("Another function.");
}

#[executable]
fn main() {
    println!("Hello, world!");
    another_function();
}
```

我们在 Cairo 中定义函数是通过输入 `fn` 加上函数名和一组括号来进行的。花括号告诉编译器函数体在哪里开始和结束。

我们可以调用我们定义的任何函数，方法是输入其名称后跟一组括号。因为 `another_function` 在程序中被定义了，它可以从 `main` 函数内部被调用。注意我们在源代码中是在 `main` 函数 _之前_ 定义 `another_function` 的；我们也可以在之后定义它。Cairo 不在乎你在哪里定义你的函数，只要它们定义在一个调用者可以看到的作用域内即可。

让我们用 Scarb 启动一个名为 _functions_ 的新项目来进一步探索函数。将 `another_function` 示例放在 _src/lib.cairo_ 中并运行它。你应该看到以下输出：

```shell
$ scarb execute 
   Compiling no_listing_15_functions v0.1.0 (listings/ch02-common-programming-concepts/no_listing_15_functions/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_15_functions
Hello, world!
Another function.


```

这些行按照它们在 `main` 函数中出现的顺序执行。首先打印 `Hello, world!` 消息，然后调用 `another_function` 并打印其消息。

## 参数 (Parameters)

我们可以定义具有 _参数_ 的函数，参数是函数签名的一部分的特殊变量。当一个函数有参数时，你可以为这些参数提供具体的值。从技术上讲，具体的值被称为 _参数 (arguments)_，但在日常对话中，人们倾向于互换使用 _参数 (parameter)_ 和 _参数 (argument)_ 这两个词来指代函数定义中的变量或调用函数时传入的具体值。

在这个版本的 `another_function` 中，我们添加了一个参数：

```cairo
#[executable]
fn main() {
    another_function(5);
}

fn another_function(x: felt252) {
    println!("The value of x is: {}", x);
}
```

尝试运行这个程序；你应该得到以下输出：

```shell
$ scarb execute 
   Compiling no_listing_16_single_param v0.1.0 (listings/ch02-common-programming-concepts/no_listing_16_single_param/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_16_single_param
The value of x is: 5


```

`another_function` 的声明有一个名为 `x` 的参数。`x` 的类型被指定为 `felt252`。当我们传递 `5`给 `another_function` 时，`println!` 宏把 `5` 放在了格式字符串中包含 `x` 的那对花括号的位置。

在函数签名中，你 _必须_ 声明每个参数的类型。这是 Cairo 设计中的一个深思熟虑的决定：要求在函数定义中进行类型注解意味着编译器几乎不需要你在代码的其他地方使用它们来弄清楚你的意思是哪种类型。如果编译器知道函数期望什么类型，它也能够给出更有帮助的错误消息。

当定义多个参数时，用逗号分隔参数声明，像这样：

```cairo
#[executable]
fn main() {
    print_labeled_measurement(5, "h");
}

fn print_labeled_measurement(value: u128, unit_label: ByteArray) {
    println!("The measurement is: {value}{unit_label}");
}
```

这个例子创建了一个名为 `print_labeled_measurement` 的函数，它有两个参数。第一个参数名为 `value`，是一个 `u128`。第二个名为 `unit_label`，类型是 `ByteArray` - Cairo 用于表示字符串字面量的内部类型。然后函数打印包含 `value` 和 `unit_label` 的文本。

让我们试着运行这段代码。用前面的例子替换目前的 _functions_ 项目的 _src/lib.cairo_ 文件，并使用 `scarb execute` 运行它：

```shell
$ scarb execute 
   Compiling no_listing_17_multiple_params v0.1.0 (listings/ch02-common-programming-concepts/no_listing_17_multiple_params/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_17_multiple_params
The measurement is: 5h


```

因为我们以 `5` 作为 `value` 的值和 `"h"` 作为 `unit_label` 的值来调用函数，所以程序输出包含这些值。

### 命名参数 (Named Parameters)

在 Cairo 中，命名参数允许你在调用函数时指定参数的名称。这使得函数调用更具可读性和自描述性。如果你想使用命名参数，你需要指定参数的名称和你想要传递给它的值。语法是 `parameter_name: value`。如果你传递一个与参数同名的变量，你可以简单地写 `:parameter_name` 而不是 `parameter_name: variable_name`。

这是一个例子：

```cairo
fn foo(x: u8, y: u8) {}

#[executable]
fn main() {
    let first_arg = 3;
    let second_arg = 4;
    foo(x: first_arg, y: second_arg);
    let x = 1;
    let y = 2;
    foo(:x, :y)
}
```

## 语句和表达式 (Statements and Expressions)

函数体由一系列语句组成，可选择以一个表达式结束。到目前为止，我们涵盖的函数还没有包含结束表达式，但你已经看到过作为一个语句一部分的表达式。因为 Cairo 是一种基于表达式的语言，这是一个需要理解的重要区别。其他语言没有相同的区别，所以让我们看看语句和表达式是什么，以及它们的区别如何影响函数体。

- **语句** 是执行某些操作且不返回值的指令。
- **表达式** 计算出一个结果值。让我们看一些例子。

我们实际上已经使用过语句和表达式了。使用 `let` 关键字创建一个变量并为其赋值是一个语句。在清单 {{#ref fn-main}} 中，`let y = 6;` 是一个语句。

```cairo
#[executable]
fn main() {
    let y = 6;
}
```

{{#label fn-main}} <span class="caption">清单 {{#ref fn-main}}: 一个包含一个语句的 `main` 函数声明</span>

函数定义也是语句；整个前面的例子本身就是一个语句。

语句不返回值。因此，你不能将一个 `let` 语句赋值给另一个变量，如下面的代码试图做的那样；你会得到一个错误：

```cairo, noplayground
// TAGS: does_not_compile, ignore_fmt
#[executable]
fn main() {
    let x = (let y = 6);
}
```

当你运行这个程序时，你会得到的错误如下所示：

```shell
$ scarb execute 
   Compiling no_listing_18_statements_dont_return_values v0.1.0 (listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/Scarb.toml)
error: Missing token ')'.
 --> listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/src/lib.cairo:4:14
    let x = (let y = 6);
             ^

error: Missing token ';'.
 --> listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/src/lib.cairo:4:14
    let x = (let y = 6);
             ^

error: Missing token ';'.
 --> listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/src/lib.cairo:4:23
    let x = (let y = 6);
                      ^

error: Skipped tokens. Expected: statement.
 --> listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/src/lib.cairo:4:23
    let x = (let y = 6);
                      ^^

warn[E0001]: Unused variable. Consider ignoring by prefixing with `_`.
 --> listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/src/lib.cairo:4:9
    let x = (let y = 6);
        ^

warn[E0001]: Unused variable. Consider ignoring by prefixing with `_`.
 --> listings/ch02-common-programming-concepts/no_listing_20_statements_dont_return_values/src/lib.cairo:4:18
    let x = (let y = 6);
                 ^

error: could not compile `no_listing_18_statements_dont_return_values` due to previous error
error: `scarb` command exited with error

```

`let y = 6` 语句不返回值，所以没有什么可以让 `x` 绑定的。这与其他语言（如 C 和 Ruby）中发生的情况不同，在那些语言中赋值返回赋值的值。在那些语言中，你可以写 `x = y = 6` 并让 `x` 和 `y` 都具有值 `6`；在 Cairo 中情况并非如此。

表达式计算出一个值，并且构成了你将在 Cairo 中编写的大部分其余代码。考虑一个数学运算，如 `5 + 6`，这是一个计算结果为 `11` 的表达式。表达式可以是语句的一部分：在清单 {{#ref fn-main}} 中，语句 `let y = 6;` 中的 `6` 是一个计算结果为值 `6` 的表达式。

调用函数是一个表达式，因为它总是计算出一个值：如果指定了显式的返回值，或者是单元类型 `()`。

使用花括号创建的新作用域块是一个表达式，例如：

```cairo
#[executable]
fn main() {
    //ANCHOR: block_expr
    let y = {
        let x = 3;
        x + 1
    };
    //ANCHOR_END: block_expr

    println!("The value of y is: {}", y);
}
```

这个表达式：

```cairo, noplayground
    let y = {
        let x = 3;
        x + 1
    };
```

是一个块，在这种情况下，它的计算结果为 `4`。该值作为 `let` 语句的一部分绑定到 `y`。注意 `x + 1` 行末尾没有分号，这与你目前看到的大多数行都不一样。表达式不包含结束分号。如果你在表达式末尾添加分号，你就把它变成了一个语句，它就不会返回值了。在接下来探索函数返回值和表达式时，请记住这一点。

## 具有返回值的函数

函数可以将值返回给调用它们的代码。我们不命名返回值，但我们要必须在箭头 (`->`) 之后声明它们的类型。在 Cairo 中，函数的返回值等同于函数体块中最后一个表达式的值。你可以通过使用 `return` 关键字并指定一个值来从函数提前返回，但大多数函数隐式地返回最后一个表达式。这是一个返回值的函数的例子：

```cairo
fn five() -> u32 {
    5
}

#[executable]
fn main() {
    let x = five();
    println!("The value of x is: {}", x);
}
```

在 `five` 函数中没有函数调用，甚至没有 `let` 语句——只有数字 `5` 本身。这在 Cairo 中是一个完全有效的函数。注意函数的返回类型也被指定了，即 `-> u32`。尝试运行这段代码；输出应该看起来像这样：

```shell
$ scarb execute 
   Compiling no_listing_20_function_return_values v0.1.0 (listings/ch02-common-programming-concepts/no_listing_22_function_return_values/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_20_function_return_values
The value of x is: 5


```

`five` 中的 `5` 是函数的返回值，这就是为什么返回类型是 `u32`。让我们更详细地检查一下。有两个重要的部分：首先，`let x = five();` 这一行表明我们正在使用函数的返回值来初始化一个变量。因为函数 `five` 返回 `5`，那一行与下面是一样的：

```cairo, noplayground
let x = 5;
```

其次，`five` 函数没有参数并定义了返回值的类型，但函数体是一个孤独的 `5`，没有分号，因为它是一个我们要返回其值的表达式。让我们看另一个例子：

```cairo
#[executable]
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {}", x);
}

fn plus_one(x: u32) -> u32 {
    x + 1
}
```

运行这段代码将打印 `x = 6`。但是如果你在包含 `x + 1` 的行的末尾放置一个分号，将其从表达式更改为语句，我们会得到一个错误：

```cairo,does_not_compile
//TAG: does_not_compile

#[executable]
fn main() {
    let x = plus_one(5);

    println!("The value of x is: {}", x);
}

fn plus_one(x: u32) -> u32 {
    x + 1;
}
```

```shell
$ scarb execute 
   Compiling no_listing_22_function_return_invalid v0.1.0 (listings/ch02-common-programming-concepts/no_listing_24_function_return_invalid/Scarb.toml)
error: Unexpected return type. Expected: "core::integer::u32", found: "()".
 --> listings/ch02-common-programming-concepts/no_listing_24_function_return_invalid/src/lib.cairo:10:24
fn plus_one(x: u32) -> u32 {
                       ^^^

error: could not compile `no_listing_22_function_return_invalid` due to previous error
error: `scarb` command exited with error

```

主要错误消息 `Unexpected return type` 揭示了这段代码的核心问题。函数 `plus_one` 的定义说它将返回一个 `u32`，但语句不计算出一个值，这由 `()`（单元类型）表示。因此，没有返回任何东西，这与函数定义相矛盾并导致错误。

### Const 函数

可以在编译时计算的函数可以使用 `const fn` 语法标记为 `const`。这允许从常量上下文调用该函数，并由编译器在编译时进行解释。

将函数声明为 `const` 会限制参数和返回类型可以使用的类型，并将函数体限制为常量表达式。

核心库中的几个函数被标记为 `const`。这是一个来自核心库的例子，展示了实现为 `const fn` 的 `pow` 函数：

```cairo
use core::num::traits::Pow;

const BYTE_MASK: u16 = 2_u16.pow(8) - 1;

#[executable]
fn main() {
    let my_value = 12345;
    let first_byte = my_value & BYTE_MASK;
    println!("first_byte: {}", first_byte);
}
```

在这个例子中，`pow` 是一个 `const` 函数，允许它在常量表达式中用于在编译时定义 `mask`。这是核心库中如何使用 `const fn` 定义 `pow` 的片段：

注意，将函数声明为 `const` 对现有用法没有影响；它只对常量上下文施加限制。

{{#quiz ../quizzes/ch02-03-functions.toml}}
