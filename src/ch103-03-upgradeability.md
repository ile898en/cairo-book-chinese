# 可升级合约

Starknet 通过更新合约源代码的系统调用具有原生可升级性，消除了对代理的需求。

> **⚠️ 警告** 升级前请确保遵循安全建议。

## Starknet 中的可升级性如何工作

为了更好地理解 Starknet 中的可升级性如何工作，重要的是要理解合约及其合约类之间的区别。

[合约类 (Contract Classes)][class hash doc] 代表程序的源代码。所有合约都与一个类相关联，许多合约可以是同一个类的实例。类通常由 [类哈希 (class hash)][class hash doc] 表示，在部署某个类的合约之前，需要声明类哈希。

合约实例是对应于一个类的已部署合约，具有其自己的存储。

[class hash doc]:
  https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/class-hash
[syscalls doc]:
  https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/system-calls-cairo1/

## 替换合约类

### `replace_class_syscall`

`replace_class` [系统调用][syscalls doc] 允许合约在部署后通过替换其类哈希来更新其源代码。

要升级合约，请公开一个执行 `replace_class_syscall` 的入口点，将新类哈希作为参数：

```cairo,noplayground
use core::num::traits::Zero;
use starknet::{ClassHash, syscalls};

fn upgrade(new_class_hash: ClassHash) {
    assert!(!new_class_hash.is_zero(), "Class hash cannot be zero");
    syscalls::replace_class_syscall(new_class_hash).unwrap();
}
```

{{#label replace-class}} <span class="caption">清单 {{#ref replace-class}}: 公开 `replace_class_syscall` 以更新合约的类</span>

> **📌 注意**: 如果合约在部署时没有此机制，其类哈希仍可以通过 [库调用](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/system-calls-cairo1/#library_call) 进行替换。

> **⚠️ 警告**: 升级前彻底审查更改及其潜在影响，因为这是一个具有安全隐患的微妙过程。不要允许任意地址升级你的合约。

## OpenZeppelin 的 Upgradeable 组件

OpenZeppelin Contracts for Cairo 提供了 `Upgradeable` 组件，可以嵌入到你的合约中使其可升级。此组件是向你的合约添加可升级性的简单方法，同时依靠经过审计的库。

### 用法

升级通常是非常敏感的操作，通常需要某种形式的访问控制来避免未经授权的升级。本示例中使用 `Ownable` 组件将可升级性限制为单个地址，以便合约所有者拥有升级合约的独占权利。

```cairo,noplayground
#[starknet::contract]
mod UpgradeableContract {
    use openzeppelin::access::ownable::OwnableComponent;
    use openzeppelin_upgrades::UpgradeableComponent;
    use openzeppelin_upgrades::interface::IUpgradeable;
    use starknet::{ClassHash, ContractAddress};

    component!(path: OwnableComponent, storage: ownable, event: OwnableEvent);
    component!(path: UpgradeableComponent, storage: upgradeable, event: UpgradeableEvent);

    // Ownable Mixin
    #[abi(embed_v0)]
    impl OwnableMixinImpl = OwnableComponent::OwnableMixinImpl<ContractState>;
    impl OwnableInternalImpl = OwnableComponent::InternalImpl<ContractState>;

    // Upgradeable
    impl UpgradeableInternalImpl = UpgradeableComponent::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        #[substorage(v0)]
        ownable: OwnableComponent::Storage,
        #[substorage(v0)]
        upgradeable: UpgradeableComponent::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        #[flat]
        OwnableEvent: OwnableComponent::Event,
        #[flat]
        UpgradeableEvent: UpgradeableComponent::Event,
    }

    #[constructor]
    fn constructor(ref self: ContractState, owner: ContractAddress) {
        self.ownable.initializer(owner);
    }

    #[abi(embed_v0)]
    impl UpgradeableImpl of IUpgradeable<ContractState> {
        fn upgrade(ref self: ContractState, new_class_hash: ClassHash) {
            // This function can only be called by the owner
            self.ownable.assert_only_owner();

            // Replace the class hash upgrading the contract
            self.upgradeable.upgrade(new_class_hash);
        }
    }
}
```

{{#label upgradeable-contract}} <span class="caption">清单 {{#ref upgradeable-contract}} 在合约中集成 OpenZeppelin 的 Upgradeable 组件</span>

`UpgradeableComponent` 提供：

- 一个安全执行类替换的内部 `upgrade` 函数
- 一个当升级成功时发出的 `Upgraded` 事件
- 防止升级到零类哈希的保护

有关更多信息，请参阅 [OpenZeppelin 文档 API 参考][oz upgradeability api]。

## 安全注意事项

升级可能是非常敏感的操作，执行升级时应始终将安全性放在首位。升级前请确保彻底审查更改及其后果。需要考虑的一些方面包括：

- **API 更改** 可能会影响集成。例如，更改外部函数的参数可能会破坏调用你的合约的现有合约或链下系统。
- **存储更改** 可能会导致数据丢失（例如，更改存储槽名称，使现有存储无法访问）或数据损坏（例如，更改存储槽类型，或存储在存储中的结构体的组织方式）。
- **存储冲突**（例如，错误地重用来自另一个组件的相同存储槽）也是可能的，尽管如果遵循最佳实践（例如在存储变量前加上组件名称）则不太可能发生。
- 在 OpenZeppelin 合约版本之间升级之前，请务必检查向后兼容性。

[oz upgradeability api]:
  https://docs.openzeppelin.com/contracts-cairo/alpha/api/upgrades
