# 打印 (Printing)

在编写程序时，将以此数据打印到控制台是很常见的，无论是为了程序的正常过程还是出于调试目的。在本章中，我们将描述你打印简单和复杂数据类型的选项。

## 打印标准数据类型

Cairo 提供了两个宏来打印标准数据类型：

- `println!` 在新行上打印
- `print!` 进行内联打印

两者都接受 `ByteArray` 字符串作为第一个参数（参见 [数据类型][byte array]），可以是一个简单的字符串以打印消息，或者是一个带有占位符的字符串以格式化值的打印方式。

有两种方式使用这些占位符，两者可以混合使用：

- 空的大括号 `{}` 按相同顺序被作为参数给出的值替换给 `print!` 宏。
- 带有变量名的大括号被直接替换为变量值。

这是一些例子：

```cairo
#[executable]
fn main() {
    let a = 10;
    let b = 20;
    let c = 30;

    println!("Hello world!");
    println!("{} {} {}", a, b, c); // 10 20 30
    println!("{c} {a} {}", b); // 30 10 20
}
```

> `print!` 和 `println!` 宏在底层使用 `Display` trait，因此用于打印实现了它的类型的值。基本数据类型就是这种情况，但对于更复杂的类型则不是。如果你尝试用这些宏打印复杂数据类型的值，例如出于调试目的，你会得到一个错误。在这种情况下，你可以为你的类型 [手动实现][print with display] `Display` trait，或者使用 `Debug` trait（见 [下文][print with debug]）。

[byte array]: ./ch02-02-data-types.md#byte-array-strings
[print with display]: ./ch12-08-printing.md#printing-custom-data-types
[print with debug]: ./ch12-08-printing.md#print-debug-traces

## 格式化

Cairo 还提供了一个有用的宏来处理字符串格式化：`format!`。这个宏像 `println!` 一样工作，但它不是将输出打印到屏幕，而是返回一个包含内容的 `ByteArray`。在以下示例中，我们使用 `+` 运算符或 `format!` 宏执行字符串连接。使用 `format!` 的代码版本更容易阅读，并且 `format!` 宏生成的代码使用快照，因此此调用不获取其任何参数的所有权。

```cairo
#[executable]
fn main() {
    let s1: ByteArray = "tic";
    let s2: ByteArray = "tac";
    let s3: ByteArray = "toe";
    let s = s1 + "-" + s2 + "-" + s3;
    // using + operator consumes the strings, so they can't be used again!

    let s1: ByteArray = "tic";
    let s2: ByteArray = "tac";
    let s3: ByteArray = "toe";
    let s = format!("{s1}-{s2}-{s3}"); // s1, s2, s3 are not consumed by format!
    // or
    let s = format!("{}-{}-{}", s1, s2, s3);

    println!("{}", s);
}
```

## 打印自定义数据类型

如前所述，如果你尝试使用 `print!` 或 `println!` 宏打印自定义数据类型的值，你会得到一个错误，告诉你 `Display` trait 未为你的自定义类型实现：

```shell
error: Trait has no implementation in context: core::fmt::Display::<package_name::struct_name>
```

`println!` 宏可以做多种格式化，默认情况下，大括号告诉 `println!` 使用称为 `Display` 的格式化 —— 旨在供最终用户直接使用的输出。我们目前看到的原始类型默认实现了 `Display`，因为向用户展示 `1` 或任何其他原始类型只有一种方式。但是对于结构体，`println!` 应该如何格式化输出就不那么清楚了，因为有更多的显示可能性：我们要逗号吗？我们要打印大括号吗？应该显示所有字段吗？由于这种歧义，Cairo 不尝试猜测我们想要什么，并且结构体没有提供用于 `println!` 和 `{}` 占位符的 `Display` 实现。

这是要实现的 `Display` trait：

```cairo,noplayground
trait Display<T> {
    fn fmt(self: @T, ref f: Formatter) -> Result<(), Error>;
}
```

第二个参数 `f` 是 `Formatter` 类型，它只是一个包含 `ByteArray` 的结构体，表示格式化的待定结果：

```cairo,noplayground
#[derive(Default, Drop)]
pub struct Formatter {
    /// The pending result of formatting.
    pub buffer: ByteArray,
}
```

知道了这一点，这是一个如何为自定义 `Point` 结构体实现 `Display` trait 的示例：

```cairo
use core::fmt::{Display, Error, Formatter};

#[derive(Copy, Drop)]
struct Point {
    x: u8,
    y: u8,
}

impl PointDisplay of Display<Point> {
    fn fmt(self: @Point, ref f: Formatter) -> Result<(), Error> {
        let str: ByteArray = format!("Point ({}, {})", *self.x, *self.y);
        f.buffer.append(@str);
        Ok(())
    }
}

#[executable]
fn main() {
    let p = Point { x: 1, y: 3 };
    println!("{}", p); // Point: (1, 3)
}
```

Cairo 还提供了 `write!` 和 `writeln!` 宏以在格式化程序中写入格式化字符串。这是一个使用 `write!` 宏在同一行连接多个字符串然后打印结果的简短示例：

```cairo
use core::fmt::Formatter;

#[executable]
fn main() {
    let mut formatter: Formatter = Default::default();
    let a = 10;
    let b = 20;
    write!(formatter, "hello");
    write!(formatter, "world");
    write!(formatter, " {a} {b}");

    println!("{}", formatter.buffer); // helloworld 10 20
}
```

也可以使用这些宏为 `Point` 结构体实现 `Display` trait，如下所示：

```cairo
use core::fmt::{Display, Error, Formatter};

#[derive(Copy, Drop)]
struct Point {
    x: u8,
    y: u8,
}

impl PointDisplay of Display<Point> {
    fn fmt(self: @Point, ref f: Formatter) -> Result<(), Error> {
        let x = *self.x;
        let y = *self.y;

        writeln!(f, "Point ({x}, {y})")
    }
}

#[executable]
fn main() {
    let p = Point { x: 1, y: 3 };
    println!("{}", p); // Point: (1, 3)
}
```

> 以这种方式打印复杂数据类型可能并不理想，因为它需要额外的步骤来使用 `print!` 和 `println!` 宏。如果你需要打印复杂数据类型，尤其是在调试时，请改用下面描述的 `Debug` trait。

## 以十六进制打印

默认情况下，`Display` trait 以十进制打印整数值。但是，就像在 Rust 中一样，你可以使用 `{:x}` 符号以十六进制打印它们。

在底层，Cairo 为常见类型如无符号整数、`felt252` 和 `NonZero` 实现了 `LowerHex` trait，也为常见的 Starknet 类型如 `ContractAddress` 和 `ClassHash` 实现了该 trait。

如果这对你有意义，你也可以使用与 `Display` trait 相同的方法为你的自定义类型实现 `LowerHex` trait（参见 [打印自定义数据类型][print with display]）。

## 打印调试痕迹

Cairo 提供了 `Debug` trait，可以派生它以在调试时打印变量的值。只需在 `print!` 或 `println!` 宏字符串的大括号 `{}` 占位符内添加 `:?`。

这个 trait 非常有用，并且默认情况下为基本数据类型实现。对于复杂数据类型，只要它们包含的所有类型都实现了它，就可以使用 `#[derive(Debug)]` 属性简单地派生它。这消除了手动实现额外代码以打印复杂数据类型的需要。

注意，测试中使用的 `assert_xx!` 宏要求提供的值实现 `Debug` trait，因为它们在断言失败的情况下也会打印结果。

有关 `Debug` trait 及其在调试时打印值的用法的更多详细信息，请参阅 [可派生 Traits][debug trait] 附录。

[debug trait]:
  ./appendix-03-derivable-traits.md#debug-trait-for-printing-and-debugging
