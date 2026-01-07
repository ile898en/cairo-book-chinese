# 一个使用结构体的示例程序

为了理解我们何时可能想使用结构体，让我们编写一个计算矩形面积的程序。我们将从使用单个变量开始，然后重构程序直到我们使用结构体代替。

让我们用 Scarb 创建一个名为 _rectangles_ 的新项目，该项目将获取以像素为单位指定的矩形的宽度和高度，并计算矩形的面积。清单 {{#ref area-fn}} 展示了一个在我们的项目 _src/lib.cairo_ 中实现此目的的一种方法的简短程序。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[executable]
fn main() {
    let width = 30;
    let height = 10;
    let area = area(width, height);
    println!("Area is {}", area);
}

//ANCHOR: here
fn area(width: u64, height: u64) -> u64 {
    //ANCHOR_END: here
    width * height
}
```

{{#label area-fn}} <span class="caption">清单 {{#ref area-fn}}: 计算由单独的宽度和高度变量指定的矩形的面积。</span>

现在用 `scarb execute` 运行程序：

```shell
$ scarb execute 
   Compiling listing_04_06_no_struct v0.1.0 (listings/ch05-using-structs-to-structure-related-data/listing_03_no_struct/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing listing_04_06_no_struct
Area is 300


```

这段代码通过用每个维度调用 `area` 函数成功地计算出了矩形的面积，但我们可以做得更多，使这段代码更清晰、更易读。

这段代码的问题在 `area` 的签名中显而易见：

```cairo,noplayground
fn area(width: u64, height: u64) -> u64 {
```

`area` 函数应该计算一个矩形的面积，但我们要编写的函数有两个参数，而且在我们的程序中没有任何地方明确表明这些参数是相关的。将宽度和高度组合在一起会更易读、更易于管理。我们在 [第 {{#chap common-programming-concepts}} 章的元组部分](./ch02-02-data-types.md#the-tuple-type) 中已经讨论过一种可能的方法。

## 使用元组重构

清单 {{#ref rectangle-tuple}} 展示了我们程序的另一个使用元组的版本。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[executable]
fn main() {
    let rectangle = (30, 10);
    let area = area(rectangle);
    println!("Area is {}", area);
}

fn area(dimension: (u64, u64)) -> u64 {
    let (x, y) = dimension;
    x * y
}
```

{{#label rectangle-tuple}} <span class="caption">清单
{{#ref rectangle-tuple}}: 使用元组指定矩形的宽度和高度。</span>

在某种程度上，这个程序更好了。元组让我们增加了一点结构，而且我们现在只传递一个参数。但在另一种方式上，这个版本不太清晰：元组没有给它们的元素命名，所以我们必须索引到元组的部分，这使得我们的计算不那么明显。

混淆宽度和高度对于面积计算没有影响，但如果我们想计算差值，那就有影响了！我们必须记住 `width` 是元组索引 `0`，`height` 是元组索引 `1`。如果别人要使用我们的代码，这也将更难弄清楚并记住。因为我们没有在代码中传达数据的含义，现在更容易引入错误。

## 使用结构体重构：增加更多意义

我们使用结构体通过标记数据来增加意义。我们可以将我们正在使用的元组转换为一个结构体，并为整体命名，也为部分命名。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
struct Rectangle {
    width: u64,
    height: u64,
}

#[executable]
fn main() {
    let rectangle = Rectangle { width: 30, height: 10 };
    let area = area(rectangle);
    println!("Area is {}", area);
}

fn area(rectangle: Rectangle) -> u64 {
    rectangle.width * rectangle.height
}
```

{{#label rectangle-struct}} <span class="caption">清单
{{#ref rectangle-struct}}: 定义一个 `Rectangle` 结构体。</span>

在这里我们定义了一个结构体并将其命名为 `Rectangle`。在花括号内，我们将字段定义为 `width` 和 `height`，它们的类型都是 `u64`。然后，在 `main` 中，我们创建了一个特定的 `Rectangle` 实例，它的宽度为 `30`，高度为 `10`。我们的 `area` 函数现在被定义为有一个参数，我们将其命名为 `rectangle`，其类型为 `Rectangle` 结构体。然后我们可以用点号访问实例的字段，这给值提供了描述性的名称，而不是使用元组索引值 `0` 和 `1`。

## 自定义类型的转换

我们已经描述了如何对内置类型执行类型转换，请参见 [数据类型 > 类型转换][type-conversion]。在本节中，我们将看到如何为自定义类型定义转换。

> 注意：也可以为复合类型（例如元组）定义转换。

[type-conversion]: ./ch02-02-data-types.md#type-conversion

### Into

使用 `Into` trait 为自定义类型定义转换通常需要指定要转换为的类型，因为编译器大多数时候无法确定这一点。然而，考虑到我们要免费获得的功能，这是一个很小的权衡。

```cairo
#[derive(Drop, PartialEq)]
struct Rectangle {
    width: u64,
    height: u64,
}

#[derive(Drop)]
struct Square {
    side_length: u64,
}

impl SquareIntoRectangle of Into<Square, Rectangle> {
    fn into(self: Square) -> Rectangle {
        Rectangle { width: self.side_length, height: self.side_length }
    }
}

#[executable]
fn main() {
    let square = Square { side_length: 5 };
    // Compiler will complain if you remove the type annotation
    let result: Rectangle = square.into();
    let expected = Rectangle { width: 5, height: 5 };
    assert!(
        result == expected,
        "A square is always convertible to a rectangle with the same width and height!",
    );
}
```

### TryInto

为 `TryInto` 定义转换类似于为 `Into` 定义。

```cairo
#[derive(Drop)]
struct Rectangle {
    width: u64,
    height: u64,
}

#[derive(Drop, PartialEq)]
struct Square {
    side_length: u64,
}

impl RectangleIntoSquare of TryInto<Rectangle, Square> {
    fn try_into(self: Rectangle) -> Option<Square> {
        if self.height == self.width {
            Some(Square { side_length: self.height })
        } else {
            None
        }
    }
}

#[executable]
fn main() {
    let rectangle = Rectangle { width: 8, height: 8 };
    let result: Square = rectangle.try_into().unwrap();
    let expected = Square { side_length: 8 };
    assert!(
        result == expected,
        "Rectangle with equal width and height should be convertible to a square.",
    );

    let rectangle = Rectangle { width: 5, height: 8 };
    let result: Option<Square> = rectangle.try_into();
    assert!(
        result.is_none(),
        "Rectangle with different width and height should not be convertible to a square.",
    );
}
```

{{#quiz ../quizzes/ch05-02-an-example-program-using-structs.toml}}
