# 合约存储 (Contract Storage)

合约的存储是一个持久存储空间，你可以在其中读取、写入、修改和持久化数据。存储是一个具有 \\(2^{251}\\) 个槽位的映射 (map)，其中每个槽位是一个初始化为 0 的 `felt252`。

每个存储槽位由一个 `felt252` 值标识，称为存储地址，它是从变量名称和取决于变量类型的参数计算得出的，这在 [“存储变量的地址”][storage addresses] 部分中进行了概述。

[storage addresses]:
  ./ch101-01-00-contract-storage.md#addresses-of-storage-variables

我们可以通过两种方式与合约的存储进行交互：

1.  通过高级存储变量，这些变量在用 `#[storage]` 属性注释的特殊 `Storage` 结构体中声明。
2.  使用其计算出的地址和低级 `storage_read` 和 `storage_write` 系统调用直接访问存储槽。当你需要执行不适合存储变量结构化方法的自定义存储操作时，这很有用，但通常应避免；因此，我们将不在本章中介绍它们。

## 声明和使用存储变量

Starknet 合约中的存储变量存储在一个名为 `Storage` 的特殊结构体中：

```cairo, noplayground
    #[storage]
    struct Storage {
        owner: Person,
        expiration: Expiration,
    }
```

`Storage` 结构体像任何其他 [结构体][structs] 一样，除了它 **必须** 用 `#[storage]` 属性注释。此注释告诉编译器生成与区块链状态交互所需的代码，并允许你从存储中读取和写入数据。此结构体可以包含任何实现 `Store` trait 的类型，包括其他结构体、枚举，以及 [存储映射 (Storage Mappings)][storage mappings]、[存储向量 (Storage Vectors)][storage vecs] 和 [存储节点 (Storage Nodes)][storage nodes]。在本节中，我们将重点介绍简单的存储变量，我们将在接下来的部分中看到如何存储更复杂的类型。

[storage mappings]: ./ch101-01-01-storage-mappings.md
[storage vecs]: ./ch101-01-02-storage-vecs.md
[storage nodes]: ./ch101-01-00-contract-storage.md#storage-nodes
[structs]: ./ch05-00-using-structs-to-structure-related-data.md

### 访问存储变量

可以使用 `read` 和 `write` 函数分别访问和修改存储在 `Storage` 结构体中的变量。所有这些函数都会由编译器为每个存储变量自动生成。

要读取 `owner` 存储变量的值（其类型为 `Person`），我们在 `owner` 变量上调用 `read` 函数，不传递任何参数。

```cairo, noplayground
        fn get_owner(self: @ContractState) -> Person {
            self.owner.read()
        }
```

要将新值写入存储变量的存储槽，我们调用 `write` 函数，将该值作为参数传递。在这里，我们只传入要写入 `owner` 变量的值，因为它是一个简单的变量。

```cairo, noplayground
    #[constructor]
    fn constructor(ref self: ContractState, owner: Person) {
        self.owner.write(owner);
    }
```

当使用复合类型时，可以通过在结构体的特定成员上调用 `read` 和 `write`，而不是在结构体变量本身上调用 `read` 和 `write`（这会对每个成员执行存储操作）。这允许你直接访问和修改结构体成员的值，从而最大限度地减少执行的存储操作量。在下面的示例中，`owner` 变量是 `Person` 类型。因此，它有一个名为 `name` 的属性，我们可以在其上调用 `read` 和 `write` 函数来访问和修改其值。

```cairo, noplayground
        fn get_owner_name(self: @ContractState) -> felt252 {
            self.owner.name.read()
        }
```

## 使用 `Store` Trait 存储自定义类型

`Store` trait 定义在 `starknet::storage_access` 模块中，用于指定类型应如何在存储中存储。为了使类型能够存储在存储中，它 **必须** 实现 `Store` trait。大多数核心库类型，例如无符号整数（`u8`、`u128`、`u256`...）、`felt252`、`bool`、`ByteArray`、`ContractAddress` 等都实现了 `Store` trait，因此可以直接存储而无需进一步操作。然而，**内存集合**，如 `Array<T>` 和 `Felt252Dict<T>`，**不能** 存储在合约存储中 —— 你必须改用特殊类型 `Vec<T>` 和 `Map<K, V>`。

但是如果你想存储一个你自己定义的类型，比如一个枚举或一个结构体呢？这种情况下，你必须显式告诉编译器如何存储这种类型。

在我们的例子中，我们想在存储中存储一个 `Person` 结构体，这只有通过为 `Person` 类型实现 `Store` trait 才可能实现。这可以通过在我们的结构体定义之上简单地添加 `#[derive(starknet::Store)]` 属性来实现。请注意，结构体的所有成员都需要实现 `Store` trait 才能派生该 trait。

```cairo, noplayground
    #[derive(Drop, Serde, starknet::Store)]
    pub struct Person {
        address: ContractAddress,
        name: felt252,
    }
```

同样，枚举只有在实现 `Store` trait 时才能写入存储，只要所有关联类型都实现了 `Store` trait，就可以简单地派生它。

合约存储中使用的枚举 **必须** 定义一个默认变体。当读取空存储槽时，将返回此默认变体 —— 否则，将导致运行时错误。

这有一个如何正确定义用于合约存储的枚举的示例：

```cairo, noplayground
    #[derive(Copy, Drop, Serde, starknet::Store)]
    pub enum Expiration {
        Finite: u64,
        #[default]
        Infinite,
    }
```

在这个例子中，我们为 `Infinite` 变体添加了 `#[default]` 属性。这告诉 Cairo 编译器，如果我们尝试从存储中读取未初始化的枚举，应该返回 `Infinite` 变体。

你可能已经注意到，我们还在我们的自定义类型上派生了 `Drop` 和 `Serde`。它们两个都是正确序列化传递给入口点的参数和反序列化其输出所必需的。

## 结构体存储布局

在 Starknet 上，结构体作为基本类型的序列存储在存储中。结构体的元素按其在结构体定义中定义的相同顺序存储。结构体的第一个元素存储在结构体的基地址，该地址如 [“存储变量的地址”][storage addresses] 一节中所述计算，可以通过 `var.__base_address__` 获得。后续元素存储在与前一个元素连续的地址处。例如，类型为 `Person` 的 `owner` 变量的存储布局将产生以下布局：

| 字段    | 地址                        |
| ------- | --------------------------- |
| name    | `owner.__base_address__`    |
| address | `owner.__base_address__ +1` |

注意，元组也类似地存储在合约的存储中，元组的第一个元素存储在基地址，后续元素连续存储。

## 枚举存储布局

当你存储枚举变体时，你实际上存储的是变体的索引和最终的关联值。此索引从枚举的第一个变体的 0 开始，每个后续变体加 1。如果你的变体具有关联值，则此值从紧随变体索引地址之后的地址开始存储。例如，假设我们有 `Expiration` 枚举，其中 `Finite` 变体带有相关的限制日期，而 `Infinite` 变体没有关联数据。`Finite` 变体的存储布局将如下所示：

| 元素                       | 地址                              |
| -------------------------- | --------------------------------- |
| 变体索引 (0 for Finite)    | `expiration.__base_address__`     |
| 关联的限制日期             | `expiration.__base_address__ + 1` |

而 `Infinite` 变体的存储布局将如下所示：

| 元素                         | 地址                          |
| ---------------------------- | ----------------------------- |
| 变体索引 (1 for Infinite)    | `expiration.__base_address__` |

<!-- TODO: add example -->

## 存储节点 (Storage Nodes)

存储节点是一种特殊的结构体，可以包含特定于存储的类型，例如 [`Map`][storage mappings]、[`Vec`][storage vecs] 或其他存储节点作为成员。与常规结构体不同，存储节点只能存在于合约存储中，不能在其外部实例化或使用。你可以将存储节点视为参与表示合约存储空间的树中的地址计算的中间节点。在下一小节中，我们将介绍如何在核心库中通过建模这个概念。

存储节点的主要好处是允许你创建更复杂的存储布局，包括自定义类型中的映射或向量，并允许你对相关数据进行逻辑分组，从而提高代码的可读性和可维护性。

存储节点是使用 `#[starknet::storage_node]` 属性定义的结构体。在实现投票系统的这个新合约中，我们实现了一个 `ProposalNode` 存储节点，其中包含一个 `Map<ContractAddress, bool>` 来跟踪提案的投票者，以及其他用于存储提案元数据的字段。

```cairo, noplayground
    #[starknet::storage_node]
    struct ProposalNode {
        title: felt252,
        description: felt252,
        yes_votes: u32,
        no_votes: u32,
        voters: Map<ContractAddress, bool>,
    }
```

访问存储节点时，你不能直接 `read` 或 `write` 之。相反，你必须访问其各个成员。这是来自我们的 `VotingSystem` 合约的一个示例，演示了我们如何填充 `ProposalNode` 存储节点的每个字段：

```cairo, noplayground
    #[external(v0)]
    fn create_proposal(ref self: ContractState, title: felt252, description: felt252) -> u32 {
        let mut proposal_count = self.proposal_count.read();
        let new_proposal_id = proposal_count + 1;

        let mut proposal = self.proposals.entry(new_proposal_id);
        proposal.title.write(title);
        proposal.description.write(description);
        proposal.yes_votes.write(0);
        proposal.no_votes.write(0);

        self.proposal_count.write(new_proposal_id);

        new_proposal_id
    }
```

因为还没有选民对此提案进行投票，所以我们在创建提案时不需要填充 `voters` 映射。但是，当选民试图投票时，我们完全可以访问 `voters` 映射来检查给定地址是否已经对此提案进行了投票：

```cairo, noplayground
    #[external(v0)]
    fn vote(ref self: ContractState, proposal_id: u32, vote: bool) {
        let mut proposal = self.proposals.entry(proposal_id);
        let caller = get_caller_address();
        let has_voted = proposal.voters.entry(caller).read();
        if has_voted {
            return;
        }
        proposal.voters.entry(caller).write(true);
    }
```

在这个例子中，我们访问特定提案 ID 的 `ProposalNode`。然后，我们通过从存储节点中的 `voters` 映射读取来检查调用者是否已经投票。如果他们还没有投票，我们就写入 `voters` 映射来标记他们现在已经投票了。

## 存储变量的地址

存储变量的地址计算如下：

- 如果变量是单个值，则地址是变量名称的 ASCII 编码的 `sn_keccak` 哈希。 `sn_keccak` 是 Keccak256 哈希函数的 Starknet 版本，其输出被截断为 250 位。

- 如果变量由多个值组成（即元组、结构体或枚举），我们也使用变量名称的 ASCII 编码的 `sn_keccak` 哈希来确定存储中的基地址。然后，根据类型，存储布局会有所不同。请参阅 ["存储自定义类型"][custom types storage layout] 部分。

- 如果变量是 [存储节点][storage nodes] 的一部分，则其地址基于反映节点结构的哈希链。对于存储变量 `variable_name` 内的存储节点成员 `m`，该成员的路径计算为 `h(sn_keccak(variable_name), sn_keccak(m))`，其中 `h` 是 Pedersen 哈希。此过程对于嵌套存储节点继续进行，构建表示通往叶节点的路径的哈希链。一旦到达叶节点，存储计算就像往常一样针对该类型的变量进行。

- 如果变量是 [Map][storage mappings] 或 [Vec][storage vecs]，地址是相对于存储基地址计算的，存储基地址是变量名称的 `sn_keccak` 哈希，以及映射的键或 Vec 中的索引。 ["存储映射"][storage mappings] 和 ["存储向量"][storage vecs] 部分描述了精确的计算。

你可以通过访问变量上的 `__base_address__` 属性（返回 `felt252` 值）来访问存储变量的基地址。

```cairo, noplayground
        self.total_names.__base_address__
```

此地址计算机制通过使用 `StoragePointers` 和 `StoragePaths` 的概念对合约存储空间进行建模来执行，我们现在将介绍这些概念。

[custom types storage layout]:
  ./ch101-01-00-contract-storage.md#storing-custom-types-with-the-store-trait

## 核心库中合约存储的建模

要理解存储变量如何在 Cairo 中存储，重要的是要注意它们不是连续存储的，而是存储在合约存储的不同位置。为了便于检索这些地址，核心库通过 `StoragePointers` 和 `StoragePaths` 系统提供了合约存储的模型。

每个存储变量都可以转换为 `StoragePointer`。该指针包含两个主要字段：

- 合约存储中存储变量的基地址。
- 该指针指向的特定存储槽相对于基地址的偏移量。

一图胜千言。让我们考虑上一节定义的 `Person` 结构体：

```cairo, noplayground
    #[derive(Drop, Serde, starknet::Store)]
    pub struct Person {
        address: ContractAddress,
        name: felt252,
    }
```

当我们写 `let x = self.owner;` 时，我们访问一个 `StorageBase` 类型的变量，该变量表示 `owner` 变量在合约存储中的基位置。从这个基地址，我们可以获取指向结构体字段（如 `name` 或 `address`）的指针，或者指向结构体本身的指针。在这些指针上，我们可以调用 `Store` trait 中定义的 `read` 和 `write` 来读取和写入指向的值。

当然，所有这些对开发人员都是透明的。我们可以像访问常规变量一样读取和写入结构体的字段，但编译器会在幕后将这些访问转换为适当的 `StoragePointer` 操作。

对于存储映射，过程类似，除了我们引入了一个中间类型 `StoragePath`。`StoragePath` 是存储节点和结构体字段的链，它们构成了通往特定存储槽的路径。例如，要访问包含在 `Map<ContractAddress, u128>` 中的值，过程如下：

1. 从 `Map` 的 `StorageBase` 开始，并将其转换为 `StoragePath`。
2. 走 `StoragePath` 使用 `entry` 方法到达所需的值，在 `Map` 的情况下，该方法将当前路径与下一个键进行哈希以生成下一个 `StoragePath`。
3. 重复步骤 2，直到 `StoragePath` 指向所需的值，将最终值转换为 `StoragePointer`。
4. 在该指针处读取或写入值。

注意，我们需要在能够对其进行读取或写入之前将 `ContractAddress` 转换为 `StoragePointer`。

![核心库中存储空间的建模](mermaid-storage-model.png)

<!-- ./mermaid-storage-model.txt -->

## 总结

在本章中，我们涵盖了以下关键点：

- **存储变量**：用于在区块链上存储持久数据。它们定义在用 `#[storage]` 属性注释的特殊 `Storage` 结构体中。
- **访问存储变量**：你可以使用自动生成的 `read` 和 `write` 函数读取和写入存储变量。对于结构体，你可以直接访问单个成员。
- **使用 `Store` Trait 自定义类型**：要存储像结构体和枚举这样的自定义类型，它们必须实现 `Store` trait。这可以使用 `#[derive(starknet::Store)]` 属性或编写你自己的实现来实现。
- **存储变量的地址**：存储变量的地址是使用其名称的 `sn_keccak` 哈希计算的，对于特殊类型还有额外的步骤。对于复杂类型，存储布局由类型的结构决定。
- **结构体和枚举存储布局**：结构体作为基本类型的序列存储，而枚举存储变体索引和潜在的关联值。
- **存储节点**：可以包含特定于存储的类型（如 `Map` 或 `Vec`）的特殊结构体。它们允许更复杂的存储布局，并且只能存在于合约存储中。

接下来，我们将深入关注 `Map` 和 `Vec` 类型。
