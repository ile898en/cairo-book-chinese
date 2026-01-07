# 使用 ERC20 代币

Starknet 上的 ERC20 标准为同质化代币提供了统一的接口。这确保了任何同质化代币都可以在整个生态系统中以可预测的方式使用。本节探讨如何使用 OpenZeppelin Contracts for Cairo 创建 ERC20 代币，这是该标准的经过审计的实现。

> 注意：虽然 Openzeppelin 组件经过了审计，通过你应该始终测试并确保你的代码不会被利用。本节提供的示例仅用于教育目的，不能用于生产。

首先，我们将构建一个具有固定供应量的基本 ERC20 代币。该合约演示了使用 OpenZeppelin 组件创建代币的核心结构。

## 基本 ERC20 合约

```cairo,noplayground
#[starknet::contract]
pub mod BasicERC20 {
    use openzeppelin::token::erc20::{DefaultConfig, ERC20Component, ERC20HooksEmptyImpl};
    use starknet::ContractAddress;

    component!(path: ERC20Component, storage: erc20, event: ERC20Event);

    // ERC20 Mixin
    #[abi(embed_v0)]
    impl ERC20MixinImpl = ERC20Component::ERC20MixinImpl<ContractState>;
    impl ERC20InternalImpl = ERC20Component::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        erc20: ERC20Component::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        #[flat]
        ERC20Event: ERC20Component::Event,
    }

    #[constructor]
    fn constructor(ref self: ContractState, initial_supply: u256, recipient: ContractAddress) {
        let name = "MyToken";
        let symbol = "MTK";

        self.erc20.initializer(name, symbol);
        self.erc20.mint(recipient, initial_supply);
    }
}
```

{{#label basic-erc20}} <span class="caption">清单 {{#ref basic-erc20}}: 使用 OpenZeppelin 的基本 ERC20 代币实现</span>

### 理解实现

此合约是用 OpenZeppelin 的组件系统构建的。它嵌入了 `ERC20Component`，其中包含 ERC20 代币的所有核心逻辑，包括转账、批准和余额跟踪功能。为了使这些功能直接在合约上可用，我们实现了 `ERC20MixinImpl` trait。这种模式避免了为 ERC20 接口中的每个函数编写样板代码的需要。

当合约部署时，其构造函数被调用。构造函数首先通过调用 ERC20 组件上的 `initializer` 函数来初始化代币的元数据 —— 它的名称和符号。然后它铸造整个初始供应量并将其分配给部署合约的地址。由于没有其他函数可以创建新代币，因此总供应量从部署那一刻起是固定的。

合约的存储很少，只包含 `ERC20Component` 的状态。这包括跟踪代币余额和津贴的映射，以及代币的名称、符号和总供应量，但从合约的角度来看是抽象的。

我们刚刚实现的合约相当简单：它是一个固定供应量的代币，没有额外的功能。但我们也可以使用 OpenZeppelin 组件库来构建更复杂的代币！

以下示例展示了如何在保持符合 ERC20 标准的同时添加新功能。

### 可铸造和可销毁的代币

此扩展添加了铸造新代币和销毁现有代币的功能，允许代币供应量在部署后发生变化。这对于供应量需要根据协议活动或治理进行调整的代币很有用。

```cairo,noplayground
#[starknet::contract]
pub mod MintableBurnableERC20 {
    use openzeppelin::access::ownable::OwnableComponent;
    use openzeppelin::token::erc20::{DefaultConfig, ERC20Component, ERC20HooksEmptyImpl};
    use starknet::ContractAddress;

    component!(path: OwnableComponent, storage: ownable, event: OwnableEvent);
    component!(path: ERC20Component, storage: erc20, event: ERC20Event);

    // Ownable Mixin
    #[abi(embed_v0)]
    impl OwnableMixinImpl = OwnableComponent::OwnableMixinImpl<ContractState>;
    impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;

    // ERC20 Mixin
    #[abi(embed_v0)]
    impl ERC20MixinImpl = ERC20Component::ERC20MixinImpl<ContractState>;
    impl ERC20InternalImpl = ERC20Component::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        ownable: OwnableComponent::Storage,
        #[substorage(v0)]
        erc20: ERC20Component::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        #[flat]
        OwnableEvent: OwnableComponent::Event,
        #[flat]
        ERC20Event: ERC20Component::Event,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        let name = "MintableBurnableToken";
        let symbol = "MBT";

        self.erc20.initializer(name, symbol);
        self.ownable.initializer(owner);
    }

    #[external(v0)]
    fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) {
        // Only owner can mint new tokens
        self.ownable.assert_only_owner();
        self.erc20.mint(recipient, amount);
    }

    #[external(v0)]
    fn burn(ref self: ContractState, amount: u256) {
        // Any token holder can burn their own tokens
        let caller = starknet::get_caller_address();
        self.erc20.burn(caller, amount);
    }
}
```

{{#label mintable-burnable-erc20}} <span class="caption">清单 {{#ref mintable-burnable-erc20}}: 具有铸造和销毁功能的 ERC20</span>

此合约引入了 `OwnableComponent` 来管理访问控制。部署合约的地址成为其所有者。`mint` 函数仅限于所有者，所有者可以创建新代币并将其分配给任何地址，从而增加总供应量。

`burn` 函数允许任何代币持有者销毁他们自己的代币。此操作将代币从流通中永久移除并减少总供应量。

为了使这些函数向公众公开，我们只需在合约中将它们标记为 `#[external]`。它们成为合约入口点的一部分，任何人都可以调用它们。

### 带有访问控制的可暂停代币

第二个扩展引入了一个更复杂的安全模型，具有基于角色的权限和紧急暂停功能。这种模式对于需要对操作进行细粒度控制以及在危机期间（例如安全事件）停止活动的方法的协议非常有用。

```cairo,noplayground
#[starknet::contract]
pub mod PausableERC20 {
    use openzeppelin::access::accesscontrol::AccessControlComponent;
    use openzeppelin::introspection::src5::SRC5Component;
    use openzeppelin::security::pausable::PausableComponent;
    use openzeppelin::token::erc20::{DefaultConfig, ERC20Component};
    use starknet::ContractAddress;

    const PAUSER_ROLE: felt252 = selector!("PAUSER_ROLE");
    const MINTER_ROLE: felt252 = selector!("MINTER_ROLE");

    component!(path: AccessControlComponent, storage: accesscontrol, event: AccessControlEvent);
    component!(path: SRC5Component, storage: src5, event: SRC5Event);
    component!(path: PausableComponent, storage: pausable, event: PausableEvent);
    component!(path: ERC20Component, storage: erc20, event: ERC20Event);

    // AccessControl
    #[abi(embed_v0)]
    impl AccessControlImpl =
        AccessControlComponent::AccessControlImpl<ContractState>;
    impl AccessControlInternalImpl = AccessControlComponent::InternalImpl<ContractState>;

    // SRC5
    #[abi(embed_v0)]
    impl SRC5Impl = SRC5Component::SRC5Impl<ContractState>;

    // Pausable
    #[abi(embed_v0)]
    impl PausableImpl = PausableComponent::PausableImpl<ContractState>;
    impl PausableInternalImpl = PausableComponent::InternalImpl<ContractState>;

    // ERC20
    #[abi(embed_v0)]
    impl ERC20MixinImpl = ERC20Component::ERC20MixinImpl<ContractState>;
    impl ERC20InternalImpl = ERC20Component::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        accesscontrol: AccessControlComponent::Storage,
        #[substorage(v0)]
        src5: SRC5Component::Storage,
        #[substorage(v0)]
        pausable: PausableComponent::Storage,
        #[substorage(v0)]
        erc20: ERC20Component::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        #[flat]
        AccessControlEvent: AccessControlComponent::Event,
        #[flat]
        SRC5Event: SRC5Component::Event,
        #[flat]
        PausableEvent: PausableComponent::Event,
        #[flat]
        ERC20Event: ERC20Component::Event,
    }

    // ERC20 Hooks implementation
    impl ERC20HooksImpl of ERC20Component::ERC20HooksTrait<ContractState> {
        fn before_update(
            ref self: ERC20Component::ComponentState<ContractState>,
            from: ContractAddress,
            recipient: ContractAddress,
            amount: u256,
        ) {
            let contract_state = self.get_contract();
            // Check that the contract is not paused
            contract_state.pausable.assert_not_paused();
        }
    }

    #[constructor]
    fn constructor(ref self: ContractState, admin: ContractAddress) {
        let name = "PausableToken";
        let symbol = "PST";

        self.erc20.initializer(name, symbol);

        // Grant admin role
        self.accesscontrol.initializer();
        self.accesscontrol._grant_role(AccessControlComponent::DEFAULT_ADMIN_ROLE, admin);

        // Grant specific roles to admin
        self.accesscontrol._grant_role(PAUSER_ROLE, admin);
        self.accesscontrol._grant_role(MINTER_ROLE, admin);
    }

    #[external(v0)]
    fn pause(ref self: ContractState) {
        self.accesscontrol.assert_only_role(PAUSER_ROLE);
        self.pausable.pause();
    }

    #[external(v0)]
    fn unpause(ref self: ContractState) {
        self.accesscontrol.assert_only_role(PAUSER_ROLE);
        self.pausable.unpause();
    }

    #[external(v0)]
    fn mint(ref self: ContractState, recipient: ContractAddress, amount: u256) {
        self.accesscontrol.assert_only_role(MINTER_ROLE);
        self.erc20.mint(recipient, amount);
    }
}
```

{{#label pausable-erc20}} <span class="caption">清单 {{#ref pausable-erc20}}: 具有可暂停转账和基于角色的访问控制的 ERC20</span>

此实现结合了四个组件：用于代币功能的 `ERC20Component`、用于管理角色的 `AccessControlComponent`、用于紧急停止机制的 `PausableComponent` 和用于接口检测的 `SRC5Component`。合约定义了两个角色：`PAUSER_ROLE`，可以暂停和取消暂停合约，以及 `MINTER_ROLE`，可以创建新代币。

与单一所有者不同，这种基于角色的系统允许分离管理职责。主管理员可以将 `PAUSER_ROLE` 授予安全团队，将 `MINTER_ROLE` 授予财务经理。

暂停功能使用钩子 (hook) 系统集成到代币的转账逻辑中。合约实现了 `ERC20HooksTrait`，其 `before_update` 函数会在任何代币转账或批准之前自动调用。此函数检查合约是否已暂停。如果具有 `PAUSER_ROLE` 的地址已暂停合约，则所有转账都将被阻止，直到取消暂停。这个钩子系统是扩展 ERC20 标准函数的基本功能的一种优雅方式，而无需重新定义它们。

在部署时，构造函数将所有角色授予部署者，部署者然后可以根据需要将这些角色委托给其他地址。

这些扩展实现展示了如何组合 OpenZeppelin 的组件来构建复杂且安全的合约。通过从标准的、经过审计的组件开始，开发人员可以添加自定义功能，而不会在安全性或标准合规性上妥协。

有关更高级的功能和详细文档，请参阅 [OpenZeppelin Contracts for Cairo 文档](https://docs.openzeppelin.com/contracts-cairo/)。
