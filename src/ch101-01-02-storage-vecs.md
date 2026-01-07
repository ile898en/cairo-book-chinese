# 使用向量存储集合

`Vec` 类型提供了一种在合约存储中存储值集合的方法。在本节中，我们将探讨如何声明、添加元素到 `Vec` 以及从中检索元素，以及如何计算 `Vec` 变量的存储地址。

`Vec` 类型由 Cairo 核心库在 `starknet::storage` 模块内提供。其关联方法定义在 `VecTrait` 和 `MutableVecTrait` traits 中，你也需要导入它们以便对 `Vec` 类型进行读写操作。

> `Array<T>` 类型是 **内存** 类型，不能直接存储在合约存储中。对于存储，请使用 `Vec<T>` 类型，该类型是专为合约存储设计的 [幽灵类型 (phantom type)][phantom types]。然而，`Vec<T>` 有局限性：它不能作为常规变量实例化，不能用作函数参数，也不能作为常规结构体的成员包含在内。要处理 `Vec<T>` 的全部内容，你需要将其元素复制到内存 `Array<T>` 中（或从中复制）。

## 声明和使用存储向量

要声明存储向量，请使用尖括号 `<>` 括起来的 `Vec` 类型，指定它将存储的元素类型。在清单 {{#ref storage-vecs}} 中，我们创建了一个简单的合约，该合约注册所有调用它的地址并将它们存储在 `Vec` 中。然后我们可以检索第 `n` 个注册的地址，或所有注册的地址。

```cairo, noplayground
#[starknet::contract]
pub mod AddressList {
    use starknet::storage::{
        MutableVecTrait, StoragePointerReadAccess, StoragePointerWriteAccess, Vec, VecTrait,
    };
    use starknet::{ContractAddress, get_caller_address};

    //ANCHOR: storage_vecs
    #[storage]
    struct Storage {
        addresses: Vec<ContractAddress>,
    }
    //ANCHOR_END: storage_vecs

    #[abi(embed_v0)]
    impl AddressListImpl of super::IAddressList<ContractState> {
        //ANCHOR: push
        fn register_caller(ref self: ContractState) {
            let caller = get_caller_address();
            self.addresses.push(caller);
        }
        //ANCHOR_END: push

        //ANCHOR: read
        fn get_n_th_registered_address(
            self: @ContractState, index: u64,
        ) -> Option<ContractAddress> {
            self.addresses.get(index).map(|ptr| ptr.read())
        }

        fn get_all_addresses(self: @ContractState) -> Array<ContractAddress> {
            let mut addresses = array![];
            for i in 0..self.addresses.len() {
                addresses.append(self.addresses[i].read());
            }
            addresses
        }
        //ANCHOR_END: read

        //ANCHOR: modify
        fn modify_nth_address(ref self: ContractState, index: u64, new_address: ContractAddress) {
            self.addresses[index].write(new_address);
        }
        //ANCHOR_END: modify

        //ANCHOR: pop
        fn pop_last_registered_address(ref self: ContractState) -> Option<ContractAddress> {
            self.addresses.pop()
        }
        //ANCHOR_END: pop
    }
}
```

{{#label storage-vecs}} <span class="caption">清单 {{#ref storage-vecs}}: 在 Storage 结构体中声明存储 `Vec`</span>

要向 `Vec` 添加元素，可以使用 `push` 方法将元素添加到 `Vec` 的末尾。

```cairo, noplayground
        fn register_caller(ref self: ContractState) {
            let caller = get_caller_address();
            self.addresses.push(caller);
        }
```

要检索元素，你可以使用索引语法 (`vec[index]`) 或 `at`/`get` 方法来获取指向指定索引处元素的存储指针，然后调用 `read()` 来获取值。如果索引越界，`at`（和索引）会 panic，而 `get` 返回 `None`。

```cairo, noplayground
        fn get_n_th_registered_address(
            self: @ContractState, index: u64,
        ) -> Option<ContractAddress> {
            self.addresses.get(index).map(|ptr| ptr.read())
        }

        fn get_all_addresses(self: @ContractState) -> Array<ContractAddress> {
            let mut addresses = array![];
            for i in 0..self.addresses.len() {
                addresses.append(self.addresses[i].read());
            }
            addresses
        }
```

如果你想检索 Vec 的所有元素，你可以迭代存储 `Vec` 的索引，读取每个索引处的值，并将其追加到内存 `Array<T>` 中。同样，你不能在存储中存储 `Array<T>`：你需要迭代数组的元素并将它们追加到存储 `Vec<T>` 中。

在这一点上，你应该熟悉 [“合约存储”][contract-storage] 一节中介绍的存储指针和存储路径的概念，以及如何使用它们通过基于指针的模型访问存储变量。那么你会如何修改存储在 `Vec` 特定索引处的地址呢？

```cairo, noplayground
        fn modify_nth_address(ref self: ContractState, index: u64, new_address: ContractAddress) {
            self.addresses[index].write(new_address);
        }
```

答案相当简单：获取指向所需索引处存储指针的可变指针，并使用 `write` 方法修改该索引处的值。

你还可以使用 `pop` 方法删除存储 `Vec` 的最后一个元素。如果向量非空，它返回 `Some(value)`，否则返回 `None`，并相应地更新存储的长度。

```cairo, noplayground
        fn pop_last_registered_address(ref self: ContractState) -> Option<ContractAddress> {
            self.addresses.pop()
        }
```

[contract-storage]: ./ch101-01-00-contract-storage.md

## Vecs 的存储地址计算

存储在 `Vec` 中的变量在存储中的地址根据以下规则计算：

- `Vec` 的长度存储在基地址，计算为 `sn_keccak(variable_name)`。
- `Vec` 的元素存储在计算为 `h(base_address, i)` 的地址处，其中 `i` 是元素在 `Vec` 中的索引，`h` 是 Pedersen 哈希函数。

## 总结

- 使用 `Vec` 类型在合约存储中存储值集合
- 使用 `push` 方法添加元素，使用 `pop` 方法删除最后一个元素，使用 `at`/indexing 或 `get` 方法读取元素
- `Vec` 变量的地址是使用 `sn_keccak` 和 Pedersen 哈希函数计算的

这就结束了我们的合约存储之旅！在下一节中，我们将开始查看合约中定义的不同类型的函数。你已经知道它们中的大多数，因为我们在前面的章节中使用过它们，但我们将更详细地解释它们。
