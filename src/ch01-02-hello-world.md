# Hello, World

既然你已经通过 Scarb 安装了 Cairo，是时候编写你的第一个 Cairo 程序了。学习一门新语言的传统是写一个小程序，把文本 `Hello, world!` 输出到屏幕上，我们也照做！

> 注意：本书假设你对命令行有基本的了解。Cairo 对你的编辑器或工具或代码的位置没有特殊要求，所以如果你喜欢使用集成开发环境 (IDE) 而不是命令行，请随意使用你喜欢的 IDE。Cairo 团队开发了一个 Cairo 语言的 VSCode 扩展，你可以用它来获得语言服务器的功能和代码高亮。详见 [附录 F][devtools]。

[devtools]: ./appendix-06-useful-development-tools.md

## 创建项目目录

首先创建一个目录来存储你的 Cairo 代码。Cairo 不在乎你的代码在哪里，但为了本书的练习和项目，我们建议在你的主目录中创建一个 _cairo_projects_ 目录，并将所有项目都放在那里。

打开终端并输入以下命令来创建 _cairo_projects_ 目录。

对于 Linux, macOS 和 Windows PowerShell，输入：

```shell
mkdir ~/cairo_projects
cd ~/cairo_projects
```

对于 Windows CMD，输入：

```cmd
> mkdir "%USERPROFILE%\cairo_projects"
> cd /d "%USERPROFILE%\cairo_projects"
```

> 注意：从现在开始，对于书中展示的每个例子，我们假设你将在一个 Scarb 项目目录中工作。如果你不使用 Scarb，并尝试从不同的目录运行示例，你可能需要相应地调整命令或创建一个 Scarb 项目。

## 使用 Scarb 创建项目

让我们使用 Scarb 创建一个新项目。

导航到你的 _cairo_projects_ 目录（或者你决定存储代码的任何地方）。然后运行：

```bash
scarb new hello_world
```

Scarb 会询问你想要添加哪些依赖。你会看到两个选项：

```text
? Which test runner do you want to set up? ›
❯ Starknet Foundry (default)
  Cairo Test
```

一般来说，我们首选使用第一个 `❯ Starknet Foundry (default)`。

这将创建一个名为 _hello_world_ 的新目录和项目。我们将项目命名为 _hello_world_，Scarb 也在同名目录中创建了文件。

进入 _hello_world_ 目录（`cd hello_world`）。你会看到 Scarb 为我们生成了三个文件和两个目录：一个 _Scarb.toml_ 文件，一个包含 _lib.cairo_ 文件的 _src_ 目录，以及一个包含 _test_contract.cairo_ 文件的 _tests_ 目录。目前，我们可以删除这个 _tests_ 目录。

它还初始化了一个新的 Git 仓库以及一个 `.gitignore` 文件。

> 注意：Git 是一个常见的版本控制系统。你可以通过使用 `--no-vcs` 标志来停止使用版本控制系统。运行 `scarb new --help` 查看可用选项。

在你的文本编辑器中打开 _Scarb.toml_。它应该类似于清单 {{#ref scarb-content}} 中的代码。

<span class="filename">文件名: Scarb.toml</span>

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024_07"

# See more keys and their definitions at https://docs.swmansion.com/scarb/docs/reference/manifest.html

[dependencies]
starknet = "2.13.1"

[dev-dependencies]
snforge_std = "0.51.1"
assert_macros = "2.13.1"

[[target.starknet-contract]]
sierra = true

[scripts]
test = "snforge test"

# ...
```

{{#label scarb-content}} <span class="caption">清单 {{#ref scarb-content}}:
由 `scarb new` 生成的 _Scarb.toml_ 内容</span>

这个文件使用 [TOML][toml doc] (Tom’s Obvious, Minimal Language) 格式，这是 Scarb 的配置格式。

第一行 `[package]` 是一个部分标题，表示接下来的语句是在配置一个包。随着我们向文件添加更多信息，我们将添加其他部分。

接下来的三行设置了 Scarb 编译你的程序所需的配置信息：包的名称、要使用的 Scarb 版本以及要使用的预设 (prelude) 版本。预设是自动导入到每个 Cairo 程序中的最常用项目的集合。你可以在 [附录 D][prelude] 中了解更多关于预设的信息。

`[dependencies]` 部分是一个供你去列出任何项目依赖项的区域。在 Cairo 中，代码包被称为 crate。在这个项目中我们需要任何其他 crate。

`[dev-dependencies]` 部分是关于开发所需的依赖项，但对于项目的实际生产构建是不需要的。`snforge_std` 和 `assert_macros` 就是这种依赖项的两个例子。如果你想在不使用 Starknet Foundry 的情况下测试你的项目，你可以使用 `cairo_test`。

`[[target.starknet-contract]]` 部分允许构建 Starknet 智能合约。我们可以暂时删除它。

`[script]` 部分允许定义自定义脚本。默认情况下，有一个使用 `snforge` 运行测试的脚本 `scarb test`。我们也可以暂时删除它。

Starknet Foundry 还有更多选项，请查看 [Starknet Foundry 文档](https://foundry-rs.github.io/starknet-foundry/appendix/scarb-toml.html) 了解更多信息。

默认情况下，使用 Starknet Foundry 会添加 `starknet` 依赖项和 `[[target.starknet-contract]]` 部分，以便你可以开箱即用地为 Starknet 构建合约。我们将从纯 Cairo 程序开始，所以你可以编辑你的 _Scarb.toml_ 文件如下：

<span class="filename">文件名: Scarb.toml</span>

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024_07"

[cairo]
enable-gas = false

[dependencies]
cairo_execute = "2.13.1"


[[target.executable]]
name = "hello_world_main"
function = "hello_world::hello_world::main"
```

{{#label modified-scarb-content}} <span class="caption">清单
{{#ref modified-scarb-content}}: 修改后的 _Scarb.toml_ 内容</span>

Scarb 创建的另一个文件是 _src/lib.cairo_，让我们删除所有内容并在其中放入以下内容，我们将稍后解释原因。

```cairo,noplayground
mod hello_world;
```

然后创建一个名为 _src/hello_world.cairo_ 的新文件，并将以下代码放入其中：

<span class="filename">文件名: src/hello_world.cairo</span>

```cairo
#[executable]
fn main() {
    println!("Hello, World!");
}
```

我们刚刚创建了一个名为 _lib.cairo_ 的文件，其中包含引用另一个名为 `hello_world` 模块的模块声明，以及 _hello_world.cairo_ 文件，其中包含 `hello_world` 模块的实现细节。

Scarb 要求你的源文件位于 _src_ 目录中。

顶层项目目录保留给 _README_ 文件、许可信息、配置文件和任何其他非代码相关的内容。Scarb 确保所有项目组件都有指定的位置，从而保持结构化的组织。

如果你开始了一个不使用 Scarb 的项目，你可以将其转换为使用 Scarb 的项目。将项目代码移动到 _src_ 目录中，并创建一个合适的 _Scarb.toml_ 文件。你也可以使用 `scarb init` 命令来生成 _src_ 文件夹及其包含的 _Scarb.toml_。

```txt
├── Scarb.toml
├── src
│   ├── lib.cairo
│   └── hello_world.cairo
```

<span class="caption"> 示例 Scarb 项目结构</span>

[toml doc]: https://toml.io/
[prelude]: ./appendix-04-cairo-prelude.md
[starknet package]: https://docs.swmansion.com/scarb/docs/extensions/starknet/starknet-package.html

## 构建 Scarb 项目

在你的 _hello_world_ 目录下，输入以下命令构建你的项目：

```bash
$ scarb build 
   Compiling hello_world v0.1.0 (listings/ch01-getting-started/no_listing_01_hello_world/Scarb.toml)
    Finished `dev` profile target(s) in 1 second

```

该命令在 _target/dev_ 中创建了一个 `hello_world_main_executable.json` 文件，我们暂时忽略这个文件。

如果你正确安装了 Cairo，你应该能够使用 `scarb execute` 命令运行程序的 `main` 函数，并看到以下输出：

```shell
$ scarb execute 
   Compiling hello_world v0.1.0 (listings/ch01-getting-started/no_listing_01_hello_world/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing hello_world
Hello, World!


```

无论你的操作系统是什么，字符串 `Hello, world!` 都应该打印到终端。

如果 `Hello, world!` 打印出来了，恭喜！你已经正式编写了一个 Cairo 程序。这使你成为了一名 Cairo 程序员——欢迎！

## 剖析 Cairo 程序

让我们详细回顾一下这个 "Hello, world!" 程序。这是拼图的第一块：

```cairo,noplayground
fn main() {

}
```

这些行定义了一个名为 `main` 的函数。`main` 函数很特殊：它总是每个可执行 Cairo 程序中运行的第一个代码。在这里，第一行声明了一个名为 `main` 的函数，它没有参数也不返回任何内容。如果有参数，它们会放在括号 `()` 里。

函数体被包裹在 `{}` 中。Cairo 要求所有函数体都要用花括号括起来。良好的风格是将左花括号放在函数声明的同一行，中间加一个空格。

> 注意：如果你想在 Cairo 项目中保持标准风格，可以使用 `scarb fmt` 提供的自动格式化工具将代码格式化为特定风格（更多关于 `scarb fmt` 的信息在 [附录 F][devtools] 中）。Cairo 团队已将此工具包含在标准 Cairo 发行版中，就像 `cairo-run` 一样，所以它应该已经安装在你的电脑上了！

`main` 函数的主体包含以下代码：

```cairo,noplayground
    println!("Hello, World!");
```

这行代码在这个小程序中完成了所有工作：它将文本打印到屏幕上。这里有四个重要的细节需要注意。

首先，Cairo 的风格是使用四个空格缩进，而不是制表符 (tab)。

其次，`println!` 调用了一个 Cairo 宏。如果它调用的是函数，它会被输入为 `println`（并没有 `!`）。我们将在 ["宏"][macros] 一章更详细地讨论 Cairo 宏。现在，你只需要知道使用 `!` 意味着你在调用宏而不是普通函数，并且宏并不总是遵循与函数相同的规则。

第三，你看到了 `"Hello, world!"` 字符串。我们将此字符串作为参数传递给 `println!`，然后字符串被打印到屏幕上。

第四，我们以分号 (`;`) 结束该行，这表明该表达式结束，下一个表达式准备开始。大多数 Cairo 代码行以分号结尾。

[devtools]: ./appendix-06-useful-development-tools.md
[macros]: ./ch12-05-macros.md

{{#quiz ../quizzes/ch01-02-hello-world.toml}}

# 总结

让我们回顾一下目前为止关于 Scarb 我们学到了什么：

- 我们可以使用 `asdf` 安装一个或多个 Scarb 版本，无论是最新稳定版还是特定版本。
- 我们可以使用 `scarb new` 创建项目。
- 我们可以使用 `scarb build` 构建项目以生成编译后的 Sierra 代码。
- 我们可以使用 `scarb execute` 命令执行 Cairo 程序。

使用 Scarb 的另一个优点是，无论你在哪个操作系统上工作，命令都是相同的。因此，在这一点上，我们将不再分别为 Linux/macOS 和 Windows 提供具体说明。

你在 Cairo 之旅上已经有了一个很好的开始！现在是构建一个更实质性的程序来习惯阅读和编写 Cairo 代码的好时机。
