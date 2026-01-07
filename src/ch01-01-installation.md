# 安装

第一步是安装 Cairo。我们将通过 [starkup][starkup] 下载 Cairo，这是一个用于管理 Cairo 版本和相关工具的命令行工具。下载过程中你需要保持网络连接。

接下来的步骤将通过名为 [Scarb][scarb doc] 的二进制文件安装 Cairo 编译器的最新稳定版。Scarb 将 Cairo 编译器和 Cairo 语言服务器捆绑在一个易于安装的包中，这样你就可以立即开始编写 Cairo 代码了。

Scarb 也是 Cairo 的包管理器，它的灵感很大程度上来自于 [Cargo][cargo doc]——Rust 的构建系统和包管理器。

Scarb 为你处理许多任务，例如构建你的代码（无论是纯 Cairo 代码还是 Starknet 合约）、下载你的代码依赖的库、构建这些库，并为 VSCode Cairo 1 扩展提供 LSP 支持。

随着你编写更复杂的 Cairo 程序，你可能会添加依赖项，如果你使用 Scarb 启动项目，管理外部代码和依赖项将变得容易得多。

[Starknet Foundry][sn foundry] 是用于 Cairo 程序和 Starknet 智能合约开发的工具链。它支持许多功能，包括编写和运行具有高级特性的测试、部署合约、与 Starknet 网络交互等等。

让我们从安装 starkup 开始，它将帮助我们管理 Cairo、Scarb 和 Starknet Foundry。

[starkup]: https://github.com/software-mansion/starkup
[scarb doc]: https://docs.swmansion.com/scarb/docs
[cargo doc]: https://doc.rust-lang.org/cargo/
[sn foundry]: https://foundry-rs.github.io/starknet-foundry/index.html

## 在 Linux 或 macOS 上安装 `starkup`

如果你使用的是 Linux 或 macOS，请打开终端并输入以下命令：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.starkup.dev | sh
```

该命令会下载一个脚本并开始安装 starkup 工具，它会安装最新稳定版的 Cairo 及相关工具。系统可能会提示你输入密码。如果安装成功，将出现以下行：

```bash
starkup: Installation complete.
```

安装后，starkup 会自动安装最新稳定版的 Cairo、Scarb 和 Starknet Foundry。你可以通过在新的终端会话中运行以下命令来验证安装：

```bash
$ scarb --version
scarb 2.13.1 (639d0a65e 2025-08-04)
cairo: 2.13.1 (https://crates.io/crates/cairo-lang-compiler/2.13.1)
sierra: 1.7.0

$ snforge --version
snforge 0.51.1
```

我们将在 [第 {{#chap testing-cairo-programs}} 章][writing tests]（Cairo 程序测试）以及 [第 {{#chap starknet-smart-contracts-security}} 章][testing with snfoundry]（讨论 Starknet 智能合约测试和安全性）中更详细地描述 Starknet Foundry。

[writing tests]: ./ch10-01-how-to-write-tests.md
[testing with snfoundry]: ./ch104-02-testing-smart-contracts.md#testing-smart-contracts-with-starknet-foundry

## 安装 VSCode 扩展

Cairo 有一个 VSCode 扩展，提供语法高亮、代码补全和其他有用的功能。你可以从 [VSCode Marketplace][vsc extension] 安装它。安装后，进入扩展设置，确保勾选 `Enable Language Server` 和 `Enable Scarb` 选项。

[vsc extension]: https://marketplace.visualstudio.com/items?itemName=starkware.cairo1

{{#quiz ../quizzes/ch01-01-installation.toml}}
