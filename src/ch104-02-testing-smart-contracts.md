# 测试智能合约

测试智能合约是开发过程中至关重要的一部分。确保智能合约按预期运行并且安全非常重要。

在 Cairo Book 的前一部分中，我们学习了如何为 Cairo 程序编写和构建测试。我们演示了如何使用 `scarb` 命令行工具运行这些测试。虽然这种方法对于测试独立的 Cairo 程序和函数很有用，但它缺乏测试需要控制合约状态和执行上下文的智能合约的功能。因此，在本节中，我们将介绍如何使用 Starknet Foundry（Starknet 的智能合约开发工具链）来测试你的 Cairo 合约。

在本章中，我们将使用清单 {{#ref pizza-factory}} 中的 `PizzaFactory` 合约为作为示例，演示如何使用 Starknet Foundry 编写测试。

```cairo,noplayground
use starknet::ContractAddress;

#[starknet::interface]
pub trait IPizzaFactory<TContractState> {
    fn increase_pepperoni(ref self: TContractState, amount: u32);
    fn increase_pineapple(ref self: TContractState, amount: u32);
    fn get_owner(self: @TContractState) -> ContractAddress;
    fn change_owner(ref self: TContractState, new_owner: ContractAddress);
    fn make_pizza(ref self: TContractState);
    fn count_pizza(self: @TContractState) -> u32;
}

#[starknet::contract]
pub mod PizzaFactory {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use starknet::{ContractAddress, get_caller_address};
    use super::IPizzaFactory;

    #[storage]
    pub struct Storage {
        pepperoni: u32,
        pineapple: u32,
        pub owner: ContractAddress,
        pizzas: u32,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.pepperoni.write(10);
        self.pineapple.write(10);
        self.owner.write(owner);
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        PizzaEmission: PizzaEmission,
    }

    #[derive(Drop, starknet::Event)]
    pub struct PizzaEmission {
        pub counter: u32,
    }

    #[abi(embed_v0)]
    impl PizzaFactoryimpl of super::IPizzaFactory<ContractState> {
        fn increase_pepperoni(ref self: ContractState, amount: u32) {
            assert!(amount != 0, "Amount cannot be 0");
            self.pepperoni.write(self.pepperoni.read() + amount);
        }

        fn increase_pineapple(ref self: ContractState, amount: u32) {
            assert!(amount != 0, "Amount cannot be 0");
            self.pineapple.write(self.pineapple.read() + amount);
        }

        fn make_pizza(ref self: ContractState) {
            assert!(self.pepperoni.read() > 0, "Not enough pepperoni");
            assert!(self.pineapple.read() > 0, "Not enough pineapple");

            let caller: ContractAddress = get_caller_address();
            let owner: ContractAddress = self.get_owner();

            assert!(caller == owner, "Only the owner can make pizza");

            self.pepperoni.write(self.pepperoni.read() - 1);
            self.pineapple.write(self.pineapple.read() - 1);
            self.pizzas.write(self.pizzas.read() + 1);

            self.emit(PizzaEmission { counter: self.pizzas.read() });
        }

        fn get_owner(self: @ContractState) -> ContractAddress {
            self.owner.read()
        }

        fn change_owner(ref self: ContractState, new_owner: ContractAddress) {
            self.set_owner(new_owner);
        }

        fn count_pizza(self: @ContractState) -> u32 {
            self.pizzas.read()
        }
    }

    #[generate_trait]
    pub impl InternalImpl of InternalTrait {
        fn set_owner(ref self: ContractState, new_owner: ContractAddress) {
            let caller: ContractAddress = get_caller_address();
            assert!(caller == self.get_owner(), "Only the owner can set ownership");

            self.owner.write(new_owner);
        }
    }
}
```

{{#label pizza-factory}} <span class="caption">清单 {{#ref pizza-factory}}: 一个需要测试的披萨工厂</span>

## 使用 Starknet Foundry 配置你的 Scarb 项目

你的 Scarb 项目的设置可以在 `Scarb.toml` 文件中配置。要使用 Starknet Foundry 作为你的测试工具，你需要将其添加为 `Scarb.toml` 文件中的开发依赖项。在撰写本文时，Starknet Foundry 的最新版本是 `v0.39.0` —— 但你应该使用最新版本。

```toml,noplayground
[dev-dependencies]
snforge_std = "0.51.1"

[scripts]
test = "snforge test"

[tool.scarb]
allow-prebuilt-plugins = ["snforge_std"]
```

默认情况下，`scarb test` 命令配置为执行 `scarb cairo-test`。在我们的设置中，我们将其配置为执行 `snforge test`。这将允许我们在运行 `scarb test` 命令时使用 Starknet Foundry 运行我们的测试。

一旦你的项目配置完成，你需要按照 [Starknet Foundry 文档](https://foundry-rs.github.io/starknet-foundry/getting-started/installation.html) 中的安装指南安装 Starknet Foundry。像往常一样，我们建议使用 `asdf` 来管理你的开发工具版本。

## 使用 Starknet Foundry 测试智能合约

使用 Starknet Foundry 运行测试的常用命令是 `snforge test`。但是，当我们配置项目时，我们定义了 `scarb test` 命令将运行 `snforge test` 命令。因此，在本章的其余部分，请认为 `scarb test` 命令在底层使用的是 `snforge test`。

合约的通常测试流程如下：

1. 声明要测试的合约类，由其名称标识
2. 将构造函数 calldata 序列化为数组
3. 部署合约并检索其地址
4. 与合约的入口点交互以测试各种场景

### 部署要测试的合约

在清单 {{#ref contract-deployment}} 中，我们编写了一个函数来部署 `PizzaFactory` 合约并设置分发器以进行交互。

```cairo,noplayground
fn deploy_pizza_factory() -> (IPizzaFactoryDispatcher, ContractAddress) {
    let contract = declare("PizzaFactory").unwrap().contract_class();

    let owner: ContractAddress = contract_address_const::<'owner'>();
    let constructor_calldata = array![owner.into()];

    let (contract_address, _) = contract.deploy(@constructor_calldata).unwrap();

    let dispatcher = IPizzaFactoryDispatcher { contract_address };

    (dispatcher, contract_address)
}
```

{{#label contract-deployment}} <span class="caption">清单 {{#ref contract-deployment}} 部署要测试的合约</span>

### 测试我们的合约

确定你的合约应遵守的行为是编写测试的第一步。在 `PizzaFactory` 合约中，我们确定合约应具有以下行为：

- 部署后，合约所有者应设置为构造函数中提供的地址，工厂应拥有 10 个单位的 pepperoni 和 pineapple，并且没有创建披萨。
- 如果有人试图制作披萨并且他们不是所有者，则操作应失败。否则，披萨计数应增加，并且应发出事件。
- 如果有人试图获取合约的所有权并且他们不是所有者，则操作应失败。否则，所有者应更新。

#### 使用 `load` 访问存储变量

```cairo,noplayground
#[test]
fn test_constructor() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();

    let pepperoni_count = load(pizza_factory_address, selector!("pepperoni"), 1);
    let pineapple_count = load(pizza_factory_address, selector!("pineapple"), 1);
    assert_eq!(pepperoni_count, array![10]);
    assert_eq!(pineapple_count, array![10]);
    assert_eq!(pizza_factory.get_owner(), owner());
}
```

{{#label test-constructor}} <span class="caption">清单 {{#ref test-constructor}}: 通过加载存储变量测试初始状态 </span>

一旦我们的合约部署完毕，我们想断言初始值已按预期设置。如果我们的合约有一个返回存储变量值的入口点，我们可以调用此入口点。否则，我们可以使用 `snforge` 中的 `load` 函数在我们的合约内加载存储变量的值，即使它未通过入口点公开。

#### 使用 `start_cheat_caller_address` 模拟调用者地址

我们要工厂的安全性依赖于所有者是唯一能够制作披萨和转移所有权的人。为了测试这一点，我们可以使用 `start_cheat_caller_address` 函数来模拟调用者地址并断言合约按预期行事。

```cairo,noplayground
#[test]
fn test_change_owner_should_change_owner() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();

    let new_owner: ContractAddress = contract_address_const::<'new_owner'>();
    assert_eq!(pizza_factory.get_owner(), owner());

    start_cheat_caller_address(pizza_factory_address, owner());

    pizza_factory.change_owner(new_owner);

    assert_eq!(pizza_factory.get_owner(), new_owner);
}

#[test]
#[should_panic(expected: "Only the owner can set ownership")]
fn test_change_owner_should_panic_when_not_owner() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();
    let not_owner = contract_address_const::<'not_owner'>();
    start_cheat_caller_address(pizza_factory_address, not_owner);
    pizza_factory.change_owner(not_owner);
    stop_cheat_caller_address(pizza_factory_address);
}
```

{{#label test-owner}} <span class="caption">清单 {{#ref test-owner}}: 通过模拟调用者地址测试合约的所有权 </span>

使用 `start_cheat_caller_address`，我们首先作为所有者调用 `change_owner` 函数，然后作为不同的地址调用。我们断言当调用者不是所有者时操作失败，并且当调用者是所有者时所有者更新。

#### 使用 `spy_events` 捕获事件

创建披萨时，合约会发出事件。为了测试这一点，我们可以使用 `spy_events` 函数来捕获发出的事件，并断言事件是使用预期的参数发出的。自然，我们也可以断言披萨计数已增加，并且只有所有者可以制作披萨。

```cairo,noplayground
#[test]
#[should_panic(expected: "Only the owner can make pizza")]
fn test_make_pizza_should_panic_when_not_owner() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();
    let not_owner = contract_address_const::<'not_owner'>();
    start_cheat_caller_address(pizza_factory_address, not_owner);

    pizza_factory.make_pizza();
}

#[test]
fn test_make_pizza_should_increment_pizza_counter() {
    // Setup
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();
    start_cheat_caller_address(pizza_factory_address, owner());
    let mut spy = spy_events();

    // When
    pizza_factory.make_pizza();

    // Then
    let expected_event = PizzaEvents::PizzaEmission(PizzaEmission { counter: 1 });
    assert_eq!(pizza_factory.count_pizza(), 1);
    spy.assert_emitted(@array![(pizza_factory_address, expected_event)]);
}
```

{{#label capture-pizza-emission-event}} <span class="caption">清单 {{#ref capture-pizza-emission-event}}: 测试创建披萨时发出的事件</span>

#### 使用 `contract_state_for_testing` 访问内部函数

到目前为止我们看到的所有测试都使用了涉及部署合约并与合约入口点交互的工作流。但是，有时我们可能希望直接测试合约的内部，而不部署合约。如果我们纯粹用 Cairo 术语进行推理，这该如何完成呢？

回想一下 `ContractState` 结构体，它用作合约所有入口点的参数。简而言之，此结构体包含零大小的字段，对应于合约的存储变量。这些字段的唯一目的是允许 Cairo 编译器生成用于访问存储变量的正确代码。如果我们可以创建此结构体的实例，我们就可以直接访问这些存储变量，而无需部署合约……

……这正是 `contract_state_for_testing` 函数所做的！它创建一个 `ContractState` 结构体的实例，允许我们调用任何以 `ContractState` 结构体为参数的函数，而无需部署合约。为了正确地与存储变量交互，我们需要手动导入定义对存储变量访问的 trait。

```cairo,noplayground
use crate::pizza::PizzaFactory::{InternalTrait};
use crate::pizza::{IPizzaFactoryDispatcher, IPizzaFactoryDispatcherTrait, PizzaFactory};

fn owner() -> ContractAddress {
    contract_address_const::<'owner'>()
}

//ANCHOR: deployment
fn deploy_pizza_factory() -> (IPizzaFactoryDispatcher, ContractAddress) {
    let contract = declare("PizzaFactory").unwrap().contract_class();

    let owner: ContractAddress = contract_address_const::<'owner'>();
    let constructor_calldata = array![owner.into()];

    let (contract_address, _) = contract.deploy(@constructor_calldata).unwrap();

    let dispatcher = IPizzaFactoryDispatcher { contract_address };

    (dispatcher, contract_address)
}
//ANCHOR_END: deployment

//ANCHOR: test_constructor
#[test]
fn test_constructor() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();

    let pepperoni_count = load(pizza_factory_address, selector!("pepperoni"), 1);
    let pineapple_count = load(pizza_factory_address, selector!("pineapple"), 1);
    assert_eq!(pepperoni_count, array![10]);
    assert_eq!(pineapple_count, array![10]);
    assert_eq!(pizza_factory.get_owner(), owner());
}
//ANCHOR_END: test_constructor

//ANCHOR: test_owner
#[test]
fn test_change_owner_should_change_owner() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();

    let new_owner: ContractAddress = contract_address_const::<'new_owner'>();
    assert_eq!(pizza_factory.get_owner(), owner());

    start_cheat_caller_address(pizza_factory_address, owner());

    pizza_factory.change_owner(new_owner);

    assert_eq!(pizza_factory.get_owner(), new_owner);
}

#[test]
#[should_panic(expected: "Only the owner can set ownership")]
fn test_change_owner_should_panic_when_not_owner() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();
    let not_owner = contract_address_const::<'not_owner'>();
    start_cheat_caller_address(pizza_factory_address, not_owner);
    pizza_factory.change_owner(not_owner);
    stop_cheat_caller_address(pizza_factory_address);
}
//ANCHOR_END: test_owner

//ANCHOR: test_make_pizza
#[test]
#[should_panic(expected: "Only the owner can make pizza")]
fn test_make_pizza_should_panic_when_not_owner() {
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();
    let not_owner = contract_address_const::<'not_owner'>();
    start_cheat_caller_address(pizza_factory_address, not_owner);

    pizza_factory.make_pizza();
}

#[test]
fn test_make_pizza_should_increment_pizza_counter() {
    // Setup
    let (pizza_factory, pizza_factory_address) = deploy_pizza_factory();
    start_cheat_caller_address(pizza_factory_address, owner());
    let mut spy = spy_events();

    // When
    pizza_factory.make_pizza();

    // Then
    let expected_event = PizzaEvents::PizzaEmission(PizzaEmission { counter: 1 });
    assert_eq!(pizza_factory.count_pizza(), 1);
    spy.assert_emitted(@array![(pizza_factory_address, expected_event)]);
}
//ANCHOR_END: test_make_pizza

//ANCHOR: test_internals
#[test]
fn test_set_as_new_owner_direct() {
    let mut state = PizzaFactory::contract_state_for_testing();
    let owner: ContractAddress = contract_address_const::<'owner'>();
    state.set_owner(owner);
    assert_eq!(state.owner.read(), owner);
}
//ANCHOR_END: test_internals


```

{{#label test-internal}} <span class="caption">清单 {{#ref test-internal}}: 对我们的合约进行单元测试而无需部署</span>

这些导入使我们可以访问我们的内部函数（特别是 `set_owner`），以及对 `owner` 存储变量的读/写访问权限。一旦我们有了这些，我们就可以直接与合约交互，通过调用可以通过 `InternalTrait` 访问的 `set_owner` 方法来更改所有者的地址，并读取 `owner` 存储变量。

> 注意：两种方法不能同时使用。如果你决定部署合约，你需要使用分发器与它交互。如果你决定测试内部函数，你需要直接与 `ContractState` 对象交互。

```bash,noplayground
$ scarb test 
     Running test listing_02_pizza_factory_snfoundry (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(listing_02_pizza_factory_snfoundry_unittest) listing_02_pizza_factory_snfoundry v0.1.0 (listings/ch104-starknet-smart-contracts-security/listing_02_pizza_factory_snfoundry/Scarb.toml)
warn: Usage of deprecated feature `"deprecated-starknet-consts"` with no `#[feature("deprecated-starknet-consts")]` attribute. Note: "Use `TryInto::try_into` in const context instead."
 --> listings/ch104-starknet-smart-contracts-security/listing_02_pizza_factory_snfoundry/src/tests/foundry_test.cairo:8:33
use starknet::{ContractAddress, contract_address_const};
                                ^^^^^^^^^^^^^^^^^^^^^^

    Finished `dev` profile target(s) in 1 second


Collected 6 test(s) from listing_02_pizza_factory_snfoundry package
Running 6 test(s) from src/
[PASS] listing_02_pizza_factory_snfoundry::tests::foundry_test::test_set_as_new_owner_direct (l1_gas: ~0, l1_data_gas: ~160, l2_gas: ~80000)
[PASS] listing_02_pizza_factory_snfoundry::tests::foundry_test::test_constructor (l1_gas: ~0, l1_data_gas: ~384, l2_gas: ~360000)
[PASS] listing_02_pizza_factory_snfoundry::tests::foundry_test::test_change_owner_should_panic_when_not_owner (l1_gas: ~0, l1_data_gas: ~384, l2_gas: ~360000)
[PASS] listing_02_pizza_factory_snfoundry::tests::foundry_test::test_make_pizza_should_increment_pizza_counter (l1_gas: ~0, l1_data_gas: ~480, l2_gas: ~655360)
[PASS] listing_02_pizza_factory_snfoundry::tests::foundry_test::test_make_pizza_should_panic_when_not_owner (l1_gas: ~0, l1_data_gas: ~384, l2_gas: ~360000)
[PASS] listing_02_pizza_factory_snfoundry::tests::foundry_test::test_change_owner_should_change_owner (l1_gas: ~0, l1_data_gas: ~384, l2_gas: ~640000)
Tests: 6 passed, 0 failed, 0 ignored, 0 filtered out


```

测试的输出显示所有测试都成功通过，以及每个测试消耗的 gas 估算值。

## 总结

在本章中，我们学习了如何使用 Starknet Foundry 测试智能合约。我们演示了如何部署合约并使用分发器与之交互。我们还展示了如何通过模拟调用者地址和捕获事件来测试合约的行为。最后，我们演示了如何直接测试合约的内部函数，而无需部署合约。

要了解有关 Starknet Foundry 的更多信息，请参阅 [Starknet Foundry 文档](https://foundry-rs.github.io/starknet-foundry/index.html)。
