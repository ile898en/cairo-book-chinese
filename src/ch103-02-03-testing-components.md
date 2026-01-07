# 测试组件

测试组件与测试合约略有不同。合约需要针对特定状态进行测试，这可以通过在测试中部署合约，或者通过简单地获取 `ContractState` 对象并在测试上下文中修改它来实现。

组件是一个通用的结构，旨在集成到合约中，不能单独部署，也没有我们可以使用的 `ContractState` 对象。那么我们如何测试它们呢？

让我们考虑我们要测试一个非常简单的名为 "Counter" 的组件，它允许每个合约拥有一个可以递增的计数器。组件定义在清单 {{#ref test_component}} 中：

```cairo, noplayground
#[starknet::component]
pub mod CounterComponent {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    pub struct Storage {
        value: u32,
    }

    #[embeddable_as(CounterImpl)]
    pub impl Counter<
        TContractState, +HasComponent<TContractState>,
    > of super::ICounter<ComponentState<TContractState>> {
        //ANCHOR: get_counter
        fn get_counter(self: @ComponentState<TContractState>) -> u32 {
            self.value.read()
        }
        //ANCHOR_END: get_counter

        //ANCHOR: increment
        fn increment(ref self: ComponentState<TContractState>) {
            self.value.write(self.value.read() + 1);
        }
        //ANCHOR_END: increment
    }
}
```

{{#label test_component}} <span class="caption">清单 {{#ref test_component}}: 一个简单的 Counter 组件</span>

## 通过部署模拟合约测试组件

测试组件最简单的方法是将它集成到一个模拟合约中。这个模拟合约仅用于测试目的，并且仅集成你要测试的组件。这允许你在合约的上下文中测试组件，并使用分发器调用组件的入口点。

我们可以如下定义这样的模拟合约：

```cairo, noplayground
#[starknet::contract]
mod MockContract {
    use super::counter::CounterComponent;

    component!(path: CounterComponent, storage: counter, event: CounterEvent);

    #[storage]
    struct Storage {
        #[substorage(v0)]
        counter: CounterComponent::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        CounterEvent: CounterComponent::Event,
    }

    #[abi(embed_v0)]
    impl CounterImpl = CounterComponent::CounterImpl<ContractState>;
}
```

该合约完全致力于测试 `Counter` 组件。它使用 `component!` 宏嵌入组件，通过用 `#[abi(embed_v0)]` 注释 impl 别名来公开组件的入口点。

我们还需要定义一个接口，以便在外部与此模拟合约进行交互。

```cairo, noplayground
#[starknet::interface]
pub trait ICounter<TContractState> {
    fn get_counter(self: @TContractState) -> u32;
    fn increment(ref self: TContractState);
}
```

我们现在可以通过部署此模拟合约并调用其入口点来为组件编写测试，就像我们对典型合约所做的那样。

```cairo, noplayground
use starknet::SyscallResultTrait;
use starknet::syscalls::deploy_syscall;
use super::MockContract;
use super::counter::{ICounterDispatcher, ICounterDispatcherTrait};

fn setup_counter() -> ICounterDispatcher {
    let (address, _) = deploy_syscall(
        MockContract::TEST_CLASS_HASH.try_into().unwrap(), 0, array![].span(), false,
    )
        .unwrap_syscall();
    ICounterDispatcher { contract_address: address }
}

#[test]
fn test_constructor() {
    let counter = setup_counter();
    assert_eq!(counter.get_counter(), 0);
}

#[test]
fn test_increment() {
    let counter = setup_counter();
    counter.increment();
    assert_eq!(counter.get_counter(), 1);
}
```

## 在不部署合约的情况下测试组件

在 [组件底层原理][components inner working] 中，我们看到组件利用泛型来定义可以嵌入多个合约的存储和逻辑。如果一个合约嵌入了一个组件，该合约中就会创建一个 `HasComponent` trait，并且组件方法变得可用。

这告诉我们，如果我们能为 `ComponentState` 结构体提供一个实现了 `HasComponent` trait 的具体 `TContractState`，就应该能够使用这个具体的 `ComponentState` 对象直接调用组件的方法，而不必部署模拟合约。

让我们看看如何使用类型别名来做到这一点。我们仍然需要定义一个模拟合约 —— 让我们使用与上面相同的合约 —— 但这次我们不需要部署它。

首先，我们需要使用类型别名定义泛型 `ComponentState` 类型的具体实现。我们将使用 `MockContract::ContractState` 类型来做到这一点。

```cairo, noplayground
type TestingState = CounterComponent::ComponentState<MockContract::ContractState>;

// You can derive even `Default` on this type alias
impl TestingStateDefault of Default<TestingState> {
    fn default() -> TestingState {
        CounterComponent::component_state_for_testing()
    }
}
```

我们将 `TestingState` 类型定义为 `CounterComponent::ComponentState<MockContract::ContractState>` 类型的别名。通过传递 `MockContract::ContractState` 类型作为 `ComponentState` 的具体类型，我们将 `ComponentState` 结构体的具体实现别名为 `TestingState`。

因为 `MockContract` 嵌入了 `CounterComponent`，在 `Counter` impl 块中定义的 `CounterComponent` 方法（通过 `CounterImpl` 可嵌入别名公开）现在可以在 `TestingState` 对象上使用。

既然我们已经使这些方法可用，我们需要实例化一个 `TestingState` 类型的对象，我们将用它来测试组件。我们可以通过调用 `component_state_for_testing` 函数来做到这一点，该函数会自动推断它应该返回一个 `TestingState` 类型的对象。

我们甚至可以将其作为 `Default` trait 的一部分来实现，这允许我们使用 `Default::default()` 语法返回一个空的 `TestingState`。

让我们总结一下到目前为止我们所做的工作：

- 我们定义了一个模拟合约，它嵌入了我们要测试的组件。
- 我们使用类型别名和 `MockContract::ContractState` 定义了 `ComponentState<TContractState>` 的具体实现，我们将其命名为 `TestingState`。
- 我们定义了一个函数，使用 `component_state_for_testing` 返回 `TestingState` 对象。

我们现在可以通过直接调用其函数来为组件编写测试，而不必部署模拟合约。这种方法比前一种方法更轻量级，并且它允许测试通常不向外界公开的组件内部函数。

```cairo, noplayground
#[test]
fn test_increment() {
    let mut counter: TestingState = Default::default();

    counter.increment();
    counter.increment();

    assert_eq!(counter.get_counter(), 2);
}
```

[components inner working]: ./ch103-02-01-under-the-hood.md
