# 数据类型

Cairo 中的每个值都属于某种 _数据类型_，这告诉 Cairo 指定的是哪种数据，以便它知道如何处理该数据。本节涵盖两个数据类型子集：标量（scalars）和复合（compounds）。

请记住，Cairo 是一种 _静态类型_ 语言，这意味着它必须在编译时知道所有变量的类型。编译器通常可以根据值及其用法推断出所需的类型。在许多类型都可能的情况下，我们可以使用一种转换方法，在其中指定所需的输出类型。

```cairo
#[executable]
fn main() {
    let x: felt252 = 3;
    let y: u32 = x.try_into().unwrap();
}
```

你会看到其他数据类型的不同类型注解。

## 标量类型 (Scalar Types)

_标量_ 类型代表单个值。Cairo 有三种主要的标量类型：felt（域元素）、整数和布尔值。你可能从其他编程语言中认出这些。让我们通过它们在 Cairo 中是如何工作的。

### Felt 类型

在 Cairo 中，如果你不指定变量或参数的类型，其类型默认为域元素（field element），由关键字 `felt252` 表示。在 Cairo 的语境中，当我们说“一个域元素”时，我们指的是范围 \\( 0 \leq x < P \\) 内的一个整数，其中 \\( P \\) 是一个非常大的质数，当前等于 \\( {2^{251}} + 17 \cdot {2^{192}} + 1 \\)。在进行加法、减法或乘法运算时，如果结果超出了指定质数的范围，就会发生溢出（或下溢），并加上或减去 \\( P \\) 的适当倍数以使结果回到范围内（即，结果是按 \\( \mod P \\) 计算的）。

整数和域元素之间最重要的区别是除法：域元素的除法（以及因此 Cairo 中的除法）不同于常规 CPU 除法，常规整数除法 \\( \frac{x}{y} \\) 定义为 \\( \left\lfloor \frac{x}{y} \right\rfloor \\)，返回商的整数部分（所以你会得到 \\( \frac{7}{3} = 2 \\)），它可能满足也可能不满足方程 \\( \frac{x}{y} \cdot y == x \\)，这取决于 `x` 是否能被 `y` 整除。

在 Cairo 中，\\( \frac{x}{y} \\) 的结果被定义为总是满足方程 \\( \frac{x}{y} \cdot y == x \\)。如果 y 能整除 x（作为整数），你将在 Cairo 中得到预期的结果（例如 \\( \frac{6}{2} \\) 确实会得到 `3`）。但是当 y 不能整除 x 时，你可能会得到一个令人惊讶的结果：例如，因为 \\( 2 \cdot \frac{P + 1}{2} = P + 1 \equiv 1 \mod P \\)，在 Cairo 中 \\( \frac{1}{2} \\) 的值是 \\( \frac{P + 1}{2} \\)（而不是 0 或 0.5），因为它满足上述方程。

### 整数类型 (Integer Types)

`felt252` 类型是一个基础类型，作为创建核心库中所有类型的基础。然而，强烈建议程序员尽可能使用整数类型而不是 `felt252` 类型，因为 `integer` 类型带有额外的安全特性，可以提供额外的保护，防止代码中的潜在漏洞，如溢出和下溢检查。通过使用这些整数类型，程序员可以确保他们的程序更安全，更不容易受到攻击或其他安全威胁。一个 `integer` 是一个没有小数部分的数字。这种类型声明表明了程序员可以用来存储整数的位数。表 3-1 展示了 Cairo 中的内置整数类型。我们可以使用这些变体中的任何一个来声明整数值的类型。

| 长度    | 无符号   |
| ------- | -------- |
| 8-bit   | `u8`     |
| 16-bit  | `u16`    |
| 32-bit  | `u32`    |
| 64-bit  | `u64`    |
| 128-bit | `u128`   |
| 256-bit | `u256`   |
| 32-bit  | `usize`  |

<br>
<div align="center"><span class="caption">表 3-1: Cairo 中的整数类型。</span></div>

每个变体都有明确的大小。注意，目前 `usize` 类型只是 `u32` 的别名；然而，当将来 Cairo 可以编译为 MLIR 时，它可能会很有用。由于变量是无符号的，它们不能包含负数。这段代码会导致程序 panic：

```cairo
// TAG: does_not_run

fn sub_u8s(x: u8, y: u8) -> u8 {
    x - y
}

#[executable]
fn main() {
    sub_u8s(1, 3);
}
```

所有前面提到的整数类型都可以放入一个 `felt252` 中，只有 `u256` 除外，它需要多 4 位来存储。在底层，`u256` 基本上是一个有 2 个字段的结构体：`u256 {low: u128, high: u128}`。

Cairo 还提供对有符号整数的支持，以前缀 `i` 开头。这些整数可以表示正值和负值，大小范围从 `i8` 到 `i128`。每个有符号变体可以存储从 \\( -({2^{n - 1}}) \\) 到 \\( {2^{n - 1}} - 1 \\)（含）的数字，其中 `n` 是该变体使用的位数。所以一个 `i8` 可以存储从 \\( -({2^7}) \\) 到 \\( {2^7} - 1 \\) 的数字，即 `-128` 到 `127`。

你可以以表 3-2 中所示的任何形式编写整数文字。注意，可以是多种数字类型的数字文字允许使用类型后缀，例如 `57_u8`，来指定类型。也可以使用视觉分隔符 `_` 来表示数字文字，以提高代码的可读性。

| 数字文字 (Numeric literals) | 例子      |
| --------------------------- | --------- |
| 十进制 (Decimal)            | `98222`   |
| 十六进制 (Hex)              | `0xff`    |
| 八进制 (Octal)              | `0o04321` |
| 二进制 (Binary)             | `0b01`    |

<br>
<div align="center"><span class="caption">表 3-2: Cairo 中的整数文字。</span></div>

那么你怎么知道该使用哪种类型的整数呢？试着估计你的 int 可以拥有的最大值并选择合适的大小。你会使用 `usize` 的主要情况是在索引某种类型的集合时。

### 数值运算

Cairo 支持你期望的所有整数类型的基本数学运算：加法、减法、乘法、除法和余数。整数除法向零截断到最接近的整数。以下代码显示了如何在 `let` 语句中使用每个数值运算：

```cairo
#[executable]
fn main() {
    // addition
    let sum = 5_u128 + 10_u128;

    // subtraction
    let difference = 95_u128 - 4_u128;

    // multiplication
    let product = 4_u128 * 30_u128;

    // division
    let quotient = 56_u128 / 32_u128; //result is 1
    let quotient = 64_u128 / 32_u128; //result is 2

    // remainder
    let remainder = 43_u128 % 5_u128; // result is 3
}
```

这些语句中的每个表达式都使用了一个数学运算符并计算为一个值，然后绑定到一个变量。

[附录 B][operators] 包含 Cairo 提供的所有运算符的列表。

[operators]: ./appendix-02-operators-and-symbols.md#operators

### 布尔类型 (Boolean)

与大多数其他编程语言一样，Cairo 中的布尔类型有两个可能的值：`true` 和 `false`。布尔值的大小为一个 `felt252`。Cairo 中的布尔类型使用 `bool` 指定。例如：

```cairo
#[executable]
fn main() {
    let t = true;

    let f: bool = false; // with explicit type annotation
}
```

当声明一个 `bool` 变量时，必须使用 `true` 或 `false` 字面量作为值。因此，不允许使用整数文字（即用 `0` 代替 false）进行 `bool` 声明。

使用布尔值的主要方式是通过条件语句，例如 `if` 表达式。我们将在 ["控制流"][control-flow] 部分介绍 `if` 表达式在 Cairo 中是如何工作的。

[control-flow]: ./ch02-05-control-flow.md

### 字符串类型 (String Types)

Cairo 没有原生的字符串类型，但提供了两种处理它们的方法：使用单引号的短字符串和使用双引号的 ByteArray。

#### 短字符串 (Short strings)

短字符串是一个 ASCII 字符串，其中每个字符编码在一个字节上（见 [ASCII 表][ascii]）。例如：

- `'a'` 等价于 `0x61`
- `'b'` 等价于 `0x62`
- `'c'` 等价于 `0x63`
- `0x616263` 等价于 `'abc'`。

Cairo 使用 `felt252` 来表示短字符串。由于 `felt252` 是 251 位的，一个短字符串限制为 31 个字符（31 \* 8 = 248 位，这是适合 251 位的最大 8 的倍数）。

你可以选择用十六进制值如 `0x616263` 来表示你的短字符串，或者直接使用单引号写字符串如 `'abc'`，这更方便。

以下是在 Cairo 中声明短字符串的一些示例：

```cairo
<!-- Warning: Anchor '2' not found in lib.cairo -->
```

[ascii]: https://www.asciitable.com/

#### 字节数组字符串 (ByteArray Strings)

Cairo 的核心库提供了一个 `ByteArray` 类型，用于处理比短字符串更长的字符串和字节序列。这种类型对于较长的字符串或当你需要对字符串数据执行操作时特别有用。

Cairo 中的 `ByteArray` 实现为两部分的组合：

1. 一个 `bytes31` 字的数组，其中每个字包含 31 字节的数据。
2. 一个待处理的 `felt252` 字，作为一个缓冲区，用于存放尚未填满一个完整 `bytes31` 字的字节。

这种设计使得能够高效地处理字节序列，同时与 Cairo 的内存模型和基本类型保持一致。开发者通过其提供的方法和运算符与 `ByteArray` 交互，以此抽象了内部实现细节。

与短字符串不同，`ByteArray` 字符串可以包含超过 31 个字符，并且使用双引号编写：

```cairo
<!-- Warning: Anchor '8' not found in lib.cairo -->
```

## 复合类型 (Compound Types)

### 元组类型 (Tuple)

_元组_ 是一种将多种类型的多个值组合成一个复合类型的通用方法。元组具有固定的长度：一旦声明，它们的大小就不能增长或缩小。

我们通过在括号内写一个逗号分隔的值列表来创建一个元组。元组中的每个位置都有一个类型，元组中不同值的类型不必相同。在这个例子中我们添加了可选的类型注解：

```cairo
#[executable]
fn main() {
    let tup: (u32, u64, bool) = (10, 20, true);
}
```

变量 `tup` 绑定到整个元组，因为元组被视为单个复合元素。为了从元组中取出单个值，我们可以使用模式匹配来解构元组值，像这样：

```cairo
#[executable]
fn main() {
    let tup = (500, 6, true);

    let (x, y, z) = tup;

    if y == 6 {
        println!("y is 6!");
    }
}
```

这个程序首先创建一个元组并将其绑定到变量 `tup`。然后它使用带有 `let` 的模式将 `tup` 变成三个独立的变量，`x`，`y` 和 `z`。这被称为 _解构_（destructuring），因为它将单个元组分解成三个部分。最后，程序打印 `y is 6!`，因为 `y` 的值是 `6`。

我们也可以用值和类型声明元组，并同时解构它。例如：

```cairo
#[executable]
fn main() {
    let (x, y): (felt252, felt252) = (2, 3);
}
```

#### 单元类型 () (Unit Type)

_单元类型_ 是一种只有一个值 `()` 的类型。它由一个没有元素的元组表示。它的大小总是零，并且保证在编译的代码中不存在。

你可能想知道为什么你甚至需要一个单元类型？在 Cairo 中，一切都是表达式，返回空的表达式实际上隐式地返回 `()`。

### 固定大小数组类型 (Fixed Size Array)

另一种拥有多个值集合的方法是使用 _固定大小数组_。与元组不同，固定大小数组的每个元素必须具有相同的类型。

我们将固定大小数组中的值写成方括号内逗号分隔的列表。数组的类型是使用方括号写的，包含每个元素的类型、一个分号，然后是数组中的元素数量，像这样：

```cairo
#[executable]
fn main() {
    let arr1: [u64; 5] = [1, 2, 3, 4, 5];
}
```

在类型注解 `[u64; 5]` 中，`u64` 指定了每个元素的类型，而分号后的 `5` 定义了数组的长度。这种语法确保数组总是恰好包含 5 个类型为 `u64` 的元素。

当你想要在程序中直接硬编码一个可能很长的数据序列时，固定大小数组很有用。这种类型的数组不能与 [`Array<T>` 类型][arrays] 混淆，后者是核心库提供的相似集合类型，但 _被允许_ 增长大小。如果你不确定是使用固定大小数组还是 `Array<T>` 类型，你很可能正在寻找 `Array<T>` 类型。

因为它们的大小在编译时是已知的，固定大小数组不需要运行时内存管理，这使得它们比动态大小的数组更高效。总的来说，当你确定元素数量不需要改变时，它们更有用。例如，它们可以用来高效地存储在运行时不会改变的查找表。如果你在程序中使用月份的名字，你可能会使用固定大小数组而不是 `Array<T>`，因为你知道它总是包含 12 个元素：

```cairo
    let months = [
        'January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September',
        'October', 'November', 'December',
    ];

```

你也可以初始化一个数组，使其每个元素包含相同的值，方法是指定初始值，后跟分号，然后是方括号中的数组长度，如下所示：

```cairo
    let a = [3; 5];
```

名为 `a` 的数组将包含 `5` 个元素，它们最初都将被设置为值 `3`。这与编写 `let a = [3, 3, 3, 3, 3];` 是一样的，但方式更简洁。

#### 访问固定大小数组元素

由于固定大小数组是在编译时已知的数据结构，它的内容表示为程序字节码中的一系列值。访问该数组的一个元素将简单地高效地从程序字节码读取该值。

我们有两种不同的访问固定大小数组元素的方法：

- 将数组解构为多个变量，就像我们对元组所做的那样。

```cairo
#[executable]
fn main() {
    let my_arr = [1, 2, 3, 4, 5];

    // Accessing elements of a fixed-size array by deconstruction
    let [a, b, c, _, _] = my_arr;
    println!("c: {}", c); // c: 3
}
```

- 将数组转换为支持索引的 [Span][span]。这个操作是 _免费_ 的，不会产生任何运行时成本。

```cairo
#[executable]
fn main() {
    let my_arr = [1, 2, 3, 4, 5];

    // Accessing elements of a fixed-size array by index
    let my_span = my_arr.span();
    println!("my_span[2]: {}", my_span[2]); // my_span[2]: 3
}
```

注意，如果我们计划重复访问数组，那么只调用一次 `.span()` 并在整个访问过程中保持其可用是有意义的。

## 类型转换

Cairo 通过使用核心库中的 `TryInto` 和 `Into` trait 提供的 `try_into` 和 `into` 方法来解决类型之间的转换问题。标准库中有许多这些 trait 的实现，用于类型之间的转换，它们也可以为 [自定义类型][custom-type-conversion] 实现。

### Into

`Into` trait 允许一个类型定义如何将自己转换为另一个类型。当成功是保证的时候（例如当源类型小于目标类型时），它可以用于类型转换。

要执行转换，在源值上调用 `var.into()` 以将其转换为另一种类型。新变量的类型必须显式定义，如下例所示。

```cairo
#[executable]
fn main() {
    let my_u8: u8 = 10;
    let my_u16: u16 = my_u8.into();
    let my_u32: u32 = my_u16.into();
    let my_u64: u64 = my_u32.into();
    let my_u128: u128 = my_u64.into();

    let my_felt252 = 10;
    // As a felt252 is smaller than a u256, we can use the into() method
    let my_u256: u256 = my_felt252.into();
    let my_other_felt252: felt252 = my_u8.into();
    let my_third_felt252: felt252 = my_u16.into();
}
```

### TryInto

类似于 `Into`，`TryInto` 是一个用于类型之间转换的泛型 trait。与 `Into` 不同，`TryInto` trait 用于可能失败的转换，因此返回 [Option\<T\>][option]。可能失败的转换的一个例子是目标类型可能无法容纳源值。

同样类似于 `Into` 的是执行转换的过程；只需在源值上调用 `var.try_into()` 将其转换为另一种类型。新变量的类型也必须显式定义，如下例所示。

```cairo
// TAG: does_not_run

#[executable]
fn main() {
    let my_u256: u256 = 10;

    // Since a u256 might not fit in a felt252, we need to unwrap the Option<T> type
    let my_felt252: felt252 = my_u256.try_into().unwrap();
    let my_u128: u128 = my_felt252.try_into().unwrap();
    let my_u64: u64 = my_u128.try_into().unwrap();
    let my_u32: u32 = my_u64.try_into().unwrap();
    let my_u16: u16 = my_u32.try_into().unwrap();
    let my_u8: u8 = my_u16.try_into().unwrap();

    let my_large_u16: u16 = 2048;
    let my_large_u8: u8 = my_large_u16.try_into().unwrap(); // panics with 'Option::unwrap failed.'
}
```

{{#quiz ../quizzes/ch02-02-data-types.toml}}

[arrays]: ./ch03-01-arrays.md
[option]: ./ch06-01-enums.md#the-option-enum-and-its-advantages
[custom-type-conversion]: ./ch05-02-an-example-program-using-structs.md#conversions-of-custom-types
[span]: ./ch03-01-arrays.md#Span
