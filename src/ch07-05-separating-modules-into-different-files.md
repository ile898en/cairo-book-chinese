# 将模块分离到不同文件

到目前为止，本章中的所有示例都在一个文件中定义了多个模块。当模块变大时，你可能希望将它们的定义移动到单独的文件中，以使代码更易于浏览。

例如，让我们从清单 {{#ref use-keyword}} 中的代码开始，它有多个餐厅模块。我们将把模块提取到文件中，而不是将所有模块定义在 crate 根文件中。在这种情况下，crate 根文件是 _src/lib.cairo_。

首先，我们将把 `front_of_house` 模块提取到它自己的文件中。删除 `front_of_house` 模块花括号内的代码，只保留 `mod front_of_house;` 声明，以便 _src/lib.cairo_ 包含清单 {{#ref front-extraction}} 中显示的代码。注意，在我们创建 _src/front_of_house.cairo_ 文件之前，这不会编译。

<span class="filename">文件名: src/lib.cairo</span>

```cairo,noplayground
mod front_of_house;
use crate::front_of_house::hosting;

fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
```

{{#label front-extraction}} <span class="caption">清单
{{#ref front-extraction}}: 声明 `front_of_house` 模块，其主体将位于 _src/front_of_house.cairo_ 中</span>

接下来，将花括号中的代码放入名为 _src/front_of_house.cairo_ 的新文件中，如清单 {{#ref module-foh}} 所示。编译器知道在这个文件中查找，因为它在 crate 根目录中遇到了名为 `front_of_house` 的模块声明。

<span class="filename">文件名: src/front_of_house.cairo</span>

```cairo,noplayground
pub mod hosting {
    pub fn add_to_waitlist() {}
}
```

{{#label module-foh}} <span class="caption">清单 {{#ref module-foh}}:
_src/front_of_house.cairo_ 中 `front_of_house` 模块内的定义</span>

注意，在你的模块树中，你也只需要使用 `mod` 声明加载一次文件。一旦编译器知道该文件是项目的一部分（并且知道代码位于模块树中的何处，因为你放置了 `mod` 语句），项目中的其他文件应该使用指向其声明位置的路径来引用加载文件的代码，这在 [“引用模块树中项的路径”][path] 一章中已涵盖。换句话说，`mod` _不是_ 你可能在其他编程语言中看到的“包含”操作。

接下来，我们将把 `hosting` 模块提取到它自己的文件中。过程有点不同，因为 `hosting` 是 `front_of_house` 的子模块，而不是根模块的子模块。我们将把 `hosting` 的文件放在一个以此模块树中的祖先命名的新目录中，在这种情况下是 _src/front_of_house/_。

开始移动 `hosting`，我们更改 _src/front_of_house.cairo_ 以仅包含 `hosting` 模块的声明：

<span class="filename">文件名: src/front_of_house.cairo</span>

```cairo,noplayground
pub mod hosting;
```

然后我们创建一个 _src/front_of_house_ 目录和一个文件 _hosting.cairo_ 来包含 `hosting` 模块中的定义：

<span class="filename">文件名: src/front_of_house/hosting.cairo</span>

```cairo,noplayground
pub fn add_to_waitlist() {}
```

如果我们把 _hosting.cairo_ 放在 _src_ 目录中，编译器会期望 _hosting.cairo_ 代码位于 crate 根目录声明的 `hosting` 模块中，而不是声明为 `front_of_house` 模块的子模块。编译器关于检查哪些文件以获取哪些模块代码的规则意味着目录和文件更紧密地匹配模块树。

我们将每个模块的代码移动到了单独的文件中，模块树保持不变。`eat_at_restaurant` 中的函数调用无需任何修改即可工作，即使定义位于不同的文件中。这种技术允许你在模块大小增长时将模块移动到新文件中。

注意 _src/lib.cairo_ 中的 `use crate::front_of_house::hosting;` 语句也没有改变，`use` 对作为 crate 一部分编译的文件也没有任何影响。`mod` 关键字声明模块，Cairo 会在与模块同名的文件中查找进入该模块的代码。

[path]: ./ch07-03-paths-for-referring-to-an-item-in-the-module-tree.md

## 总结

Cairo 允许你将一个包拆分为多个 crate，将一个 crate 拆分为多个模块，以便你可以从另一个模块引用一个模块中定义的项。你可以通过指定绝对路径或相对路径来做到这一点。这些路径可以使用 `use` 语句引入作用域，以便你可以在该作用域中多次使用该项时使用更短的路径。模块代码默认是 **私有** 的。

{{#quiz ../quizzes/ch07-05-separate-modules.toml}}
