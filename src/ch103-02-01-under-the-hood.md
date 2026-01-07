# 组件：底层原理

组件为 Starknet 合约提供了强大的模块化。但这一切背后的魔法究竟是如何发生的呢？

本章将深入研究编译器内部机制，解释实现组件可组合性的机制。

## 可嵌入 Impl 入门

在深入研究组件之前，我们需要了解 _可嵌入 impl (embeddable impls)_。

Starknet 接口 trait（标记为 `#[starknet::interface]`）的 impl 可以变为可嵌入的。可嵌入 impl 可以注入到任何合约中，添加新的入口点并修改合约的 ABI。

让我们看一个例子来观察它的实际效果：

```cairo,noplayground
#[starknet::interface]
trait SimpleTrait<TContractState> {
    fn ret_4(self: @TContractState) -> u8;
}

#[starknet::embeddable]
impl SimpleImpl<TContractState> of SimpleTrait<TContractState> {
    fn ret_4(self: @TContractState) -> u8 {
        4
    }
}

#[starknet::contract]
mod simple_contract {
    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl MySimpleImpl = super::SimpleImpl<ContractState>;
}
```

通过嵌入 `SimpleImpl`，我们在合约的 ABI 中外部公开了 `ret4`。

现在我们对嵌入机制有了更深入的了解，现在我们可以看看组件是如何在此基础上构建的。

## 组件内部：泛型 Impl

回想一下组件中使用的 impl 块语法：

```cairo,noplayground
    #[embeddable_as(OwnableImpl)]
    impl Ownable<
        TContractState, +HasComponent<TContractState>,
    > of super::IOwnable<ComponentState<TContractState>> {
```

关键点：

- 泛型 impl `Ownable` 需要底层合约实现 `HasComponent<TContractState>` trait，这在使用 `component!()` 宏在合约内使用组件时会自动生成。

  编译器将生成一个可嵌入的 impl，它包装 `Ownable` 中的任何函数，将 `self: ComponentState<TContractState>` 参数替换为 `self: TContractState`，其中对组件状态的访问是通过 `HasComponent<TContractState>` trait 中的 `get_component` 函数进行的。

  对于每个组件，编译器生成一个 `HasComponent` trait。此 trait 定义了在通用合约的实际 `TContractState` 和 `ComponentState<TContractState>` 之间建立桥梁的接口。

  ```cairo,noplayground
  // 每个组件生成
  trait HasComponent<TContractState> {
      fn get_component(self: @TContractState) -> @ComponentState<TContractState>;
      fn get_component_mut(ref self: TContractState) -> ComponentState<TContractState>;
      fn get_contract(self: @ComponentState<TContractState>) -> @TContractState;
      fn get_contract_mut(ref self: ComponentState<TContractState>) -> TContractState;
      fn emit<S, impl IntoImp: traits::Into<S, Event>>(ref self: ComponentState<TContractState>, event: S);
  }
  ```

  在我们的上下文中，`ComponentState<TContractState>` 是特定于 ownable 组件的类型，即它具有基于 `ownable_component::Storage` 中定义的存储变量的成员。从通用的 `TContractState` 转移到 `ComponentState<TContractState>` 将允许我们将 `OwnableImpl` 嵌入到任何想要使用它的合约中。相反的方向（`ComponentState<TContractState>` 到 `ContractState`）对于依赖项很有用（参见 [组件依赖](./ch103-02-02-component-dependencies.md) 部分中的 `Upgradeable` 组件依赖 `IOwnable` 实现的示例）。

  简而言之，可以将上述 `HasComponent<T>` 的实现理解为：**“其状态 T 具有 upgradeable 组件的合约”。**

- `Ownable` 用 `embeddable_as(<name>)` 属性注释：

  `embeddable_as` 类似于 `embeddable`；它仅适用于 `starknet::interface` trait 的 impl，并允许将此 impl 嵌入到合约模块中。话虽如此，`embeddable_as(<name>)` 在组件的上下文中具有另一个作用。最终，当将 `OwnableImpl` 嵌入到某个合约中时，我们期望得到一个具有以下函数的 impl：

  ```cairo,noplayground
    fn owner(self: @TContractState) -> ContractAddress;
    fn transfer_ownership(ref self: TContractState, new_owner: ContractAddress);
    fn renounce_ownership(ref self: TContractState);
  ```

  注意，虽然从接收泛型类型 `ComponentState<TContractState>` 的函数开始，但我们希望最终得到一个接收 `ContractState` 的函数。这就是 `embeddable_as(<name>)` 出现的地方。为了看到全貌，我们需要看看编译器由于 `embeddable_as(OwnableImpl)` 注释而生成的 impl 是什么样的：

```cairo,noplayground
//TAG: does_not_compile
#[starknet::embeddable]
impl OwnableImpl<
    TContractState, +HasComponent<TContractState>, impl TContractStateDrop: Drop<TContractState>,
> of super::IOwnable<TContractState> {
    fn owner(self: @TContractState) -> ContractAddress {
        let component = HasComponent::get_component(self);
        Ownable::owner(component)
    }

    fn transfer_ownership(ref self: TContractState, new_owner: ContractAddress) {
        let mut component = HasComponent::get_component_mut(ref self);
        Ownable::transfer_ownership(ref component, new_owner)
    }

    fn renounce_ownership(ref self: TContractState) {
        let mut component = HasComponent::get_component_mut(ref self);
        Ownable::renounce_ownership(ref component)
    }
}
```

注意，多亏有了 `HasComponent<TContractState>` 的 impl，编译器能够将我们的函数包装在一个新的 impl 中，该 impl 不直接了解 `ComponentState` 类型。`OwnableImpl`（我们在编写 `embeddable_as(OwnableImpl)` 时选择的名称）是我们将嵌入到想要所有权的合约中的 impl。

## 合约集成

我们已经看到了泛型 impl 如何实现组件的可重用性。接下来让我们看看合约如何集成组件。

合约使用 **impl 别名** 用合约的具体 `ContractState` 实例化组件的泛型 impl。

```cairo,noplayground
    #[abi(embed_v0)]
    impl OwnableImpl = OwnableComponent::OwnableImpl<ContractState>;

    impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;
```

上述行使用 Cairo impl 嵌入机制和 impl 别名语法。我们要用具体类型 `ContractState` 实例化可嵌入的生成 impl `OwnableImpl<TContractState>`。回想一下，泛型 impl 是 `Ownable<TContractState>`，而 `component!` 宏提供了 `HasComponent<TContractState>` 以便包装器可以委托给它。

注意，只有使用合约才能实现此 trait，因为只有它知道合约状态和组件状态。

这将所有内容粘合在一起，将组件逻辑注入到合约中。

## 关键要点

- 可嵌入 impl 允许通过添加入口点和修改合约 ABI 将组件逻辑注入到合约中。
- 当在合约中使用组件时，编译器会自动生成 `HasComponent` trait 实现。这在合约状态和组件状态之间建立了一座桥梁，使两者之间能够进行交互。
- 组件以通用的、与合约无关的方式封装可重用的逻辑。合约通过 impl 别名集成组件，并通过生成的 `HasComponent` trait 访问它们。
- 组件建立在可嵌入 impl 之上，通过定义可以集成到任何想要使用该组件的合约中的通用组件逻辑。Impl 别名使用合约的具体存储类型实例化这些通用 impl。
