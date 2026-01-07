# 合约类 ABI

合约类 _应用程序二进制接口 (Application Binary Interface, ABI)_ 是合约接口的高级规范。它描述了可以调用的函数、它们的预期参数和返回值，以及这些参数和返回值的类型。它允许外部源（包括区块链外部和其他合约）通过根据合约接口对数据进行编码和解码来与合约进行通信。

通常，区块链外部的源使用 ABI 的 JSON 表示形式与合约进行交互。此 JSON 表示形式是从合约类生成的，并且包含一个项目数组，这些项目可以是类型、函数或事件。

另一方面，合约通过 _分发器 (dispatcher)_ 模式直接在 Cairo 中使用另一个合约的 ABI，这是一种实现调用另一个合约函数的方法的特定类型。这些方法是自动生成的，并且包含对发送到合约的数据进行编码和解码所需的全部逻辑。

当你使用像 [Voyager][voyager] 或 [Starkscan][starkscan] 这样的区块浏览器与智能合约交互时，JSON ABI 用于正确编码你发送给合约的数据并解码它返回的数据。

[voyager]: https://voyager.online/
[starkscan]: https://starkscan.co/

## 入口点 (Entrypoints)

合约 ABI 中公开的所有函数都称为 _入口点_。入口点是可以从合约类外部调用的函数。

Starknet 合约中有 3 种不同类型的入口点：

- [公共函数][public function]，最常见的入口点，根据其状态可变性作为 `view` 或 `external` 公开。

> 注意：入口点可以标记为 `view`，但如果合约使用不可变性未由编译器强制执行的低级调用，则在随交易一起调用时仍可能修改合约的状态。

- 可选的唯一 [_构造函数_][constructor]，这是一个特定的入口点，仅在合约部署期间调用一次。

- L1 处理程序 (L1-Handlers)，这些函数只能由定序器在收到来自 L1 网络的包含调用合约指令的 [消息][L1-L2 messaging] 后触发。

[public function]: ./ch101-02-contract-functions.md#2-public-functions
[constructor]: ./ch101-02-contract-functions.md#1-constructors
[L1-L2 messaging]: ./ch103-04-L1-L2-messaging.md

在 Cairo 合约类中，函数入口点由 _选择器 (selector)_ 和 `function_idx` 表示。

## 函数选择器 (Function Selector)

虽然函数是用名称定义的，但入口点是由它们的 _选择器_ 标识的。选择器是从函数名称派生的唯一标识符，简单地计算为 `sn_keccak(function_name)`。由于在 Cairo 中无法使用不同参数重载函数，因此函数名称的哈希足以唯一标识要调用的函数。

虽然此过程通常由库和使用分发器时抽象化，但要知道可以通过提供其选择器直接调用函数，例如在使用低级系统调用如 `starknet::call_contract_syscall` 或与 RPC 交互时。

## 编码 (Encoding)

智能合约是用像 Cairo 这样的高级语言编写的，使用强类型来告知我们要处理的数据。然而，在区块链上执行的代码被编译成低级 CASM 指令序列。Starknet 中的基本数据类型是 `felt252`，这也是 CASM 级别操作的唯一数据。因此，所有数据必须在发送到合约之前序列化为 `felt252`。ABI 指定了类型如何编码为 `felt252` 序列，以及如何解码回其原始形式。
