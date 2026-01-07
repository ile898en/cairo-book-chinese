# 简介

## 什么是 Cairo？

Cairo 是一门编程语言，旨在利用数学证明的力量来实现计算完整性（Computational Integrity）。正如 C.S. Lewis 将正直（integrity）定义为“即使在没人注视时也做正确的事”，Cairo 使得程序能够证明它们执行了正确的计算，即使是在不可信的机器上执行时也是如此。

这门语言建立在 STARK 技术之上，这是一种 PCP（概率可检测证明，Probabilistically Checkable Proofs）的现代演进，它将计算主张转化为约束系统。Cairo 的最终目的是生成这些可以被高效且绝对确定地验证的数学证明。

## 你能用它做什么？

Cairo 下开启了我们对可信计算思考方式的范式转变。它目前的主要应用是 Starknet，这是一个以太坊的 Layer 2 扩容解决方案，旨在解决区块链的基本挑战之一：在不牺牲安全性的前提下实现可扩展性。

在传统的区块链模型中，每个参与者都必须验证每一次计算。Starknet 通过使用 Cairo 的证明系统改变了这一点：计算由证明者（Prover）在链下执行并生成 STARK 证明，然后由以太坊智能合约进行验证。这种验证所需的计算能力远低于重新执行计算所需的计算能力，从而在保持安全性的同时实现了巨大的可扩展性。

然而，Cairo 的潜力不仅仅局限于区块链。任何需要高效验证计算完整性的场景都可以从 Cairo 的可验证计算能力中受益。

## 本书的读者对象

本书主要面向三类读者，每类读者都有自己的学习路径：

1.  **通用开发者**：如果你对 Cairo 在区块链之外的可验证计算能力感兴趣，你应该专注于章节 {{#chap getting-started}}-{{#chap advanced-cairo-features}}。这些章节涵盖了核心语言特性和编程概念，而没有深入探讨智能合约的细节。

2.  **新的智能合约开发者**：如果你对 Cairo 和智能合约都是新手，我们要建议从头到尾阅读本书。这将为你打下语言基础和智能合约开发原则的坚实基础。

3.  **有经验的智能合约开发者**：如果你已经熟悉其他语言（或 Rust）的智能合约开发，你可能希望遵循这条专注的路径：
    *   章节 {{#chap getting-started}}-{{#chap common-collections}} 用于了解 Cairo 基础
    *   章节 {{#chap generic-types-and-traits}} 用于了解 Cairo 的 Trait 和泛型系统
    *   跳到章节 {{#chap building-starknet-smart-contracts}} 开始智能合约开发
    *   根据需要参考其他章节

无论你的背景如何，本书假设你具备基本的编程知识，如变量、函数和常见的数据结构。虽然有 Rust 经验会有所帮助（因为 Cairo 有许多相似之处），但这并非必需。

## 参考资料

*   Cairo CPU 架构: <https://eprint.iacr.org/2021/1063>
*   Cairo, Sierra 和 Casm: <https://medium.com/nethermind-eth/under-the-hood-of-cairo-1-0-exploring-sierra-7f32808421f5>
*   非确定性状态: <https://twitter.com/PapiniShahar/status/1638203716535713798>
