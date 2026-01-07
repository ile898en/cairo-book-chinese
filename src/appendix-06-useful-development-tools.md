# 附录 F - 有用的开发工具

在本附录中，我们将讨论 Cairo 项目提供的一些有用的开发工具。我们将了解自动格式化、应用警告修复的快速方法、linter 以及与 IDE 的集成。

## 使用 `scarb fmt` 自动格式化

可以使用 `scarb fmt` 命令格式化 Scarb 项目。如果你直接使用 Cairo 二进制文件，则可以运行 `cairo-format`。许多协作项目使用 `scarb fmt` 来防止关于编写 Cairo 时使用哪种风格的争论：每个人都使用该工具格式化他们的代码。

要格式化任何 Cairo 项目，请在项目目录中输入以下内容：

```bash
scarb fmt
```

对于你不希望 `scarb fmt` 破坏的内容，请使用 `#[cairofmt::skip]`：

```cairo, noplayground
#[cairofmt::skip]
let table: Array<ByteArray> = array![
    "oxo",
    "xox",
    "oxo",
];
```

## 使用 `cairo-language-server` 进行 IDE 集成

为了帮助 IDE 集成，Cairo 社区建议使用 [`cairo-language-server`][cairo-language-server]<!-- ignore -->。此工具是一组以编译器为中心的实用程序，使用 [语言服务器协议 (Language Server Protocol)][lsp]<!--
ignore -->，这是 IDE 和编程语言相互通信的规范。不同的客户端可以使用 `cairo-language-server`，例如 [Visual Studio Code 的 Cairo 扩展][vscode-cairo]。

[lsp]: http://langserver.org/
[vscode-cairo]:
  https://marketplace.visualstudio.com/items?itemName=starkware.cairo1

访问 `vscode-cairo` [页面][vscode-cairo]<!-- ignore --> 以在 VSCode 上安装它。你将获得自动完成、跳转到定义和内联错误等功能。

[cairo-language-server]: https://github.com/software-mansion/cairols

> 注意：如果你安装了 Scarb，它应该可以与 Cairo VSCode 扩展开箱即用，无需手动安装语言服务器。
