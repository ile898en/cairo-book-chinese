# 使用 `use` 关键字将路径引入作用域

必须写出路径来调用函数可能会让人觉得不方便和重复。幸运的是，有一种方法可以简化这个过程：我们可以使用 `use` 关键字一次性创建一个路径的快捷方式，然后在作用域的其他地方使用更短的名称。

在清单 {{#ref use-keyword}} 中，我们将 `crate::front_of_house::hosting` 模块引入 `eat_at_restaurant` 函数的作用域，这样我们只需要指定 `hosting::add_to_waitlist` 就可以在 `eat_at_restaurant` 中调用 `add_to_waitlist` 函数。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
// section "Defining Modules to Control Scope"

mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist(); // ✅ Shorter path
}
```

{{#label use-keyword}} <span class="caption">清单 {{#ref use-keyword}}:
使用 `use` 将模块引入作用域</span>

在作用域中添加 `use` 和路径类似于在文件系统中创建符号链接。通过在 crate 根中添加 `use crate::front_of_house::hosting;`，`hosting` 现在在该作用域中是一个有效的名称，就像 `hosting` 模块被定义在 crate 根中一样。

注意 `use` 只能在 `use` 发生的特定作用域内创建快捷方式。清单 {{#ref use-scope}} 将 `eat_at_restaurant` 函数移动到一个名为 `customer` 的新子模块中，这与 `use` 语句是不同的作用域，所以函数体将无法编译：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
use crate::front_of_house::hosting;

mod customer {
    pub fn eat_at_restaurant() {
        hosting::add_to_waitlist();
    }
}
```

{{#label use-scope}} <span class="caption">清单 {{#ref use-scope}}: `use` 语句只适用于它所在的作用域。</span>

编译器错误显示快捷方式不再在 `customer` 模块内适用：

```shell
$ scarb build 
   Compiling listing_07_05 v0.1.0 (listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_07_use_and_scope/Scarb.toml)
warn: Unused import: `listing_07_05::hosting`
 --> listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_07_use_and_scope/src/lib.cairo:9:28
use crate::front_of_house::hosting;
                           ^^^^^^^

error[E0006]: Identifier not found.
 --> listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_07_use_and_scope/src/lib.cairo:13:9
        hosting::add_to_waitlist();
        ^^^^^^^

error: could not compile `listing_07_05` due to previous error

```

## 创建惯用的 `use` 路径

在清单 {{#ref use-keyword}} 中，你可能想知道为什么我们指定 `use crate::front_of_house::hosting` 然后在 `eat_at_restaurant` 中调用 `hosting::add_to_waitlist`，而不是指定 `use` 路径一直到 `add_to_waitlist` 函数以实现相同的结果，如清单 {{#ref unidiomatic-use}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}
use crate::front_of_house::hosting::add_to_waitlist;

pub fn eat_at_restaurant() {
    add_to_waitlist();
}
```

{{#label unidiomatic-use}} <span class="caption">清单
{{#ref unidiomatic-use}}: 使用 `use` 将 `add_to_waitlist` 函数引入作用域，这是不惯用的</span>

虽然清单 {{#ref use-keyword}} 和 {{#ref unidiomatic-use}} 完成了相同的任务，但清单 {{#ref use-keyword}} 是使用 `use` 将函数引入作用域的惯用方式。使用 `use` 将函数的父模块引入作用域意味着我们在调用函数时必须指定父模块。在调用函数时指定父模块可以清楚地表明该函数不是在本地定义的，同时仍最大程度地减少完整路径的重复。清单 {{#ref unidiomatic-use}} 中的代码不清楚 `add_to_waitlist` 是在哪里定义的。

另一方面，当使用 `use` 引入结构体、枚举、trait 和其他项时，指定完整路径是惯用的。清单 {{#ref idiomatic-use}} 展示了将核心库的 `BitSize` trait 引入作用域的惯用方式，允许调用 `bits` 方法来检索类型的位数大小。

```cairo
use core::num::traits::BitSize;

#[executable]
fn main() {
    let u8_size: usize = BitSize::<u8>::bits();
    println!("A u8 variable has {} bits", u8_size)
}
```

{{#label idiomatic-use}} <span class="caption">清单 {{#ref idiomatic-use}}: 以惯用方式将 `BitSize` trait 引入作用域</span>

这种习惯没有什么强烈的理由：这只是 Rust 社区中出现的惯例，人们已经习惯了以这种方式阅读和编写 Rust 代码。由于 Cairo 共享许多 Rust 的惯名词，我们也遵循这个惯例。

这个惯用语法的例外是如果我们使用 `use` 语句将两个同名的项引入作用域，因为 Cairo 不允许这样做。

### 使用 `as` 关键字提供新名称

对于使用 `use` 将两个同名类型引入同一作用域的问题，还有另一种解决方案：在路径之后，我们可以指定 `as` 和一个新的本地名称，或类型的 _别名 (alias)_。清单 {{#ref as-keyword}} 展示了如何使用 `as` 重命名导入：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
use core::array::ArrayTrait as Arr;

#[executable]
fn main() {
    let mut arr = Arr::new(); // ArrayTrait was renamed to Arr
    arr.append(1);
}
```

{{#label as-keyword}} <span class="caption">清单 {{#ref as-keyword}}:
使用 `as` 关键字将 trait 引入作用域时重命名它</span>

在这里，我们将 `ArrayTrait` 引入作用域，别名为 `Arr`。我们现在可以使用 `Arr` 标识符访问该 trait 的方法。

### 从同一模块导入多个项

当你想从同一个模块导入多个项（如函数、结构体或枚举）时，你可以使用花括号 `{}` 列出你想导入的所有项。这有助于保持你的代码整洁易读，避免一长串单独的 `use` 语句。

从同一模块导入多个项的一般语法是：

```cairo
use module::{item1, item2, item3};
```

这是一个我们从同一个模块导入三个结构的例子：

```cairo
// Assuming we have a module called `shapes` with the structures `Square`, `Circle`, and `Triangle`.
mod shapes {
    #[derive(Drop)]
    pub struct Square {
        pub side: u32,
    }

    #[derive(Drop)]
    pub struct Circle {
        pub radius: u32,
    }

    #[derive(Drop)]
    pub struct Triangle {
        pub base: u32,
        pub height: u32,
    }
}

// We can import the structures `Square`, `Circle`, and `Triangle` from the `shapes` module like
// this:
use shapes::{Circle, Square, Triangle};

// Now we can directly use `Square`, `Circle`, and `Triangle` in our code.
#[executable]
fn main() {
    let sq = Square { side: 5 };
    let cr = Circle { radius: 3 };
    let tr = Triangle { base: 5, height: 2 };
    // ...
}
```

{{#label multiple-imports}} <span class="caption">清单
{{#ref multiple-imports}}: 从同一模块导入多个项</span>

## 在模块文件中重导出名称

当我们使用 `use` 关键字将名称引入作用域时，新作用域中可用的名称可以像在该代码作用域中定义的一样被导入。这种技术称为 _重导出 (re-exporting)_，因为我们将一个项引入作用域，但也通过 `pub` 关键字使该项可供其他人引入他们的作用域。

例如，让我们在餐厅示例中重导出 `add_to_waitlist` 函数：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

{{#label reexporting}} <span class="caption">清单 {{#ref reexporting}}:
使用 `pub use` 使名称可供任何代码从新作用域使用</span>

在此更改之前，外部代码必须使用路径 `restaurant::front_of_house::hosting::add_to_waitlist()` 调用 `add_to_waitlist` 函数。现在 `pub use` 已经从根模块重导出了 `hosting` 模块，外部代码现在可以使用路径 `restaurant::hosting::add_to_waitlist()`。

当你的代码的内部结构与调用你的代码的程序员对领域的思考方式不同时，重导出很有用。例如，在这个餐厅隐喻中，经营餐厅的人会考虑“前厅”和“后厨”。但是访问餐厅的顾客可能不会用这些术语来思考餐厅的部分。使用 `pub use`，我们可以用一种结构编写代码，但暴露另一种结构。这样做使我们的库对于处理库的程序员和调用库的程序员来说都组织良好。

## 使用 Scarb 在 Cairo 中使用外部包

你可能需要使用外部包来利用社区提供的功能。Scarb 允许你通过从 Git 仓库克隆包来使用依赖项。要在 Scarb 项目中使用外部包，只需在 _Scarb.toml_ 配置文件中的专用 `[dependencies]` 部分中声明你要添加的依赖项的 Git 仓库 URL。请注意，URL 可能对应于 main 分支，或任何特定的提交、分支或标签。为此，你需要分别传递额外的 `rev`、`branch` 或 `tag` 字段。例如，以下代码从 _alexandria_ 包导入 _alexandria_math_ crate 的 main 分支：

```cairo
[dependencies]
alexandria_math = { git = "https://github.com/keep-starknet-strange/alexandria.git" }
```

而由于以下代码导入特定分支（已弃用，不应使用）：

```cairo
[dependencies]
alexandria_math = { git = "https://github.com/keep-starknet-strange/alexandria.git", branch = "cairo-v2.3.0-rc0" }
```

如果你想在项目中导入多个包，你只需要创建一个 `[dependencies]` 部分并在其下列出所有所需的包。你也可以通过声明 `[dev-dependencies]` 部分来指定开发依赖项。

之后，只需运行 `scarb build` 即可获取所有外部依赖项并编译包含所有依赖项的包。

注意，也可以使用 `scarb add` 命令添加依赖项，该命令会自动为你编辑 _Scarb.toml_ 文件。对于开发依赖项，只需使用 `scarb add --dev` 命令。

要删除依赖项，只需从 _Scarb.toml_ 文件中删除相应的行，或使用 `scarb rm` 命令。

## Glob 运算符

如果我们想将通过路径定义的所有公共项引入作用域，我们可以指定该路径后跟 `*` glob 运算符：

```rust
use core::num::traits::*;
```

这个 `use` 语句将 `core::num::traits` 中定义的所有公共项引入当前作用域。使用 glob 运算符时要小心！Glob 会使你更难分辨哪些名称在作用域内，以及程序中使用的名称是在哪里定义的。

Glob 运算符通常在测试时用于将所有测试内容引入 `tests` 模块；我们将在第 {{#chap how-to-write-tests}} 章的 [“如何编写测试”][writing-tests] 一节中讨论这一点。

[writing-tests]: ./ch10-01-how-to-write-tests.md

{{#quiz ../quizzes/ch07-04-bringing-paths-into-scope.toml}}
