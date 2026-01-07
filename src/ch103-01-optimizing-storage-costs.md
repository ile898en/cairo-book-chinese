# 优化存储成本

位打包 (Bit-packing) 是一个简单的概念：使用尽可能少的位来存储一条数据。如果做得好，它可以显着减少你需要存储的数据的大小。这在智能合约中尤其重要，因为存储非常昂贵。

在编写 Cairo 智能合约时，优化存储使用以降低 gas 成本非常重要。实际上，与交易相关的大部分成本都与存储更新有关；并且每个存储槽的写入都要花费 gas。这意味着通过将多个值打包到更少的槽中，你可以减少智能合约用户产生的 gas 成本。

## 整数结构和位运算符

整数是在一定数量的位上编码的，具体取决于其大小（例如，`u8` 整数是在 8 位上编码的）。

<div align="center">
    <img src="integer_in_bits.png" alt="a u8 integer in bits" width="500px"/>
<div align="center">
</div>
    <span class="caption">u8 整数在位中的表示</span>
</div>

直观地说，如果单个整数的大小大于或等于整数大小的总和（例如，一个 `u32` 中的两个 `u8` 和一个 `u16`），则可以将多个整数合并为一个整数。

但是，要做到这一点，我们需要一些位运算符：

- 乘以或除以 2 的幂分别将整数值向左或向右移动

<div align="center">
    <img src="shift.png" alt="shift operators" width="500px"/>
<div align="center">
</div>
    <span class="caption">向左或向右移动整数值</span>
</div>

- 对整数值应用掩码（`AND` 运算符）会隔离此整数的某些位

<div align="center">
    <img src="mask.png" alt="applying a mask" width="500px"/>
<div align="center">
</div>
    <span class="caption">用掩码隔离位</span>
</div>

- 相加（`OR` 运算符）两个整数将把两个值合并为一个。

<div align="center">
    <img src="combine.png" alt="combining two values" width="500px"/>
<div align="center">
</div>
    <span class="caption">合并两个整数</span>
</div>

有了这些位运算符，让我们在以下示例中看看如何将两个 `u8` 整数合并为一个 `u16` 整数（称为 `packing`，打包）以及反向操作（称为 `unpacking`，解包）：

<div align="center">
    <img src="pack.png" alt="packing and unpacking integer values" width="500px"/>
<div align="center">
</div>
    <span class="caption">打包和解包整数值</span>
</div>

## Cairo 中的位打包

Starknet 智能合约的存储是一个具有 2<sup>251</sup> 个槽位的映射 (map)，其中每个槽位是一个初始化为 0 的 `felt252`。

正如我们之前所看到的，为了减少由于存储更新引起的 gas 成本，我们必须使用尽可能少的位，因此我们必须通过打包它们来组织存储的变量。

例如，考虑以下具有 3 个不同类型字段的 `Sizes` 结构体：一个 `u8`，一个 `u32` 和一个 `u64`。总大小为 8 + 32 + 64 = 104 位。这小于一个槽位大小（即 251 位），因此我们可以将它们打包在一起存储到一个槽位中。

注意，因为它也适合 `u128`，所以使用最小的类型来打包所有变量是一个好习惯，因此这里应该使用 `u128`。

```cairo,noplayground
struct Sizes {
    tiny: u8,
    small: u32,
    medium: u64,
}
```

要将这 3 个变量打包到一个 `u128` 中，我们需要依次向左移动它们，最后将它们相加。

<div align="center">
    <img src="sizes-packing.png" alt="Sizes packing" width="800px"/>
<div align="center">
</div>
    <span class="caption">Sizes 打包</span>
</div>

要从 `u128` 中解包这 3 个变量，我们需要依次向右移动它们并使用掩码来隔离它们。

<div align="center">
    <img src="sizes-unpacking.png" alt="Sizes unpacking" width="800px"/>
<div align="center">
</div>
    <span class="caption">Sizes 解包</span>
</div>

## `StorePacking` Trait

Cairo 提供了 `StorePacking` trait 来支持将结构体字段打包到更少的存储槽中。`StorePacking<T, PackedT>` 是一个泛型 trait，它将你要打包的类型 (`T`) 和目标类型 (`PackedT`) 作为参数。它提供了两个要实现的函数：`pack` 和 `unpack`。

这是上一章示例的实现：

```cairo,noplayground
use starknet::storage_access::StorePacking;

#[derive(Drop, Serde)]
//ANCHOR:struct
struct Sizes {
    tiny: u8,
    small: u32,
    medium: u64,
}
//ANCHOR_END:struct

const TWO_POW_8: u128 = 0x100;
const TWO_POW_40: u128 = 0x10000000000;

const MASK_8: u128 = 0xff;
const MASK_32: u128 = 0xffffffff;

impl SizesStorePacking of StorePacking<Sizes, u128> {
    fn pack(value: Sizes) -> u128 {
        value.tiny.into() + (value.small.into() * TWO_POW_8) + (value.medium.into() * TWO_POW_40)
    }

    fn unpack(value: u128) -> Sizes {
        let tiny = value & MASK_8;
        let small = (value / TWO_POW_8) & MASK_32;
        let medium = (value / TWO_POW_40);

        Sizes {
            tiny: tiny.try_into().unwrap(),
            small: small.try_into().unwrap(),
            medium: medium.try_into().unwrap(),
        }
    }
}

#[starknet::contract]
mod SizeFactory {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use super::Sizes;
    use super::SizesStorePacking; //don't forget to import it!

    #[storage]
    struct Storage {
        remaining_sizes: Sizes,
    }

    #[abi(embed_v0)]
    fn update_sizes(ref self: ContractState, sizes: Sizes) {
        // This will automatically pack the
        // struct into a single u128
        self.remaining_sizes.write(sizes);
    }


    #[abi(embed_v0)]
    fn get_sizes(ref self: ContractState) -> Sizes {
        // this will automatically unpack the
        // packed-representation into the Sizes struct
        self.remaining_sizes.read()
    }
}
```

<div align="center">
    <span class="caption">通过实现 `StorePacking` trait 优化存储。</span>
</div>

在这个代码片段中，你可以看到：

- `TWO_POW_8` 和 `TWO_POW_40` 用在 `pack` 函数中左移，和在 `unpack` 函数中右移，
- `MASK_8` 和 `MASK_32` 用在 `unpack` 函数中隔离变量，
- 所有来自存储的变量都转换为 `u128` 以便能够使用位运算符。

此技术可用于适合打包存储类型位大小的任何字段组。例如，如果你有一个包含多个字段的结构体，其位大小加起来为 256 位，你可以将它们打包到一个 `u256` 变量中。如果位大小加起来为 512 位，你可以将它们打包到一个 `u512` 变量中，依此类推。你可以定义自己的结构体和逻辑来打包和解包它们。

剩下的工作由编译器神奇地完成 —— 如果一个类型实现了 `StorePacking` trait，那么编译器就会知道它可以使用 `Store` trait 的 `StoreUsingPacking` 实现，以便在写入前打包和从存储读取后解包。然而，一个重要的细节是，`StorePacking::pack` 输出的类型也必须实现 `Store` 以便 `StoreUsingPacking` 工作。大多数时候，我们会想要打包成 `felt252` 或 `u256` —— 但如果你想打包成你自己的类型，请确保这一个实现了 `Store` trait。
