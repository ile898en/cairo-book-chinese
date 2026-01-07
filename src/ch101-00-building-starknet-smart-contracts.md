# 构建 Starknet 智能合约

在上一节中，我们给出了一个用 Cairo 编写的智能合约的入门示例，描述了在 Starknet 上构建智能合约的基本模块。在本节中，我们将逐步深入研究智能合约的所有组件。

当我们讨论 [_接口 (interfaces)_][contract interface] 时，我们指定了两种 _公共函数 (public functions)_ 之间的区别，即 _外部函数 (external functions)_ 和 _视图函数 (view functions)_，并且我们提到了如何与合约的 _存储 (storage)_ 进行交互。

在这一点上，你应该会有多个问题浮现在脑海中：

- 我如何通过存储更复杂的数据类型？
- 我如何定义内部/私有函数？
- 我如何发出事件？我如何索引它们？
- 有没有办法减少样板代码？

幸运的是，我们将在本章回答所有这些问题。

[contract interface]:
  ./ch100-00-introduction-to-smart-contracts.md#the-interface-the-contracts-blueprint
