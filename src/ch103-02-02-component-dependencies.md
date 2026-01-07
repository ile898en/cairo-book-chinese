# 组件依赖

当我们尝试在一个组件内部使用另一个组件时，使用组件变得更加复杂。如前所述，组件只能嵌入在合约中，这意味着不可能在另一个组件中嵌入一个组件。但是，这并不意味着我们不能在一个组件内部使用另一个组件。在本节中，我们将看到如何将一个组件用作另一个组件的依赖项。

考虑一个名为 `OwnableCounter` 的组件，其目的是创建一个只能由所有者增减的计数器。此组件可以嵌入在任何合约中，以便任何使用它的合约都将拥有一个只能由其所有者增减的计数器。

实现此目的的第一种方法是创建一个单独的组件，该组件在单个组件内包含计数器和所有权功能。然而，不建议这样做：我们的目标是最大程度地减少代码重复量并利用组件的可重用性。相反，我们可以创建一个新组件，它 _依赖_ 于 `Ownable` 组件来获取所有权功能，并在内部定义计数器的逻辑。

清单 {{#ref ownable_component}} 显示了完整的实现，我们将紧接着对其进行分解：

```cairo,noplayground
//ANCHOR: interface

#[starknet::interface]
trait IOwnableCounter<TContractState> {
    fn get_counter(self: @TContractState) -> u32;
    fn increment(ref self: TContractState);
}
//ANCHOR_END: interface

//ANCHOR: component
#[starknet::component]
pub mod OwnableCounterComponent {
    use listing_03_component_dep::owner::OwnableComponent;
    use listing_03_component_dep::owner::OwnableComponent::InternalImpl;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    pub struct Storage {
        value: u32,
    }

    //ANCHOR: component_impl
    #[embeddable_as(OwnableCounterImpl)]
    //ANCHOR: component_signature
    impl OwnableCounter<
        TContractState,
        +HasComponent<TContractState>,
        +Drop<TContractState>,
        impl Owner: OwnableComponent::HasComponent<TContractState>,
    > of super::IOwnableCounter<ComponentState<TContractState>> {
        //ANCHOR_END: component_signature
        //ANCHOR: get_counter
        fn get_counter(self: @ComponentState<TContractState>) -> u32 {
            self.value.read()
        }
        //ANCHOR_END: get_counter

        //ANCHOR: increment
        fn increment(ref self: ComponentState<TContractState>) {
            let ownable_comp = get_dep_component!(@self, Owner);
            ownable_comp.assert_only_owner();
            self.value.write(self.value.read() + 1);
        }
        //ANCHOR_END: increment
    }
    //ANCHOR_END: component_impl
}
//ANCHOR_END: component
```

{{#label ownable_component}} <span class="caption">清单 {{#ref ownable_component}}: 一个 OwnableCounter 组件</span>

## 特性

### 指定对另一个组件的依赖

```cairo,noplayground
    impl OwnableCounter<
        TContractState,
        +HasComponent<TContractState>,
        +Drop<TContractState>,
        impl Owner: OwnableComponent::HasComponent<TContractState>,
    > of super::IOwnableCounter<ComponentState<TContractState>> {
```

在 [第 8 章][cairo traits] 中，我们介绍了 trait 约束 (trait bounds)，用于指定泛型类型必须实现某个 trait。同样，我们可以通过限制 `impl` 块仅适用于包含所需组件的合约来指定一个组件依赖于另一个组件。在我们的例子中，这是通过添加限制 `impl Owner: ownable_component::HasComponent<TContractState>` 来完成的，这表明此 `impl` 块仅适用于包含 `ownable_component::HasComponent` trait 实现的合约。这本质上意味着 `TContractState` 类型可以访问 ownable 组件。有关更多信息，请参阅 [组件底层原理][component impl]。

尽管大多数 trait 约束是使用 [匿名参数][anonymous generic impl operator] 定义的，但对 `Ownable` 组件的依赖是使用命名参数（这里是 `Owner`）定义的。当在 `impl` 块内访问 `Ownable` 组件时，我们将需要使用此显式名称。

虽然这种机制很冗长，一开始可能不容易上手，但它是 Cairo 中 trait 系统的强大杠杆。这种机制的内部工作原理对用户来说是抽象的，你只需要知道，当你在合约中嵌入组件时，同一合约中的所有其他组件都可以访问它。

[cairo traits]: ./ch08-02-traits-in-cairo.md
[component impl]: ch103-02-01-under-the-hood.md#inside-components-generic-impls
[anonymous generic impl operator]:
  ./ch08-01-generic-data-types.md#anonymous-generic-implementation-parameter--operator

### 使用依赖项

既然我们已经让我们的 `impl` 依赖于 `Ownable` 组件，我们就可以在实现块内访问它的函数、存储和事件。要将 `Ownable` 组件带入作用域，我们有两个选择，这取决于我们是否打算改变 `Ownable` 组件的状态。如果我们想访问 `Ownable` 组件的状态而不改变它，我们使用 `get_dep_component!` 宏。如果我们想改变 `Ownable` 组件的状态（例如，更改当前所有者），我们使用 `get_dep_component_mut!` 宏。这两个宏都接受两个参数：第一个是 `self`，根据可变性作为快照或引用，表示使用依赖项的组件的状态，第二个是要访问的组件。

```cairo,noplayground
        fn increment(ref self: ComponentState<TContractState>) {
            let ownable_comp = get_dep_component!(@self, Owner);
            ownable_comp.assert_only_owner();
            self.value.write(self.value.read() + 1);
        }
```

在这个函数中，我们要确保只有所有者才能调用 `increment` 函数。我们需要使用 `Ownable` 组件中的 `assert_only_owner` 函数。我们将使用 `get_dep_component!` 宏，它将返回请求的组件状态的快照，并在其上调用 `assert_only_owner`，作为该组件的方法。

对于 `transfer_ownership` 函数，我们要改变该状态以更改当前所有者。因为我们知道嵌入此组件的合约实现了 `OwnableComponent` trait，所以不需要在 `OwnableCounterComponent` 组件中定义此函数：宿主合约将通过 `OwnableComponent` 公开它。

最终的宿主合约如清单 {{#ref contract}} 所示。

```cairo,noplayground
#[starknet::contract]
mod OwnableCounter {
    use listing_03_component_dep::counter::OwnableCounterComponent;
    use listing_03_component_dep::owner::OwnableComponent;
    use starknet::ContractAddress;

    component!(path: OwnableCounterComponent, storage: counter, event: CounterEvent);
    component!(path: OwnableComponent, storage: ownable, event: OwnableEvent);

    //ANCHOR:  embedded_impl
    #[abi(embed_v0)]
    impl OwnableCounterImpl =
        OwnableCounterComponent::OwnableCounterImpl<ContractState>;
    //ANCHOR_END: embedded_impl

    #[abi(embed_v0)]
    impl OwnableImpl = OwnableComponent::OwnableImpl<ContractState>;
    impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        counter: OwnableCounterComponent::Storage,
        #[substorage(v0)]
        ownable: OwnableComponent::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        CounterEvent: OwnableCounterComponent::Event,
        OwnableEvent: OwnableComponent::Event,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.ownable.initializer(owner);
    }
}
```

{{#label contract}} <span class="caption">清单 {{#ref contract}}: 宿主合约</span>
