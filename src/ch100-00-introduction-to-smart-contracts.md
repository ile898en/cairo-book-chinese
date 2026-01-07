---
meta:
  "在这个为使用 Cairo 的开发人员量身定制的全面介绍中探索智能合约的基础知识。了解智能合约如何工作以及它们在区块链技术中的作用。"
---

# 智能合约简介

本章将为你提供关于什么是智能合约、它们的用途以及为什么区块链开发人员会使用 Cairo 和 Starknet 的高级介绍。如果你已经熟悉区块链编程，请随意跳过本章。不过，最后一部分可能仍然很有趣。

## 智能合约 - 简介

随着以太坊的诞生，智能合约变得越来越流行和普及。智能合约本质上是部署在区块链上的程序。“智能合约”这个术语有点误导，因为它们既不“智能”也不完全是“合约”，而是根据特定输入执行的代码和指令。它们主要由两个组件组成：存储和函数。一旦部署，用户可以通过启动包含执行数据（调用哪个函数以及使用什么输入）的区块链交易来与智能合约交互。智能合约可以修改和读取底层区块链的存储。智能合约有自己的地址，被认为是一个区块链账户，这意味着它可以持有代币。

用于编写智能合约的编程语言因区块链而异。例如，在以太坊和 [兼容 EVM 的生态系统][evm] 上，最常用的语言是 Solidity，而在 Starknet 上，它是 Cairo。代码编译的方式也根据区块链而有所不同。在以太坊上，Solidity 被编译成字节码。在 Starknet 上，Cairo 被编译成 Sierra，然后编译成 Cairo 汇编 (CASM)。

智能合约拥有几个独特的特征。它们是 **无需许可的 (permissionless)**，这意味着任何人都可以网络上部署智能合约（当然是在去中心化区块链的背景下）。智能合约也是 **透明的 (transparent)**；任何人都可以访问智能合约存储的数据。组成合约的代码也可以是透明的，从而实现 **可组合性 (composability)**。这允许开发人员编写使用其他智能合约的智能合约。智能合约只能访问和交互其部署所在的区块链上的数据。它们需要第三方软件（称为 _预言机 Oracles_）来访问外部数据（例如代币的价格）。

为了让开发人员构建可以相互交互的智能合约，需要知道其他合约是什么样子的。因此，以太坊开发人员开始为智能合约开发建立标准，即 `ERCxx`。两个最常用和著名的标准是 `ERC20`（用于构建像 `USDC`、`DAI` 或 `STARK` 这样的代币）和 `ERC721`（用于像 `CryptoPunks` 或 `Everai` 这样的 NFT（非同质化代币））。

[evm]: https://ethereum.org/en/developers/docs/evm/

## 智能合约 - 用例

智能合约有许多可能的用例。唯一的限制是区块链的技术约束和开发人员的创造力。

### DeFi

目前，智能合约的主要用例类似于以太坊或比特币，本质上是处理资金。在比特币承诺的替代支付系统的背景下，以太坊上的智能合约使得创建不再依赖传统金融中介的去中心化金融应用程序成为可能。这就是我们要说的 DeFi（去中心化金融）。DeFi 由各种项目组成，如借贷应用程序、去中心化交易所 (DEX)、链上衍生品、稳定币、去中心化对冲基金、保险等等。

### 代币化 (Tokenization)

智能合约可以促进现实世界资产的代币化，例如房地产、艺术品或贵金属。代币化将资产分割成数字代币，可以在区块链平台上轻松交易和管理。这可以增加流动性，实现部分所有权，并简化买卖过程。

### 投票

智能合约可用于创建安全透明的投票系统。投票可以记录在区块链上，确保不可篡改和透明度。智能合约可以自动统计选票并宣布结果，最大限度地减少欺诈或操纵的可能性。

### 版税 (Royalties)

智能合约可以为艺术家、音乐家和其他内容创作者自动支付版税。当消费或出售一份内容时，智能合约可以自动计算并将版税分发给合法的所有者，确保公平的报酬并减少对中介的需求。

### 去中心化身份 (DIDs)

智能合约可用于创建和管理数字身份，允许个人控制其个人信息并安全地与第三方共享。智能合约可以验证用户身份的真实性，并根据用户的凭据自动授予或撤销对特定服务的访问权限。

<br/>
<br/>
随着以太坊的不断成熟，我们可以预期智能合约的用例和应用将进一步扩展，带来令人兴奋的新机会并重塑传统系统，使其变得更好。

## Starknet 和 Cairo 的兴起

以太坊作为最广泛使用和最具弹性的智能合约平台，成为了其自身成功的受害者。随着一些前面提到的用例（主要是 DeFi）的迅速采用，执行交易的成本变得极高，使得网络几乎无法使用。生态系统中的工程师和研究人员开始致力于解决这种可扩展性问题的解决方案。

区块链领域一个著名的三难困境（区块链不可能三角）指出，很难同时实现高水平的可扩展性、去中心化和安全性；必须做出权衡。以太坊处于去中心化和安全性的交叉点。最终决定以太坊的目的将是作为一个安全的结算层，而复杂的计算将被转移到建立在以太坊之上的其他网络上。这些被称为 Layer 2 (L2)。
L2 主要有两种类型：乐观汇总 (optimistic rollups) 和有效性汇总 (validity rollups)。这两种方法都涉及将大量交易压缩和批处理在一起，计算新状态，并将结果结算在以太坊 (L1) 上。区别在于结果在 L1 上结算的方式。对于乐观汇总，新状态默认被认为是有效的，但有一个 7 天的窗口期供节点识别恶意交易。

相比之下，有效性汇总（如 Starknet）使用密码学来证明新状态已被正确计算。这就是 STARKs 的目的，这种密码学技术可以允许有效性汇总比乐观汇总更显著地扩展。你可以从 Starkware 的 Medium [文章][starks article] 中了解有关 STARKs 的更多信息，这是一个很好的入门读物。

> Starknet 的架构在 [Starknet 文档](https://docs.starknet.io/documentation/architecture_and_concepts/) 中有详细描述，这是了解更多关于 Starknet 网络的绝佳资源。

还记得 Cairo 吗？事实上，它是一种专门为了与 STARKs 一起工作并使它们具有通用性而开发的语言。使用 Cairo，我们可以编写 **可证明代码 (provable code)**。在 Starknet 的背景下，这允许证明从一个状态到另一个状态的计算的正确性。

与大多数（如果不是全部）选择使用 EVM（按原样或稍作修改）作为基础层的 Starknet 竞争对手不同，Starknet 采用了自己的 VM。这使开发人员从 EVM 的限制中解放出来，开辟了更广泛的可能性。加上交易成本的降低，Starknet 和 Cairo 的结合为开发人员创造了一个令人兴奋的游乐场。原生账户抽象为账户（我们称之为“智能账户”）和交易流实现了更复杂的逻辑。新兴的用例包括 **透明 AI** 和机器学习应用程序。最后，**区块链游戏** 可以完全 **在链上** 开发。Starknet 经过专门设计，旨在最大化 STARK 证明的功能，以此实现最佳的可扩展性。

> 在 [Starknet 文档](https://docs.starknet.io/documentation/architecture_and_concepts/Account_Abstraction/introduction/) 中了解有关账户抽象的更多信息。

[starks article]:
  https://medium.com/starkware/starks-starkex-and-starknet-9a426680745a

## Cairo 程序和 Starknet 智能合约：有什么区别？

Starknet 合约是 Cairo 程序的一个特殊超集，因此本书前面学到的概念仍然适用于编写 Starknet 合约。正如你可能已经注意到的，Cairo 程序必须始终有一个用作此程序入口点的 `main` 函数：

```cairo
fn main() {}
```

部署在 Starknet 网络上的合约本质上是由定序器 (sequencer) 运行的程序，因此可以访问 Starknet 的状态。合约没有 `main` 函数，但有一个或多个可用作入口点的函数。

Starknet 合约在 [模块][module chapter] 中定义。为了让模块被编译器作为合约处理，它必须用 `#[starknet::contract]` 属性进行注释。

[module chapter]: ./ch07-02-defining-modules-to-control-scope.md

## 简单合约剖析

本章将使用一个非常简单的智能合约作为示例，向你介绍 Starknet 合约的基础知识。你将学习如何编写一个允许任何人在 Starknet 区块链上存储单个数字的合约。

让我们在整章中考虑以下合约。要把它们一次全部理解可能并不容易，但我们会一步一步地进行：

```cairo,noplayground
#[starknet::interface]
trait ISimpleStorage<TContractState> {
    fn set(ref self: TContractState, x: u128);
    fn get(self: @TContractState) -> u128;
}

#[starknet::contract]
mod SimpleStorage {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        stored_data: u128,
    }

    #[abi(embed_v0)]
    impl SimpleStorage of super::ISimpleStorage<ContractState> {
        fn set(ref self: ContractState, x: u128) {
            self.stored_data.write(x);
        }

        fn get(self: @ContractState) -> u128 {
            self.stored_data.read()
        }
    }
}
```

{{#label simple-contract}} <span class="caption">清单 {{#ref simple-contract}}: 一个简单的存储合约</span>

### 这个合约是什么？

合约是通过将状态和逻辑封装在一个用 `#[starknet::contract]` 属性注释的模块中来定义的。

状态是在 `Storage` 结构体中定义的，并且总是被初始化为空。在这里，我们的结构体包含一个名为 `stored_data` 的字段，类型为 `u128`（128 位无符号整数），表明我们的合约可以存储 0 到 \\( {2^{128}} - 1 \\) 之间的任何数字。

逻辑是由与状态交互的函数定义的。在这里，我们的合约定义并公开暴露了 `set` 和 `get` 函数，可用于修改或检索存储变量的值。你可以把它想象成数据库中的单个插槽，你可以通过调用管理数据库的代码的函数来查询和修改它。

### 接口：合约的蓝图

```cairo,noplayground
#[starknet::interface]
trait ISimpleStorage<TContractState> {
    fn set(ref self: TContractState, x: u128);
    fn get(self: @TContractState) -> u128;
}
```

{{#label interface}} <span class="caption">清单 {{#ref interface}}: 一个基本的合约接口</span>

接口代表合约的蓝图。它们定义了合约向外部世界公开的函数，而不包括函数体。在 Cairo 中，它们是通过用 `#[starknet::interface]` 属性注释 trait 来定义的。该 trait 的所有函数都被视为实现此 trait 的任何合约的公共函数，并且可以从外部世界调用。

> 合约构造函数不是接口的一部分。内部函数也不是。

所有合约接口都使用通用类型作为 `self` 参数，代表合约状态。我们在接口中选择将此通用参数命名为 `TContractState`，但这不是强制性的，可以选择任何名称。

在我们的接口中，注意 `self` 参数的通用类型 `TContractState` 是通过引用传递给 `set` 函数的。看到 `self` 参数如果不动地传入合约函数告诉我们该函数可以访问合约的状态。`ref` 修饰符暗示 `self` 可能会被修改，这意味着合约的存储变量可能会在 `set` 函数内部被修改。

另一方面，`get` 函数采用 `TContractState` 的快照，这立即告诉我们它不修改状态（实际上，如果我们试图在 `get` 函数内修改存储，编译器会报错）。

通过利用 Cairo 的 [traits & impls](./ch08-02-traits-in-cairo.md) 机制，我们可以确保存好的实际实现与其接口匹配。事实上，如果你的合约不符合声明的接口，你将得到一个编译错误。例如，清单 {{#ref wrong-impl}} 显示了 `ISimpleStorage` 接口的一个错误实现，包含一个稍微不同的 `set` 函数，其签名不同。

```cairo,noplayground
    #[abi(embed_v0)]
    impl SimpleStorage of super::ISimpleStorage<ContractState> {
        fn set(ref self: ContractState, x: u256) {
            self.stored_data.write(x);
        }

        fn get(self: @ContractState) -> u128 {
            self.stored_data.read()
        }
    }
```

{{#label wrong-impl}} <span class="caption">清单 {{#ref wrong-impl}}: 合约接口的错误实现。这无法编译，因为 `set` 的签名与 trait 不匹配。</span>

尝试使用此实现编译合约将导致以下错误：

```shell
error: Interaction with the trait ISimpleStorage resulted in a ppanic.
 --> contract.cairo:4:1
trait ISimpleStorage<TContractState> {
^************************************^

error: The value of the associated item "set" in the trait implementation mismatch the value in the trait.
 --> contract.cairo:17:5
    impl SimpleStorage of super::ISimpleStorage<ContractState>{
    ^*********************************************************^

```

### 在实现块中定义公共函数

在我们进一步探讨之前，让我们定义一些术语。

- 在 Starknet 的语境中，_公共函数 (public function)_ 是向外部世界公开的函数。公共函数可以被任何人调用，无论是从合约外部还是从合约内部。在上面的例子中，`set` 和 `get` 是公共函数。

- 我们所说的 _外部 (external)_ 函数是一种公共函数，可以通过 Starknet 交易直接调用，并且可以改变合约的状态。`set` 是一个外部函数。

- _视图 (view)_ 函数是一种公共函数，通常是只读的，不能改变合约的状态。然而，这个限制只是由编译器强制执行的，而不是由 Starknet 本身强制执行的。我们将在后面的章节讨论其含义。`get` 是一个视图函数。

```cairo,noplayground
    #[abi(embed_v0)]
    impl SimpleStorage of super::ISimpleStorage<ContractState> {
        fn set(ref self: ContractState, x: u128) {
            self.stored_data.write(x);
        }

        fn get(self: @ContractState) -> u128 {
            self.stored_data.read()
        }
    }
```

{{#label implementation}} <span class="caption">清单 {{#ref implementation}}: `SimpleStorage` 实现</span>

由于合约接口被定义为 `ISimpleStorage` trait，为了匹配接口，合约的公共函数必须在此 trait 的实现中定义 —— 这允许我们确保合约的实现与其接口匹配。

然而，仅仅在实现块中定义函数是不够的。实现块必须用 `#[abi(embed_v0)]` 属性注释。此属性将此实现中定义的函数公开给外部世界 —— 忘记添加它，你的函数将无法从外部调用。所有在标记为 `#[abi(embed_v0)]` 的块中定义的函数因此都是 _公共函数_。

因为 `SimpleStorage` 合约被定义为一个模块，我们需要访问父模块中定义的接口。我们可以使用 `use` 关键字将其带入当前作用域，或者直接使用 `super` 引用它。

在编写接口的实现时，trait 方法中的 `self` 参数 **必须** 是 `ContractState` 类型。`ContractState` 类型由编译器生成，并提供对 `Storage` 结构体中定义的存储变量的访问。此外，`ContractState` 赋予了我们发出事件的能力。`ContractState` 这个名字并不奇怪，因为它是合约状态的表示，这就是我们在合约接口 trait 中认为的 `self`。当 `self` 是 `ContractState` 的快照时，只允许读取访问，并且不可能发出事件。

### 访问和修改合约状态

通常使用两种方法来访问或修改合约的状态：

- `read`，它返回存储变量的值。此方法在变量本身上调用，不带任何参数。

```cairo,noplayground
            self.stored_data.read()
```

- `write`，它允许在存储槽中写入新值。此方法也在变量本身上调用，并接受一个参数，即要写入的值。注意，`write` 可能接受多个参数，具体取决于存储变量的类型。例如，在映射 (mapping) 上写入需要 2 个参数：键和要写入的值。

```cairo,noplayground
            self.stored_data.write(x);
```

> 提醒：如果合约状态作为快照用 `@` 传递，而不是用 `ref` 引用传递，尝试修改合约状态将导致编译错误。

这个合约除了允许任何人存储一个世界上任何人都可以访问的单个数字外，没有做太多事情。任何人都可以用不同的值再次调用 `set` 并覆盖当前的数字。尽管如此，存储在合约存储中的每个值仍将存储在区块链的历史记录中。在本书的后面，你将看到如何施加访问限制，以便只有你可以更改数字。
