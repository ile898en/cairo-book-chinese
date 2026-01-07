# Cairo 书籍 (The Cairo Book)

_由 Cairo 社区及其 [贡献者](https://github.com/cairo-book/cairo-book.github.io) 编写。特别感谢 [StarkWare](https://starkware.co/) 通过 [OnlyDust](https://www.onlydust.xyz/) 和 [Voyager](https://voyager.online/) 支持本书的创作。_

本文档版本假设你使用的是 [Cairo](https://github.com/starkware-libs/cairo) [版本 2.13.1](https://github.com/starkware-libs/cairo/releases) 和 [Starknet Foundry](https://foundry-rs.github.io/starknet-foundry/index.html) [版本 0.51.1](https://github.com/foundry-rs/starknet-foundry/releases)。请参阅第 {{#chap getting-started}} 章的 [安装](ch01-01-installation.md) 部分以安装或更新 Cairo 和 Starknet Foundry。

这本书是开源的。发现错别字或想做贡献？查看本书的 [GitHub 仓库](https://github.com/cairo-book/cairo-book)。

精通 Cairo 的其他资源：

- [Cairo Playground](https://www.cairo-lang.org/cairovm/)：基于浏览器的 Cairo 游乐场，无需任何设置即可通过编写、编译、调试和证明 Cairo 代码来探索和试验 Cairo。

  > 注意：你可以使用 Cairo Playground 来试验本书的代码片段，并查看它们如何编译成 Sierra（中间表示）和 Casm（Cairo 汇编）。

- [Cairo 核心库文档](https://docs.cairo-lang.org/core?_=60)：Cairo 核心库的文档，核心库是内置于语言中的一组标准类型、trait 和实用程序，提供了 Cairo 生态系统中使用的基本构建块，并在每个 Cairo 项目中自动可用。

- [Cairo 包注册表](https://scarbs.xyz/)：Cairo 不断增长的可重用库集合的主机，包括 [Alexandria](https://github.com/keep-starknet-strange/alexandria)、[Open Zeppelin Contracts for Cairo](https://docs.openzeppelin.com/contracts-cairo/1.0.0/)，所有这些都可以通过 Scarb 轻松集成，从而简化开发和依赖项管理。

- [Scarb 文档](https://docs.swmansion.com/scarb/docs.html)：Cairo 包管理器和构建工具的官方文档，涵盖如何创建和管理包、使用依赖项、运行构建和配置项目。

- [Cairo 白皮书](https://eprint.iacr.org/2021/1063.pdf)：StarkWare 介绍 Cairo 的原始论文，解释了 Cairo 作为一种编写可证明程序的语言，详细介绍了其架构，并展示了它是如何在不依赖可信设置的情况下实现可扩展、可验证计算的。

如果你有任何问题、反馈或意见，可以通过 [Github Issues](https://github.com/cairo-book/cairo-book/issues) 联系，或直接联系 [维护者](https://relens.ai/blog/author/eni)。
