# 引用模块树中项的路径

为了向 Cairo 展示在模块树中哪里可以找到一个项，我们使用路径，就像我们在浏览文件系统时使用路径一样。要调用一个函数，我们需要知道它的路径。

路径可以采取两种形式：

- _绝对路径 (absolute path)_ 是从 crate 根开始的完整路径。绝对路径以 crate 名称开头。
- _相对路径 (relative path)_ 从当前模块开始。

绝对路径和相对路径后都跟一个或多个由双冒号 (`::`) 分隔的标识符。

为了说明这个概念，让我们拿回我们在上一章使用的餐厅示例清单 {{#ref front_of_house}}。我们有一个名为 _restaurant_ 的 crate，其中有一个名为 `front_of_house` 的模块，它包含一个名为 `hosting` 的模块。`hosting` 模块包含一个名为 `add_to_waitlist` 的函数。我们想从 `eat_at_restaurant` 函数调用 `add_to_waitlist` 函数。我们需要告诉 Cairo `add_to_waitlist` 函数的路径，以便它可以找到它。

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


pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

{{#label path-types}} <span class="caption">清单 {{#ref path-types}}: 使用绝对路径和相对路径调用 `add_to_waitlist` 函数</span>

`eat_at_restaurant` 函数是我们库的公共 API 的一部分，所以我们用 `pub` 关键字标记它。我们将在 [“使用 `pub` 关键字暴露路径”][pub] 一节中详细介绍 `pub`。

我们第一次在 `eat_at_restaurant` 中调用 `add_to_waitlist` 函数时，我们使用了绝对路径。`add_to_waitlist` 函数定义在与 `eat_at_restaurant` 相同的 crate 中。在 Cairo 中，绝对路径从 crate 根开始，你需要使用 crate 名称来引用它。你可以想象一个具有相同结构的文件系统：我们会指定路径 _/front_of_house/hosting/add_to_waitlist_ 来运行 _add_to_waitlist_ 程序；使用 crate 名称从 crate 根开始就像在 shell 中使用斜杠 (`/`) 从文件系统根目录开始一样。

我们要第二次调用 `add_to_waitlist` 时，我们使用了相对路径。路径以 `front_of_house` 开头，这是在与 `eat_at_restaurant` 相同的模块树级别定义的模块名称。在这里，等效的文件系统将使用路径 _./front_of_house/hosting/add_to_waitlist_。以模块名称开头意味着路径是相对于当前模块的。

让我们尝试编译清单 {{#ref path-types}} 并找出为什么它还不能编译！我们得到以下错误：

```shell
$ scarb execute 
   Compiling listing_07_02 v0.1.0 (listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_02_paths/Scarb.toml)
error: Item `listing_07_02::front_of_house::hosting` is not visible in this context.
 --> listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_02_paths/src/lib.cairo:22:28
    crate::front_of_house::hosting::add_to_waitlist();
                           ^^^^^^^

error: Item `listing_07_02::front_of_house::hosting::add_to_waitlist` is not visible in this context.
 --> listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_02_paths/src/lib.cairo:22:37
    crate::front_of_house::hosting::add_to_waitlist();
                                    ^^^^^^^^^^^^^^^

error: Item `listing_07_02::front_of_house::hosting` is not visible in this context.
 --> listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_02_paths/src/lib.cairo:25:21
    front_of_house::hosting::add_to_waitlist();
                    ^^^^^^^

error: Item `listing_07_02::front_of_house::hosting::add_to_waitlist` is not visible in this context.
 --> listings/ch07-managing-cairo-projects-with-packages-crates-and-modules/listing_02_paths/src/lib.cairo:25:30
    front_of_house::hosting::add_to_waitlist();
                             ^^^^^^^^^^^^^^^

error: could not compile `listing_07_02` due to previous error
error: `scarb` command exited with error

```

错误消息说模块 `hosting` 和 `add_to_waitlist` 函数不可见。换句话说，我们有 `hosting` 模块和 `add_to_waitlist` 函数的正确路径，但 Cairo 不让我们使用它们，因为它没有访问权限。在 Cairo 中，所有项（函数、方法、结构体、枚举、模块和常量）默认对其父模块是私有的。如果你想让一个像函数或结构体这样的项私有，你就把它放在一个模块里。

父模块中的项不能使用子模块内的私有项，但子模块中的项可以使用其祖先模块中的项。这是因为子模块包装并隐藏了它们的实现细节，但子模块可以看到它们被定义的上下文。继续我们的比喻，把隐私规则想象成餐厅的后勤办公室：那里发生的事情对餐厅顾客是私密的，但办公室经理可以看到并做他们经营的餐厅里的一切。

Cairo 选择让模块系统以这种方式运作，以便默认隐藏内部实现细节。这样，你就知道你可以更改内部代码的哪些部分而不会破坏外部代码。然而，Cairo 确实给了你选择，通过使用 `pub` 关键字将项公开，将子模块代码的内部部分暴露给外部祖先模块。

[pub]: ./ch07-03-paths-for-referring-to-an-item-in-the-module-tree.md#exposing-paths-with-the-pub-keyword

## 使用 `pub` 关键字暴露路径

让我们回到之前的错误，它告诉我们 `hosting` 模块和 `add_to_waitlist` 函数不可见。我们希望父模块中的 `eat_at_restaurant` 函数能够访问子模块中的 `add_to_waitlist` 函数，所以我们用 `pub` 关键字标记 `hosting` 模块，如清单 {{#ref pub-keyword-not-compiling}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo,noplayground
//TAG: does_not_compile
mod front_of_house {
    pub mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```

{{#label pub-keyword-not-compiling}} <span class="caption">清单
{{#ref pub-keyword-not-compiling}}: 将 `hosting` 模块声明为 `pub` 以便从 `eat_at_restaurant` 使用它</span>

不幸的是，清单 {{#ref pub-keyword-not-compiling}} 中的代码仍然导致错误。

发生了什么？在 `mod hosting;` 前添加 `pub` 关键字使模块公有。有了这个变化，如果我们能访问 `front_of_house`，我们就能访问 `hosting`。但是 `hosting` 的内容仍然是私有的；使模块公有并不会使其内容公有。模块上的 `pub` 关键字只允许其祖先模块中的代码引用它，而不能访问其内部代码。因为模块是容器，仅仅使模块公有我们做不了太多事情；我们需要更进一步，选择将模块内的一个或多个项也公开。

让我们也通过在定义前添加 `pub` 关键字来使 `add_to_waitlist` 函数公有，如清单 {{#ref pub-keyword}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo,noplayground
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist(); // ✅ Compiles

    // Relative path
    front_of_house::hosting::add_to_waitlist(); // ✅ Compiles
}
```

{{#label pub-keyword}} <span class="caption">清单 {{#ref pub-keyword}}:
将 `hosting` 模块声明为 `pub` 以便从 `eat_at_restaurant` 使用它</span>

现在代码可以编译了！要了解为什么添加 `pub` 关键字让我们可以在遵守隐私规则的情况下在 `add_to_waitlist` 中使用这些路径，让我们看看绝对路径和相对路径。

在绝对路径中，我们从 crate 根开始，即我们 crate 模块树的根。`front_of_house` 模块定义在 crate 根中。虽然 `front_of_house` 不是公有的，因为 `eat_at_restaurant` 函数定义在与 `front_of_house` 相同的模块中（也就是说，`front_of_house` 和 `eat_at_restaurant` 是兄弟），我们可以从 `eat_at_restaurant` 引用 `front_of_house`。接下来是用 `pub` 标记的 `hosting` 模块。我们可以访问 `hosting` 的父模块，所以我们可以访问 `hosting` 本身。最后，`add_to_waitlist` 函数用 `pub` 标记，我们可以访问其父模块，所以这个函数调用有效！

在相对路径中，逻辑与绝对路径相同，除了第一步：路径不是从 crate 根开始，而是从 `front_of_house` 开始。`front_of_house` 模块定义在与 `eat_at_restaurant` 相同的模块内，所以从定义 `eat_at_restaurant` 的模块开始的相对路径有效。接下来，因为 `hosting` 和 `add_to_waitlist` 标记为 `pub`，路径的其余部分有效，这个函数调用是有效的！

{{#quiz ../quizzes/ch07-03-paths-in-module-tree-1.toml}}

## 使用 `super` 开始相对路径

我们可以通过在路径开头使用 `super`，构造从父模块而不是当前模块或 crate 根开始的相对路径。这就像以 `..` 语法开始文件系统路径一样。使用 `super` 允许我们引用我们知道在父模块中的项，这可以在模块与父模块紧密相关，但父模块将来可能移动到模块树中其他位置时，更容易重新排列模块树。

考虑清单 {{#ref relative-path}} 中的代码，它模拟了厨师修复错误订单并亲自将其带给顾客的情况。定义在 `back_of_house` 模块中的函数 `fix_incorrect_order` 调用定义在父模块中的函数 `deliver_order`，通过指定以 `super` 开头的 `deliver_order` 路径：

<span class="filename">文件名: src/lib.cairo</span>

```cairo,noplayground
fn deliver_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::deliver_order();
    }

    fn cook_order() {}
}
```

{{#label relative-path}} <span class="caption">清单 {{#ref relative-path}}:
使用以 `super` 开头的相对路径调用函数</span>

在这里你可以直接看到你可以使用 `super` 轻松访问父模块，而在以前并非如此。注意 `back_of_house` 保持私有，因为外部用户不应该直接与后厨交互。

## 使结构体和枚举公有

我们也可以使用 `pub` 来指定结构体和枚举为公有，但使用 `pub` 处理结构体和枚举时有一些额外的细节需要考虑。

- 如果我们在结构体定义前使用 `pub`，我们使结构体公有，但结构体的字段仍然是私有的。我们可以根据具体情况使每个字段公有或不公有。
- 相反，如果我们使枚举公有，它的所有变体将是公有的。我们只需要在 `enum` 关键字前加上 `pub`。

还有一种涉及 `pub` 的情况我们没有涉及到，那就是我们最后的模块系统功能：`use` 关键字。我们将首先单独介绍 `use`，然后我们将展示如何结合 `pub` 和 `use`。

{{#quiz ../quizzes/ch07-03-paths-in-module-tree-2.toml}}
