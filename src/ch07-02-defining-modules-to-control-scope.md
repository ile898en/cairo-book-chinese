# 定义模块以控制作用域

在本节中，我们将讨论模块和模块系统的其他部分，即允许你命名项的 _路径 (paths)_ 和将路径引入作用域的 `use` 关键字。

首先，我们将从一系列规则开始，以便你在将来组织代码时可以轻松参考。然后我们将详细解释每条规则。

## 模块备忘单

这里我们提供关于模块、路径和 `use` 关键字如何在编译器中工作，以及大多数开发人员如何组织他们的代码的快速参考。我们将在本章中通过示例来讲解这些规则中的每一条，但这是一个很好的参考位置，可以作为模块工作原理的提醒。你可以使用 `scarb new backyard` 创建一个新的 Scarb 项目来跟随操作。

- **从 crate 根开始**: 编译 crate 时，编译器首先在 crate 根文件 (_src/lib.cairo_) 中查找要编译的代码。
- **声明模块**: 在 crate 根文件中，你可以声明新模块；比如，你声明一个 `garden` 模块 `mod garden;`。编译器会在这些地方查找模块的代码：
  - 内联，在替换 `mod garden` 后面的分号的花括号内。

    ```cairo,noplayground
      // crate 根文件 (src/lib.cairo)
      mod garden {
          // 定义 garden 模块的代码放在这里
      }
    ```

  - 在文件 _src/garden.cairo_ 中。

- **声明子模块**: 在 crate 根以外的任何文件中，你可以声明子模块。例如，你可能在 _src/garden.cairo_ 中声明 `mod vegetables;`。编译器会在以父模块命名的目录中的这些地方查找子模块的代码：
  - 内联，直接跟在 `mod vegetables` 后面，在花括号内而不是分号。

    ```cairo,noplayground
    // src/garden.cairo 文件
    mod vegetables {
        // 定义 vegetables 子模块的代码放在这里
    }
    ```

  - 在文件 _src/garden/vegetables.cairo_ 中。

- **模块中代码的路径**: 一旦模块成为你的 crate 的一部分，你就可以从同一个 crate 中的任何其他地方引用该模块中的代码，使用代码的路径。例如，`vegetables` 子模块中的 `Asparagus` 类型可以在 `crate::garden::vegetables::Asparagus` 找到。
- **私有与公有**: 默认情况下，模块内的代码对其父模块是私有的。这意味着它只能被当前模块及其后代访问。要使模块公有，请使用 `pub mod` 而不是 `mod` 声明它。要使公有模块内的项也公有，请在它们的声明前使用 `pub`。Cairo 还提供了 `pub(crate)` 关键字，允许项或模块仅在包含定义的 crate 内可见。
- **`use` 关键字**: 在一个作用域内，`use` 关键字创建项的快捷方式，以减少长路径的重复。在任何可以引用 `crate::garden::vegetables::Asparagus` 的作用域中，你可以用 `use crate::garden::vegetables::Asparagus;` 创建一个快捷方式，从那时起你只需要写 `Asparagus` 就可以在作用域中使用该类型。

这里我们创建一个名为 `backyard` 的 crate 来演示这些规则。crate 的目录，也名为 `backyard`，包含这些文件和目录：

```text
backyard/
├── Scarb.toml
└── src
    ├── garden
    │   └── vegetables.cairo
    ├── garden.cairo
    └── lib.cairo
```

在这种情况下，crate 根文件是 _src/lib.cairo_，它包含：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
pub mod garden;
use crate::garden::vegetables::Asparagus;

#[executable]
fn main() {
    let plant = Asparagus {};
    println!("I'm growing {:?}!", plant);
}
```

`pub mod garden;` 行导入 `garden` 模块。使用 `pub` 使 `garden` 公开访问，或者如果你真的想让 `garden` 仅对你的 crate 可用则使用 `pub(crate)`，在这里运行我们的程序是可选的，因为 `main` 函数驻留在与 `pub mod garden;` 声明相同的模块中。尽管如此，不声明 `garden` 为 `pub` 将使其无法从任何其他包访问。这行告诉编译器包含它在 _src/garden.cairo_ 中找到的代码，即：

<span class="filename">文件名: src/garden.cairo</span>

```cairo,noplayground
pub mod vegetables;
```

在这里，`pub mod vegetables;` 意味着 _src/garden/vegetables.cairo_ 中的代码也被包含在内。该代码是：

```cairo,noplayground
#[derive(Drop, Debug)]
pub struct Asparagus {}
```

行 `use crate::garden::vegetables::Asparagus;` 让我们将 `Asparagus` 类型引入作用域，以便我们可以在 `main` 函数中使用它。

现在让我们深入了解这些规则的细节并在实际操作中演示它们！

## 在模块中分组相关代码

_模块 (Modules)_ 让我们在一个 crate 内组织代码，以提高可读性和易于重用。模块还允许我们控制项的隐私，因为默认情况下模块内的代码是私有的。私有项是不可供外部使用的内部实现细节。我们可以选择使模块及其内部的项公有，这将公开它们以允许外部代码使用和依赖它们。

作为一个例子，让我们编写一个提供餐厅功能的库 Crate。我们将定义函数的签名但将它们的主体留空，以专注于代码的组织，而不是餐厅的实现。

在餐饮业中，餐厅的某些部分被称为 _前厅 (front of house)_，其他部分被称为 _后厨 (back of house)_。前厅是顾客所在的地方；这包括迎宾员安排顾客入座、服务员点菜和付款、以及调酒师调制饮料的地方。后厨是厨师和帮厨在厨房工作、洗碗工清理、以及经理做行政工作的地方。

为了以这种方式构建我们的 crate，我们可以将其函数组织成嵌套模块。通过运行 `scarb new restaurant` 创建一个名为 _restaurant_ 的新包；然后将清单 {{#ref front_of_house}} 中的代码输入 _src/lib.cairo_ 以定义一些模块和函数签名。这是前厅部分：

<span class="filename">文件名: src/lib.cairo</span>

```cairo,noplayground
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```

{{#label front_of_house}} <span class="caption">清单 {{#ref front_of_house}}:
一个包含其他模块的 `front_of_house` 模块，这些模块又包含函数</span>

我们使用 `mod` 关键字后跟模块名称（在本例中为 `front_of_house`）来定义一个模块。模块的主体然后放在花括号内。在模块内，我们可以放置其他模块，就像本例中的 `hosting` 和 `serving` 模块一样。模块还可以保存其他项的定义，例如结构体、枚举、常量、trait 和函数。

通过使用模块，我们可以将相关定义分组在一起，并命名它们为什么相关。使用此代码的程序员可以根据组来浏览代码，而不必通读所有定义，从而更容易找到与他们相关的定义。向此代码添加新功能的程序员将知道在哪里放置代码以保持程序井井有条。

之前，我们提到 _src/lib.cairo_ 被称为 crate 根。这个名字的原因是这个文件的内容形成了一个以 crate 名称命名的模块，位于 crate 模块结构的根部，称为 _模块树 (module tree)_。

清单 {{#ref module-tree}} 展示了清单 {{#ref front_of_house}} 中结构的模块树。

```text
restaurant
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

{{#label module-tree}} <span class="caption">清单 {{#ref module-tree}}: 清单 {{#ref front_of_house}} 中代码的模块树</span>

这棵树展示了一些模块如何嵌套在另一个模块中；例如，`hosting` 嵌套在 `front_of_house` 中。该树还显示一些模块彼此是 _兄弟 (siblings)_，意味着它们定义在同一个模块中；`hosting` 和 `serving` 是定义在 `front_of_house` 内的兄弟。如果模块 A 包含在模块 B 内，我们说模块 A 是模块 B 的 _子 (child)_，模块 B 是模块 A 的 _父 (parent)_。注意整个模块树都位于名为 _restaurant_ 的显式名称之下。

模块树可能会让你想起计算机上的文件系统目录树；这是一个非常恰当的类比！就像文件系统中的目录一样，你使用模块来组织你的代码。就像目录中的文件一样，我们需要一种方法来找到我们的模块。

{{#quiz ../quizzes/ch07-02-defining-modules-to-control-scope.toml}}
