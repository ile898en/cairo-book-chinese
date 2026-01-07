# 哈希 (Hashes)

本质上，哈希是将任意长度的输入数据（通常称为消息）转换为固定大小值的过程，该值通常称为“哈希”。这种转换是确定性的，意味着相同的输入将始终产生相同的哈希值。哈希函数是包括数据存储、密码学和数据完整性验证在内的各个领域的基本组件。它们在开发智能合约时经常被使用，尤其是在使用 [Merkle 树][merkle tree wiki] 时。

在本章中，我们将介绍 Cairo 核心库中原生实现的两个哈希函数：`Poseidon` 和 `Pedersen`。我们将讨论何时以及如何使用它们，并查看 Cairo 程序的示例。

[merkle tree wiki]: https://zh.wikipedia.org/wiki/Merkle树

### Cairo 中的哈希函数

Cairo 核心库提供了两个哈希函数：Pedersen 和 Poseidon。

Pedersen 哈希函数是依赖于 [椭圆曲线密码学][ec wiki] 的加密算法。这些函数对椭圆曲线上的点执行操作 —— 本质上是对这些点的位置进行数学运算 —— 这些操作在一个方向上很容易做，但在很难撤消。这种单向难度基于椭圆曲线离散对数问题 (ECDLP)，这是一个非常难以解决的问题，从而保证了哈希函数的安全性。逆转这些操作的难度使得 Pedersen 哈希函数在加密目的上既安全又可靠。

Poseidon 是一族设计为作为代数电路非常高效的哈希函数。它的设计对于零知识证明系统（包括 STARKs，即 Cairo）特别高效。Poseidon 使用一种称为“海绵构造 (sponge construction)”的方法，该方法吸收数据并使用称为 Hades 排列的过程安全地转换数据。Cairo 版本的 Poseidon 基于具有 [特定参数][poseidon parameters] 的三元素状态排列。

[ec wiki]: https://zh.wikipedia.org/wiki/椭圆曲线密码学
[poseidon parameters]:
  https://github.com/starkware-industries/poseidon/blob/main/poseidon3.txt

#### 何时使用它们？

Pedersen 是 Starknet 上使用的第一个哈希函数，并且仍然用于计算存储中变量的地址（例如，`LegacyMap` 使用 Pedersen 对 Starknet 上的存储映射的键进行哈希）。然而，由于 Poseidon 在使用 STARK 证明系统时比 Pedersen 更便宜、更快，它现在是 Cairo 程序中推荐使用的哈希函数。

### 使用哈希

核心库使得使用哈希变得容易。`Hash` trait 为所有可以转换为 `felt252` 的类型实现，包括 `felt252` 本身。对于像结构体这样的更复杂类型，派生 `Hash` 允许使用你选择的哈希函数轻松地对它们进行哈希 —— 前提是结构体的所有字段本身都是可哈希的。你不能在包含不可哈希值的结构体上派生 `Hash` trait，例如 `Array<T>` 或 `Felt252Dict<T>`，即使 `T` 本身是可哈希的。

`Hash` trait 伴随着 `HashStateTrait` 和 `HashStateExTrait`，它们定义了使用哈希的基本方法。它们允许你初始化一个哈希状态，该状态将在每次应用哈希函数后包含哈希的临时值，更新哈希状态，并在计算完成时将其定型。`HashStateTrait` 和 `HashStateExTrait` 定义如下：

```cairo,noplayground
/// A trait for hash state accumulators.
trait HashStateTrait<S> {
    fn update(self: S, value: felt252) -> S;
    fn finalize(self: S) -> felt252;
}

/// Extension trait for hash state accumulators.
trait HashStateExTrait<S, T> {
    /// Updates the hash state with the given value.
    fn update_with(self: S, value: T) -> S;
}

/// A trait for values that can be hashed.
trait Hash<T, S, +HashStateTrait<S>> {
    /// Updates the hash state with the given value.
    fn update_state(state: S, value: T) -> S;
}
```

要在你的代码中使用哈希，你必须首先导入相关的 traits 和函数。在以下示例中，我们将演示如何使用 Pedersen 和 Poseidon 哈希函数对结构体进行哈希。

第一步是用 `PoseidonTrait::new() -> HashState` 或 `PedersenTrait::new(base: felt252) -> HashState` 初始化哈希，具体取决于我们想要使用哪个哈希函数。然后可以使用 `update(self: HashState, value: felt252) -> HashState` 或 `update_with(self: S, value: T) -> S` 函数根据需要多次更新哈希状态。然后对哈希状态调用函数 `finalize(self: HashState) -> felt252`，它将哈希值作为 `felt252` 返回。

```cairo
use core::hash::{HashStateExTrait, HashStateTrait};
use core::poseidon::PoseidonTrait;

#[derive(Drop, Hash)]
struct StructForHash {
    first: felt252,
    second: felt252,
    third: (u32, u32),
    last: bool,
}

#[executable]
fn main() -> felt252 {
    let struct_to_hash = StructForHash { first: 0, second: 1, third: (1, 2), last: false };

    let hash = PoseidonTrait::new().update_with(struct_to_hash).finalize();
    hash
}
```

Pedersen 与 Poseidon 不同，因为它从一个基本状态开始。这个基本状态必须是 `felt252` 类型，这迫使我们要么使用 `update_with` 方法用任意基本状态对结构体进行哈希，要么将结构体序列化为数组以循环遍历其所有字段并将元素哈希在一起。

这是一个 Pedersen 哈希的简短示例：

```cairo
#[executable]
fn main() -> (felt252, felt252) {
    let struct_to_hash = StructForHash { first: 0, second: 1, third: (1, 2), last: false };

    // hash1 is the result of hashing a struct with a base state of 0
    let hash1 = PedersenTrait::new(0).update_with(struct_to_hash).finalize();

    let mut serialized_struct: Array<felt252> = ArrayTrait::new();
    Serde::serialize(@struct_to_hash, ref serialized_struct);
    let first_element = serialized_struct.pop_front().unwrap();
    let mut state = PedersenTrait::new(first_element);

    for value in serialized_struct {
        state = state.update(value);
    }

    // hash2 is the result of hashing only the fields of the struct
    let hash2 = state.finalize();

    (hash1, hash2)
}
```

### 高级哈希：使用 Poseidon 对数组进行哈希

让我们看一个对包含 `Span<felt252>` 的结构体进行哈希的例子。要对 `Span<felt252>` 或包含 `Span<felt252>` 的结构体进行哈希，你可以使用内置函数 `poseidon_hash_span(mut span: Span<felt252>) -> felt252`。同样，你可以通过对其 span 调用 `poseidon_hash_span` 来对 `Array<felt252>` 进行哈希。

首先，让我们导入以下 traits 和函数：

```cairo,noplayground
use core::hash::{HashStateExTrait, HashStateTrait};
use core::poseidon::{PoseidonTrait, poseidon_hash_span};
```

现在我们定义结构体。正如你可能注意到的，我们没有派生 `Hash` trait。如果你尝试为此结构体派生 `Hash` trait，它将导致错误，因为该结构体包含不可哈希的字段。

```cairo, noplayground
#[derive(Drop)]
struct StructForHashArray {
    first: felt252,
    second: felt252,
    third: Array<felt252>,
}
```

在这个例子中，我们初始化了一个 `HashState` (`hash`)，更新了它，然后对 `HashState` 调用函数 `finalize()` 以获取计算出的哈希 `hash_felt252`。我们在 `Array<felt252>` 的 `Span` 上使用了 `poseidon_hash_span` 来计算其哈希值。

```cairo
#[executable]
fn main() {
    let struct_to_hash = StructForHashArray { first: 0, second: 1, third: array![1, 2, 3, 4, 5] };

    let mut hash = PoseidonTrait::new().update(struct_to_hash.first).update(struct_to_hash.second);
    let hash_felt252 = hash.update(poseidon_hash_span(struct_to_hash.third.span())).finalize();
}
```
