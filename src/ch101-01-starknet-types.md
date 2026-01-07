# Starknet 类型

在 Starknet 上构建智能合约时，你将使用表示区块链特定概念的专用类型。这些类型允许你通过地址与已部署的合约进行交互，处理跨链通信并处理合约特定的数据类型。本章介绍核心库提供的 Starknet 特定类型。

## 合约地址 (Contract Address)

`ContractAddress` 类型表示 Starknet 上已部署合约的地址。每个已部署的合约都有一个唯一的地址，用于在网络上标识它。你将使用它来调用其他合约、检查调用者身份、管理访问控制以及任何涉及链上账户的事情。

```cairo
use starknet::{ContractAddress, get_caller_address};

#[starknet::interface]
pub trait IAddressExample<TContractState> {
    fn get_owner(self: @TContractState) -> ContractAddress;
    fn transfer_ownership(ref self: TContractState, new_owner: ContractAddress);
}

#[starknet::contract]
mod AddressExample {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use super::{ContractAddress, get_caller_address};

    #[storage]
    struct Storage {
        owner: ContractAddress,
    }

    #[constructor]
    fn constructor(ref self: ContractState, initial_owner: ContractAddress) {
        self.owner.write(initial_owner);
    }

    #[abi(embed_v0)]
    impl AddressExampleImpl of super::IAddressExample<ContractState> {
        fn get_owner(self: @ContractState) -> ContractAddress {
            self.owner.read()
        }

        fn transfer_ownership(ref self: ContractState, new_owner: ContractAddress) {
            let caller = get_caller_address();
            assert!(caller == self.owner.read(), "Only owner can transfer");
            self.owner.write(new_owner);
        }
    }
}
```

Starknet 中的合约地址具有 `[0, 2^251)` 的值范围，这是由类型系统强制执行的。你可以使用常规的 `TryInto` trait 从 `felt252` 创建 `ContractAddress`。

## 存储地址 (Storage Address)

`StorageAddress` 类型表示值在合约存储中的位置。虽然你通常不会直接创建这些地址（存储系统通过 [Map](./ch101-01-01-storage-mappings.md) 和 [Vec](./ch101-01-02-storage-vecs.md) 等类型为你处理），但了解此类型对于高级存储模式很重要。存储在 `Storage` 结构体中的每个值都有自己的 `StorageAddress`，并且可以按照 [存储 (Storage)](./ch101-01-00-contract-storage.md) 章节中定义的规则直接访问。

```cairo
#[starknet::contract]
mod StorageExample {
    use starknet::storage_access::StorageAddress;

    #[storage]
    struct Storage {
        value: u256,
    }

    // This is an internal function that demonstrates StorageAddress usage
    // In practice, you rarely need to work with StorageAddress directly
    fn read_from_storage_address(address: StorageAddress) -> felt252 {
        starknet::syscalls::storage_read_syscall(0, address).unwrap()
    }
}
```

存储地址与合约地址 `[0, 2^251)` 共享相同的值范围。相关的 `StorageBaseAddress` 类型表示可以与偏移量组合的基地址，其范围略小 `[0, 2^251 - 256)` 以容纳偏移量计算。

## 以太坊地址 (Ethereum Address)

`EthAddress` 类型表示 20 字节的以太坊地址，主要用于在 Starknet 上构建跨链应用程序。该类型用于 L1-L2 消息传递、代币桥接以及任何需要与以太坊交互的合约。

```cairo
use starknet::EthAddress;

#[starknet::interface]
pub trait IEthAddressExample<TContractState> {
    fn set_l1_contract(ref self: TContractState, l1_contract: EthAddress);
    fn send_message_to_l1(ref self: TContractState, recipient: EthAddress, amount: felt252);
}

#[starknet::contract]
mod EthAddressExample {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use starknet::syscalls::send_message_to_l1_syscall;
    use super::EthAddress;

    #[storage]
    struct Storage {
        l1_contract: EthAddress,
    }

    #[abi(embed_v0)]
    impl EthAddressExampleImpl of super::IEthAddressExample<ContractState> {
        fn set_l1_contract(ref self: ContractState, l1_contract: EthAddress) {
            self.l1_contract.write(l1_contract);
        }

        fn send_message_to_l1(ref self: ContractState, recipient: EthAddress, amount: felt252) {
            // Send a message to L1 with recipient and amount
            let payload = array![recipient.into(), amount];
            send_message_to_l1_syscall(self.l1_contract.read().into(), payload.span()).unwrap();
        }
    }

    #[l1_handler]
    fn handle_message_from_l1(ref self: ContractState, from_address: felt252, amount: felt252) {
        // Verify the message comes from the expected L1 contract
        assert!(from_address == self.l1_contract.read().into(), "Invalid L1 sender");
        // Process the message...
    }
}
```

此示例展示了 `EthAddress` 的主要用途：

- 存储 L1 合约地址
- 使用 `send_message_to_l1_syscall` 向以太坊发送消息
- 使用 `#[l1_handler]` 从 L1 接收和验证消息

`EthAddress` 类型确保类型安全，并且可以与 `felt252` 相互转换以进行 L1-L2 消息序列化。

## 类哈希 (Class Hash)

`ClassHash` 类型表示合约类（合约代码）的哈希。在 Starknet 的架构中，合约类与合约实例分部署，允许用于多个合约共享相同的代码。因此，你可以使用相同的类哈希部署多个合约，或者是将合约升级到新版本。

```cairo
use starknet::ClassHash;

#[starknet::interface]
pub trait IClassHashExample<TContractState> {
    fn get_implementation_hash(self: @TContractState) -> ClassHash;
    fn upgrade(ref self: TContractState, new_class_hash: ClassHash);
}

#[starknet::contract]
mod ClassHashExample {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use starknet::syscalls::replace_class_syscall;
    use super::ClassHash;

    #[storage]
    struct Storage {
        implementation_hash: ClassHash,
    }

    #[constructor]
    fn constructor(ref self: ContractState, initial_class_hash: ClassHash) {
        self.implementation_hash.write(initial_class_hash);
    }

    #[abi(embed_v0)]
    impl ClassHashExampleImpl of super::IClassHashExample<ContractState> {
        fn get_implementation_hash(self: @ContractState) -> ClassHash {
            self.implementation_hash.read()
        }

        fn upgrade(ref self: ContractState, new_class_hash: ClassHash) {
            replace_class_syscall(new_class_hash).unwrap();
            self.implementation_hash.write(new_class_hash);
        }
    }
}
```

类哈希具有与地址 `[0, 2^251)` 相同的值范围。它们唯一地标识特定版本的合约代码，并用于部署操作、代理模式和升级机制。

## 使用区块和交易信息

Starknet 提供了几个函数来访问有关当前执行上下文的信息。这些函数返回包含区块链状态信息的专用类型或结构体。

```cairo
#[starknet::interface]
pub trait IBlockInfo<TContractState> {
    fn get_block_info(self: @TContractState) -> (u64, u64);
    fn get_tx_info(self: @TContractState) -> (ContractAddress, felt252);
}

#[starknet::contract]
mod BlockInfoExample {
    use starknet::{get_block_info, get_tx_info};
    use super::ContractAddress;

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl BlockInfoImpl of super::IBlockInfo<ContractState> {
        fn get_block_info(self: @ContractState) -> (u64, u64) {
            let block_info = get_block_info();
            (block_info.block_number, block_info.block_timestamp)
        }

        fn get_tx_info(self: @ContractState) -> (ContractAddress, felt252) {
            let tx_info = get_tx_info();

            // Access transaction details
            let sender = tx_info.account_contract_address;
            let tx_hash = tx_info.transaction_hash;

            (sender, tx_hash)
        }
    }
}
```

`BlockInfo` 结构体包含有关当前区块的详细信息，包括其编号和时间戳。`TxInfo` 结构体提供特定于交易的信息，包括发送者的地址、交易哈希和费用详情。
