# 组件 (Components)：智能合约的乐高积木

开发共享通用逻辑和存储的合约可能会很痛苦且容易出错，因为这些逻辑很难重用，并且需要在每个合约中重新实现。但是，如果有办法只将你需要的额外功能插入到你的合约中，将你合约的核心逻辑与其余部分分开呢？

组件正是提供了这一点。它们是封装了可重用的逻辑、存储和事件的模块化附加组件，可以合并到多个合约中。它们可用于扩展合约的功能，而不必一遍又一遍地重新实现相同的逻辑。

将组件视为乐高积木。它们允许你通过插入你或其他人编写的模块来丰富你的合约。这个模块可以是一个简单的模块，比如所有权组件，也可以是更复杂的模块，比如一个完整的 ERC20 代币。

组件是一个单独的模块，可以包含存储、事件和函数。与合约不同，组件不能被声明或部署。它的逻辑最终将成为嵌入它的合约字节码的一部分。

## 组件里有什么？

组件非常类似于合约。它可以包含：

- 存储变量
- 事件
- 外部和内部函数

与合约不同，组件不能单独部署。组件的代码成为它嵌入的合约的一部分。

## 创建组件

要创建组件，首先在用 `#[starknet::component]` 属性装饰的自己的模块中定义它。在这个模块中，你可以声明 `Storage` 结构体和 `Event` 枚举，就像通常在 [合约][contract anatomy] 中所做的那样。

下一步是定义组件接口，包含允许外部访问组件逻辑的函数签名。你可以通过声明带有 `#[starknet::interface]` 属性的 Trait 来定义组件的接口，就像对合约所做的那样。此接口将用于使用 [分发器][contract dispatcher] 模式启用对组件函数的外部访问。

组件外部逻辑的实际实现是在标记为 `#[embeddable_as(name)]` 的 `impl` 块中完成的。通常，这个 `impl` 块将是定义组件接口的 trait 的实现。

> 注意：`name` 是我们在合约中用来引用组件的名称。它不同于你的 impl 的名称。

你也可以定义不能从外部访问的内部函数，只需省略内部 `impl` 块上方的 `#[embeddable_as(name)]` 属性。你将能够在嵌入组件的合约内部使用这些内部函数，但不能从外部与它交互，因为它们不是合约 ABI 的一部分。

这些 `impl` 块中的函数期望像 `ref self: ComponentState<TContractState>`（对于状态修改函数）或 `self: @ComponentState<TContractState>`（对于视图函数）这样的参数。这使得 impl 对 `TContractState` 是通用的，允许我们在任何合约中使用此组件。

[contract anatomy]: ./ch100-00-introduction-to-smart-contracts.md#
[contract dispatcher]: ./ch102-02-interacting-with-another-contract.md

### 示例：一个 Ownable 组件

> ⚠️ 下面显示的示例未经审计，不适用于生产用途。作者不对使用此代码造成的任何损害负责。

Ownable 组件的接口，定义了用于管理合约所有权的外部可用方法，如下所示：

```cairo,noplayground
#[starknet::interface]
trait IOwnable<TContractState> {
    //ANCHOR: trait_def
    fn owner(self: @TContractState) -> ContractAddress;
    fn transfer_ownership(ref self: TContractState, new_owner: ContractAddress);
    fn renounce_ownership(ref self: TContractState);
    //ANCHOR_END: trait_def
}
```

组件本身定义为：

```cairo,noplayground
#[starknet::component]
pub mod OwnableComponent {
    use core::num::traits::Zero;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use starknet::{ContractAddress, get_caller_address};
    use super::Errors;

    #[storage]
    pub struct Storage {
        owner: ContractAddress,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        OwnershipTransferred: OwnershipTransferred,
    }

    #[derive(Drop, starknet::Event)]
    struct OwnershipTransferred {
        previous_owner: ContractAddress,
        new_owner: ContractAddress,
    }

    //ANCHOR: impl_signature
    #[embeddable_as(OwnableImpl)]
    impl Ownable<
        TContractState, +HasComponent<TContractState>,
    > of super::IOwnable<ComponentState<TContractState>> {
        //ANCHOR_END: impl_signature
        fn owner(self: @ComponentState<TContractState>) -> ContractAddress {
            self.owner.read()
        }

        fn transfer_ownership(
            ref self: ComponentState<TContractState>, new_owner: ContractAddress,
        ) {
            assert(!new_owner.is_zero(), Errors::ZERO_ADDRESS_OWNER);
            self.assert_only_owner();
            self._transfer_ownership(new_owner);
        }

        fn renounce_ownership(ref self: ComponentState<TContractState>) {
            self.assert_only_owner();
            self._transfer_ownership(Zero::zero());
        }
    }

    #[generate_trait]
    pub impl InternalImpl<
        TContractState, +HasComponent<TContractState>,
    > of InternalTrait<TContractState> {
        fn initializer(ref self: ComponentState<TContractState>, owner: ContractAddress) {
            self._transfer_ownership(owner);
        }

        fn assert_only_owner(self: @ComponentState<TContractState>) {
            let owner: ContractAddress = self.owner.read();
            let caller: ContractAddress = get_caller_address();
            assert(!caller.is_zero(), Errors::ZERO_ADDRESS_CALLER);
            assert(caller == owner, Errors::NOT_OWNER);
        }

        fn _transfer_ownership(
            ref self: ComponentState<TContractState>, new_owner: ContractAddress,
        ) {
            let previous_owner: ContractAddress = self.owner.read();
            self.owner.write(new_owner);
            self
                .emit(
                    OwnershipTransferred { previous_owner: previous_owner, new_owner: new_owner },
                );
        }
    }
}
```

这种语法实际上与用于合约的语法非常相似。唯一的区别在于 impl 上方的 `#[embeddable_as]` 属性以及我们将详细剖析的 impl 块的泛型性。

如你所见，我们的组件有两个 `impl` 块：一个对应于接口 trait 的实现，另一个包含不应在外部公开且仅供内部使用的方法。作为接口的一部分公开 `assert_only_owner` 是没有意义的，因为它只意味着由嵌入组件的合约在内部使用。

## 仔细看看 `impl` 块

```cairo,noplayground
    #[embeddable_as(OwnableImpl)]
    impl Ownable<
        TContractState, +HasComponent<TContractState>,
    > of super::IOwnable<ComponentState<TContractState>> {
```

`#[embeddable_as]` 属性用于将 impl 标记为可嵌入合约内部。它允许我们指定在合约中用于引用此组件的 impl 的名称。在这种情况下，组件将在嵌入它的合约中被称为 `OwnableImpl`。

实现本身对 `ComponentState<TContractState>` 是通用的，并增加了 `TContractState` 必须实现 `HasComponent<T>` trait 的限制。这允许我们在任何合约中使用该组件，只要该合约实现了 `HasComponent` trait。使用组件不需要详细了解此机制，但如果你对内部工作原理感到好奇，可以在 [“组件底层原理”][components inner working] 部分阅读更多内容。

与常规智能合约的主要区别之一是，对存储和事件的访问是通过泛型 `ComponentState<TContractState>` 类型而不是 `ContractState` 完成的。注意，虽然类型不同，但访问存储或发出事件的方式类似，都是通过 `self.storage_var_name.read()` 或 `self.emit(...)`。

> 注意：为避免混淆，请遵循 OpenZeppelin 的模式：在可嵌入名称和合约的 impl 别名中保留 `Impl` 后缀（例如，`OwnableImpl`），而本地组件 impl 以 trait 命名，不带后缀（例如，`impl Ownable<...>`）。

[components inner working]: ./ch103-02-01-under-the-hood.md

## 将合约迁移到组件

由于合约和组件都有很多相似之处，因此从合约迁移到组件实际上非常容易。唯一需要的更改是：

- 向模块添加 `#[starknet::component]` 属性。
- 向将嵌入另一个合约的 `impl` 块添加 `#[embeddable_as(name)]` 属性。
- 向 `impl` 块添加泛型参数：
  - 添加 `TContractState` 作为泛型参数。
  - 添加 `+HasComponent<TContractState>` 作为 impl 限制。
- 将 `impl` 块内部函数中的 `self` 参数的类型更改为 `ComponentState<TContractState>` 而不是 `ContractState`。

对于没有显式定义并使用 `#[generate_trait]` 生成的 trait，逻辑是相同的 —— 但 trait 对 `TContractState` 是通用的，而不是 `ComponentState<TContractState>`，如 `InternalTrait` 的示例所示。

## 在合约内部使用组件

组件的主要优势在于它允许在你的合约中重用已构建的原语，且样板代码量受到限制。要将组件集成到你的合约中，你需要：

1. 使用 `component!()` 宏声明它，指定
   1. 组件的路径 `path::to::component`。
   2. 你的合约存储中引用此组件存储的变量名称（例如 `ownable`）。
   3. 你的合约事件枚举中引用此组件事件的变体名称（例如 `OwnableEvent`）。

2. 将组件的存储和事件的路径添加到合约的 `Storage` 和 `Event`。它们必须匹配步骤 1 中提供的名称（例如 `ownable: ownable_component::Storage` 和 `OwnableEvent: ownable_component::Event`）。

   存储变量 **必须** 用 `#[substorage(v0)]` 属性注释。

3. 嵌入定义在你的合约内的组件逻辑，通过使用 impl 别名用具体的 `ContractState` 实例化组件的泛型 impl。此别名必须用 `#[abi(embed_v0)]` 注释以在外部公开组件的函数。

   如你所见，InternalImpl 没有标记为 `#[abi(embed_v0)]`。确实，我们不想在外部公开在此 impl 中定义的函数。但是，我们可能仍然希望在内部访问它们。

例如，要嵌入上面定义的 `Ownable` 组件，我们会做以下操作：

```cairo,noplayground
#[starknet::contract]
mod OwnableCounter {
    use listing_01_ownable::component::OwnableComponent;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    component!(path: OwnableComponent, storage: ownable, event: OwnableEvent);

    //ANCHOR:  embedded_impl
    #[abi(embed_v0)]
    impl OwnableImpl = OwnableComponent::OwnableImpl<ContractState>;

    impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;
    //ANCHOR_END: embedded_impl

    #[storage]
    struct Storage {
        counter: u128,
        #[substorage(v0)]
        ownable: OwnableComponent::Storage,
    }


    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        OwnableEvent: OwnableComponent::Event,
    }


    #[abi(embed_v0)]
    fn foo(ref self: ContractState) {
        self.ownable.assert_only_owner();
        self.counter.write(self.counter.read() + 1);
    }
}
```

组件的逻辑现在无缝地成为了合约的一部分！我们可以通过使用以合约地址实例化的 `IOwnableDispatcher` 来在外部调用组件函数。

```cairo
#[starknet::interface]
trait IOwnable<TContractState> {
    //ANCHOR: trait_def
    fn owner(self: @TContractState) -> ContractAddress;
    fn transfer_ownership(ref self: TContractState, new_owner: ContractAddress);
    fn renounce_ownership(ref self: TContractState);
    //ANCHOR_END: trait_def
}
```

## 堆叠组件以实现最大的可组合性

当将多个组件组合在一起时，组件的可组合性才真正闪耀。每个组件都将其功能添加到合约中。你可以依靠 [Openzeppelin 的组件实现][OpenZeppelin Cairo Contracts] 来快速插入你需要合约拥有的所有常用功能。

开发人员可以专注于他们的核心合约逻辑，同时依靠经过实战测试和审计的组件来处理其他所有事情。

组件甚至可以 [依赖][component dependencies] 其他组件，通过限制它们通用的 `TContractstate` 来实现另一个组件的 trait。在我们深入研究这种机制之前，让我们先看看 [组件在底层是如何工作的][components inner working]。

[OpenZeppelin Cairo Contracts]: https://github.com/OpenZeppelin/cairo-contracts
[component dependencies]: ./ch103-02-02-component-dependencies.md
[components inner working]: ./ch103-02-01-under-the-hood.md
