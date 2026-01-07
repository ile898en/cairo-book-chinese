# Starknet 合约类 (Classes) 和实例 (Instances)

与面向对象编程一样，Starknet 通过将合约分为 [类 (classes)](#contract-classes) 和 [实例 (instances)](#contract-instances) 来区分合约及其实现。合约类是合约的定义，而合约实例是对应于类的一个已部署合约。只有合约实例充当真正的合约，因为它们拥有自己的存储，并可以被交易或其他合约调用。

<!-- > Note: A contract class does not necessarily require a deployed instance in Starknet. -->

## 合约类 (Contract Classes)

### Cairo 类定义的组成部分

定义类的组件包括：

| 名称                          | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 合约类版本                    | 合约类对象的版本。目前，Starknet OS 支持版本 0.1.0。                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| 外部函数入口点数组            | 入口点是一对 `(_selector_, _function_idx_)`，其中 `_function_idx_` 是 Sierra 程序内部函数的索引。选择器 (selector) 是一个标识符，通过它可以在交易或其他类中调用函数。选择器是函数名称的 `starknet_keccak` 哈希，以 ASCII 编码。                                                                                                                                                                                                                                                                                            |
| L1 处理程序入口点数组 (L1 Handlers)         | -                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 构造函数入口点数组            | 目前，编译器只允许一个构造函数。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ABI                           | 表示类 ABI 的字符串。ABI 哈希（影响类哈希）由 `starknet_keccak(bytes(ABI, "UTF-8"))` 给出。此字符串由声明类的用户提供（并作为 `DECLARE` 交易的一部分进行签名），并不强制要求是关联类的真实 ABI。如果不查看底层源代码（即生成该类 Sierra 的 Cairo 代码），此 ABI 应被视为声明方“预期”的 ABI，这可能是不正确的（有意或无意）。“诚实”的字符串将是 Cairo 编译器生成的合约 ABI 的 JSON 序列化。 |
| Sierra 程序                   | 表示 Sierra 指令的字段元素数组。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

### 类哈希 (Class Hash)

每个类都由其 _类哈希_ 唯一标识，类似于传统面向对象编程语言中的类名。类的哈希是其组件的链式哈希，计算如下：

```
class_hash = h(
    contract_class_version,
    external_entry_points,
    l1_handler_entry_points,
    constructor_entry_points,
    abi_hash,
    sierra_program_hash
)
```

其中：

- `h` 是 Poseidon 哈希函数
- 入口点数组 \\( (selector,index)\_{i=1}^n] \\) 的哈希由 \\( h(\text{selector}\_1,\text{index}\_1,...,\text{selector}\_n,\text{index}\_n) \\) 给出
- `sierra_program_hash` 是程序字节码数组的 Poseidon 哈希

> 注意：Starknet OS 目前支持合约类版本 0.1.0，在上述哈希计算中表示为字符串 `CONTRACT_CLASS_V0.1.0` 的 ASCII 编码（以这种方式对版本进行哈希可以让我们在类哈希和其他对象哈希之间进行域分离）。有关更多详细信息，请参阅 [Cairo 实现](https://github.com/starkware-libs/cairo-lang/blob/7712b21fc3b1cb02321a58d0c0579f5370147a8b/src/starkware/starknet/core/os/contracts.cairo#L47)。

### 使用类

- 添加新类：要向 Starknet 的状态引入新类，请使用 `DECLARE` 交易。
- 部署实例：要部署先前声明的类的新实例，请使用 `deploy` 系统调用。
- 使用类功能：要在不部署实例的情况下使用已声明类的功能，请使用 `library_call` 系统调用。类似于以太坊的 `delegatecall`，它使你能够使用现有类中的代码而无需部署合约实例。

## 合约实例 (Contract Instances)

### 合约 Nonce

合约实例具有 nonce，其值为源自此地址的交易数量加 1。例如，当你使用 `DEPLOY_ACCOUNT` 交易部署账户时，交易中账户合约的 nonce 为 `0`。在 `DEPLOY_ACCOUNT` 交易之后，直到账户合约发送下一笔交易，nonce 为 `1`。

### 合约地址

合约地址是 Starknet 上合约的唯一标识符。它是以下信息的链式哈希：

- `prefix`: 常量字符串 `STARKNET_CONTRACT_ADDRESS` 的 ASCII 编码。
- `deployer_address`，它是：
  - 当合约通过 `DEPLOY_ACCOUNT` 交易部署时为 `0`
  - 当合约通过 `deploy` 系统调用部署时，由 `deploy_from_zero` 参数的值决定
    > 注意：有关 `deploy_from_zero` 参数的信息，请参阅 [`deploy` 系统调用]()。
- `salt`: 合约调用 syscall 时传递的盐，由交易发送者提供。
- `class_hash`: 见 [类哈希](#class-hash)。
- `constructor_calldata_hash`: 构造函数输入的数组哈希。

地址计算如下：

```
contract_address = pedersen(
    “STARKNET_CONTRACT_ADDRESS”,
    deployer_address,
    salt,
    class_hash,
    constructor_calldata_hash)
```

> 注意：随机 `salt` 确保智能合约部署的地址唯一，防止部署相同合约类时发生冲突。它还通过唯一发送者地址影响交易哈希来阻止重放攻击。

有关地址计算的更多信息，请参阅 Cairo GitHub 存储库中的 [`contract_address.cairo`](https://github.com/starkware-libs/cairo/blob/2c96b181a6debe9a564b51dbeaaf48fa75808d53/corelib/src/starknet/contract_address.cairo)。
