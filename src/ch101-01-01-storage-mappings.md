# 使用映射存储键值对

Cairo 中的存储映射提供了一种将键与值相关联并将它们保存在合约存储中的方法。与传统的哈希表不同，存储映射不存储键数据本身；相反，它们使用键的哈希来计算对应于存储相应值的存储槽的地址。因此，不可能迭代存储映射的键。

<div align="center">
    <img src="mappings.png" alt="mappings" width="500px"/>
<div align="center">
    </div>
    {{#label fig-mappings}}
    <span class="caption">图 {{#ref fig-mappings}}: 将键映射到存储中的值</span>
</div>

映射没有长度的概念，也没有键值对是否设置的概念。所有值默认都设置为 0。因此，从映射中删除条目的唯一方法是将其值设置为该类型的默认值，对于 `u64` 类型为 `0`。

Cairo 核心库在 `core::starknet::storage` 模块中提供的 `Map` 类型用于在合约中声明映射。

要声明映射，请使用尖括号 `<>` 括起来的 `Map` 类型，指定键和值类型。在清单 {{#ref storage-mappings}} 中，我们创建了一个简单的合约，该合约存储映射到调用者地址的值。

> `Felt252Dict` 类型是 **内存** 类型，不能存储在合约存储中。对于键值对的持久存储，请使用 `Map` 类型，该类型是专为合约存储设计的 [幽灵类型 (phantom type)][phantom types]。然而，`Map` 有局限性：它不能作为常规变量实例化，不能用作函数参数，也不能作为常规结构体的成员包含在内。`Map` 只能作为合约存储结构体中的存储变量使用。要在内存中处理 `Map` 的内容或执行复杂操作，你需要将其元素复制到 `Felt252Dict` 或其他合适的数据结构中（或从中复制）。

## 声明和使用存储映射

<!-- TODO PHANTOM TYPES -->
<!-- [phantom types]: ./ch12-03-intro-to-phantom-data.html -->

```cairo, noplayground
#[starknet::contract]
mod UserValues {
    use starknet::storage::{
        Map, StoragePathEntry, StoragePointerReadAccess, StoragePointerWriteAccess,
    };
    use starknet::{ContractAddress, get_caller_address};

    #[storage]
    struct Storage {
        user_values: Map<ContractAddress, u64>,
    }

    #[abi(embed_v0)]
    impl UserValuesImpl of super::IUserValues<ContractState> {
        // ANCHOR: write
        fn set(ref self: ContractState, amount: u64) {
            let caller = get_caller_address();
            self.user_values.entry(caller).write(amount);
        }
        // ANCHOR_END: write

        // ANCHOR: read
        fn get(self: @ContractState, address: ContractAddress) -> u64 {
            self.user_values.entry(address).read()
        }
        // ANCHOR_END: read
    }
}
```

{{#label storage-mappings}} <span class="caption">清单 {{#ref storage-mappings}}: 在 Storage 结构体中声明存储映射</span>

要读取映射中对应于键的值，首先需要检索与该键关联的存储指针。这通过在存储映射变量上调用 `entry` 方法，传入键作为参数来完成。一旦有了 entry 路径，就可以在其上调用 `read` 函数来检索存储的值。

```cairo, noplayground
        fn get(self: @ContractState, address: ContractAddress) -> u64 {
            self.user_values.entry(address).read()
        }
```

同样，要在存储映射中写入值，你需要检索对应于键的存储指针。一旦有了这个存储指针，就可以调用 `write` 函数并传入要写入的值。

```cairo, noplayground
        fn set(ref self: ContractState, amount: u64) {
            let caller = get_caller_address();
            self.user_values.entry(caller).write(amount);
        }
```

## 嵌套映射

你还可以创建具有多个键的更复杂的映射。为了说明这一点，我们将实现一个代表分配给用户的仓库的合约，其中每个用户可以存储多个项目及其各自的数量。

`user_warehouse` 映射是一个存储映射，它将 `ContractAddress` 映射到另一个将 `u64`（项目 ID）映射到 `u64`（数量）的映射。这可以通过在存储结构体中声明 `Map<ContractAddress, Map<u64, u64>>` 来实现。`user_warehouse` 映射中的每个 `ContractAddress` 键对应一个用户的仓库，每个用户的仓库包含一个项目 ID 到其各自数量的映射。

```cairo, noplayground
    #[storage]
    struct Storage {
        user_warehouse: Map<ContractAddress, Map<u64, u64>>,
    }
```

在这种情况下，访问存储值的原则相同。你需要逐步遍历键，使用 `entry` 方法获取序列中下一个键的存储路径，最后在最内层的映射上调用 `read` 或 `write`。

```cairo, noplayground
        fn set_quantity(ref self: ContractState, item_id: u64, quantity: u64) {
            let caller = get_caller_address();
            self.user_warehouse.entry(caller).entry(item_id).write(quantity);
        }

        fn get_item_quantity(self: @ContractState, address: ContractAddress, item_id: u64) -> u64 {
            self.user_warehouse.entry(address).entry(item_id).read()
        }
```

## 映射的存储地址计算

存储在映射中的变量在存储中的地址根据以下规则计算：

- 对于单个键 `k`，键 `k` 处的值的地址是 `h(sn_keccak(variable_name), k)`，其中 `h` 是 Pedersen 哈希，最终值取模 \\( {2^{251}} - 256\\)。
- 对于多个键，地址计算为 `h(...h(h(sn_keccak(variable_name), k_1), k_2), ..., k_n)`，其中 `k_1, ..., k_n` 是构成映射的所有键。

如果映射的键是结构体，则结构体的每个元素都构成一个键。此外，结构体应该实现 `Hash` trait，可以通过 `#[derive(Hash)]` 属性派生。

## 总结

- 存储映射允许你将键映射到合约存储中的值。
- 使用 `Map` 类型声明映射。
- 使用 `entry` 方法和 `read`/`write` 函数访问映射。
- 映射可以包含其他映射，创建嵌套存储映射。
- 映射变量的地址是使用 `sn_keccak` 和 Pedersen 哈希函数计算的。
