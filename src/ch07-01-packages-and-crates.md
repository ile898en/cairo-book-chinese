# 包 (Packages) 和 Crates

## 什么是 Crate？

Crate 是在实际 Cairo 编译中使用的包的一个子集。这包括：

- 包源代码，由包名称和 crate 根标识，crate 根是包的主要入口点。
- 包元数据的子集，用于标识 Cairo 编译器的 crate 级设置，例如 _Scarb.toml_ 文件中的 `edition` 字段。

Crates 可以包含模块，并且模块可以定义在与 crate 一起编译的其他文件中，这将在后续章节中讨论。

## 什么是 Crate 根？

Crate 根是 Cairo 编译器开始的 _lib.cairo_ 源文件，构成你的 crate 的根模块。我们将在 [“定义模块以控制作用域”][modules] 一章中深入解释模块。

[modules]: ./ch07-02-defining-modules-to-control-scope.md

## 什么是包？

一个 Cairo 包是一个包含以下内容的目录（或等效物）：

- 带有 `[package]` 部分的 _Scarb.toml_ 清单文件。
- 关联的源代码。

这个定义意味着一个包可能包含其他包，每个包都有一个对应的 _Scarb.toml_ 文件。

## 使用 Scarb 创建包

你可以使用 Scarb 命令行工具创建一个新的 Cairo 包。要创建一个新包，请运行以下命令：

```bash
scarb new my_package
```

此命令将生成一个名为 _my_package_ 的新包目录，结构如下：

```
my_package/
├── Scarb.toml
└── src
    └── lib.cairo
```

- _src/_ 是将存储包的所有 Cairo 源文件的主目录。
- _lib.cairo_ 是 crate 的默认根模块，也是包的主要入口点。
- _Scarb.toml_ 是包清单文件，包含包的元数据和配置选项，例如依赖项、包名称、版本和作者。你可以在 [Scarb 参考][manifest] 上找到关于它的文档。

```toml
[package]
name = "my_package"
version = "0.1.0"
edition = "2024_07"

[executable]

[cairo]
enable-gas = false

[dependencies]
cairo_execute = "2.13.1"
```

在你开发包时，你可能希望将代码组织成多个 Cairo 源文件。你可以通过在 _src_ 目录或其子目录中创建额外的 _.cairo_ 文件来做到这一点。

{{#quiz ../quizzes/ch07-01-packages-crates.toml}}

[manifest]: https://docs.swmansion.com/scarb/docs/reference/manifest.html
