# 合约函数

在本节中，我们将查看你在 Starknet 智能合约中可能遇到的不同类型的函数。

函数可以通过 `self: ContractState` 对象轻松访问合约的状态，这抽象了底层系统调用（`storage_read_syscall` 和 `storage_write_syscall`）的复杂性。编译器提供了两个修饰符：`ref` 和 `@` 来修饰 `self`，旨在区分视图函数和外部函数。

让我们考虑清单 {{#ref reference-contract}} 中的 `NameRegistry` 合约，我们将在本章中使用它：

```cairo,noplayground
use starknet::ContractAddress;

#[starknet::interface]
pub trait INameRegistry<TContractState> {
    fn store_name(ref self: TContractState, name: felt252);
    fn get_name(self: @TContractState, address: ContractAddress) -> felt252;
}

// ANCHOR: contract
#[starknet::contract]
mod NameRegistry {
    use starknet::storage::{
        Map, StoragePathEntry, StoragePointerReadAccess, StoragePointerWriteAccess,
    };
    use starknet::{ContractAddress, get_caller_address};

    //ANCHOR: storage
    #[storage]
    struct Storage {
        names: Map<ContractAddress, felt252>,
        total_names: u128,
    }
    //ANCHOR_END: storage

    //ANCHOR: person
    #[derive(Drop, Serde, starknet::Store)]
    pub struct Person {
        address: ContractAddress,
        name: felt252,
    }
    //ANCHOR_END: person

    //ANCHOR: constructor
    #[constructor]
    fn constructor(ref self: ContractState, owner: Person) {
        self.names.entry(owner.address).write(owner.name);
        self.total_names.write(1);
    }
    //ANCHOR_END: constructor

    //ANCHOR: impl_public
    // Public functions inside an impl block
    #[abi(embed_v0)]
    impl NameRegistry of super::INameRegistry<ContractState> {
        //ANCHOR: external
        fn store_name(ref self: ContractState, name: felt252) {
            let caller = get_caller_address();
            self._store_name(caller, name);
        }
        //ANCHOR_END: external

        //ANCHOR: view
        fn get_name(self: @ContractState, address: ContractAddress) -> felt252 {
            //ANCHOR: read
            self.names.entry(address).read()
            //ANCHOR_END: read
        }
        //ANCHOR_END: view
    }
    //ANCHOR_END: impl_public

    //ANCHOR: standalone
    // Standalone public function
    #[external(v0)]
    fn get_contract_name(self: @ContractState) -> felt252 {
        'Name Registry'
    }
    //ANCHOR_END: standalone

    // ANCHOR: state_internal
    // ANCHOR: generate_trait
    // Could be a group of functions about a same topic
    #[generate_trait]
    impl InternalFunctions of InternalFunctionsTrait {
        fn _store_name(ref self: ContractState, user: ContractAddress, name: felt252) {
            let total_names = self.total_names.read();

            //ANCHOR: write
            self.names.entry(user).write(name);
            //ANCHOR_END: write

            self.total_names.write(total_names + 1);
        }
    }
    // ANCHOR_END: generate_trait

    // Free function
    fn get_total_names_storage_address(self: @ContractState) -> felt252 {
        //ANCHOR: total_names_address
        self.total_names.__base_address__
        //ANCHOR_END:total_names_address
    }
    // ANCHOR_END: state_internal
}
// ANCHOR_END: contract
```

{{#label reference-contract}} <span class="caption">清单 {{#ref reference-contract}}: 本章的参考合约</span>

## 1. 构造函数 (Constructors)

构造函数是一种特殊类型的函数，仅在部署合约时运行一次，可用于初始化合约的状态。

```cairo,noplayground
    #[constructor]
    fn constructor(ref self: ContractState, owner: Person) {
        self.names.entry(owner.address).write(owner.name);
        self.total_names.write(1);
    }
```

一些需要注意的重要规则：

1. 一个合约不能有多个构造函数。
2. 构造函数必须命名为 `constructor`，并且必须用 `#[constructor]` 属性注释。

`constructor` 函数可能接受参数，这些参数在部署合约时传递。在我们的例子中，我们传递一些对应于 `Person` 类型的值作为参数，以便在合约中存储 `owner` 信息（地址和名称）。

注意，`constructor` 函数 **必须** 将 `self` 作为第一个参数，对应于合约的状态，通常使用 `ref` 关键字按引用传递，以便能够修改合约的状态。我们将很快解释 `self` 及其类型。

## 2. 公共函数 (Public Functions)

如前所述，公共函数可以从合约外部访问。它们通常定义在用 `#[abi(embed_v0)]` 属性注释的实现块内，但也可能独立定义在 `#[external(v0)]` 属性下。

`#[abi(embed_v0)]` 属性意味着嵌入在其中的所有函数都是合约 Starknet 接口的实现，因此是潜在的入口点。

用 `#[abi(embed_v0)]` 属性注释 impl 块只会影响它包含的函数的可见性（即，公开 vs 私有/内部），但它不会告知我们这些函数修改合约状态的能力。

```cairo,noplayground
    // Public functions inside an impl block
    #[abi(embed_v0)]
    impl NameRegistry of super::INameRegistry<ContractState> {
        //ANCHOR: external
        fn store_name(ref self: ContractState, name: felt252) {
            let caller = get_caller_address();
            self._store_name(caller, name);
        }
        //ANCHOR_END: external

        //ANCHOR: view
        fn get_name(self: @ContractState, address: ContractAddress) -> felt252 {
            //ANCHOR: read
            self.names.entry(address).read()
            //ANCHOR_END: read
        }
        //ANCHOR_END: view
    }
```

> 与 `constructor` 函数类似，所有公共函数，无论是用 `#[external(v0)]` 注释的独立函数，还是用 `#[abi(embed_v0)]` 属性注释的 impl 块内的函数，**必须** 将 `self` 作为第一个参数。私有函数则不是这种情况。

### 外部函数 (External Functions)

外部函数是 _公共_ 函数，其中 `self: ContractState` 参数通过 `ref` 关键字按引用传递，这同时暴露了对存储变量的 `read` 和 `write` 访问权限。这允许直接通过 `self` 修改合约的状态。

```cairo,noplayground
        fn store_name(ref self: ContractState, name: felt252) {
            let caller = get_caller_address();
            self._store_name(caller, name);
        }
```

### 视图函数 (View Functions)

视图函数是 _公共_ 函数，其中 `self: ContractState` 参数作为快照传递，这只允许对存储变量的 `read` 访问，并通过导致编译错误来限制通过 `self` 对存储进行的写入。编译器会将其 _state_mutability_ 标记为 `view`，防止通过 `self` 直接进行任何状态修改。

```cairo,noplayground
        fn get_name(self: @ContractState, address: ContractAddress) -> felt252 {
            //ANCHOR: read
            self.names.entry(address).read()
            //ANCHOR_END: read
        }
```

### 公共函数的状态可变性

然而，正如你可能已经注意到的，作为快照传递 `self` 仅限制了编译时通过 `self` 的存储写入访问。它并不阻止通过直接系统调用进行状态修改，也不阻止调用另一个会修改状态的合约。

视图函数的只读属性并未在 Starknet 上强制执行，发送针对视图函数的交易 _由于_ 可以改变状态。

<!-- TODO: add an example of a view function that could modify the state using low-level syscalls -->

总之，尽管外部函数和视图函数由 Cairo 编译器区分，但 **所有公共函数** 都可以通过 invoke 交易调用，并且有可能修改 Starknet 状态。此外，所有公共函数都可以使用 `starknet_call` RPC 方法调用，这将不会创建交易，因此不会改变状态。

> **警告：** 这与 EVM 不同，在 EVM 中提供了 `staticcall` 操作码，它可以防止当前上下文和子上下文中的存储修改。因此，开发人员 **不应** 假设在另一个合约上调用视图函数不会修改状态。

### 独立公共函数

也可以使用 `#[external(v0)]` 属性在 trait 实现之外定义公共函数。这样做会自动在合约 ABI 中生成一个条目，允许任何人从外部调用这些独立的公共函数。这些函数也可以像 Starknet 合约中的任何函数一样从合约内部调用。第一个参数必须是 `self`。

在这里，我们在 impl 块之外定义了一个独立的 `get_contract_name` 函数：

```cairo,noplayground
    // Standalone public function
    #[external(v0)]
    fn get_contract_name(self: @ContractState) -> felt252 {
        'Name Registry'
    }
```

## 3. 私有函数 (Private Functions)

没有使用 `#[external(v0)]` 属性定义的函数，或者不在用 `#[abi(embed_v0)]` 属性注释的块内的函数是私有函数（也称为内部函数）。它们只能从合约内部调用。

它们可以分组在一个专用的 impl 块中（例如，在组件中，以便在嵌入合约中一次性轻松导入内部函数），或者只是作为自由函数添加到合约模块中。注意这 2 种方法是等效的。只需选择使你的代码更具可读性和易于使用的一种即可。

```cairo,noplayground
    // ANCHOR: generate_trait
    // Could be a group of functions about a same topic
    #[generate_trait]
    impl InternalFunctions of InternalFunctionsTrait {
        fn _store_name(ref self: ContractState, user: ContractAddress, name: felt252) {
            let total_names = self.total_names.read();

            //ANCHOR: write
            self.names.entry(user).write(name);
            //ANCHOR_END: write

            self.total_names.write(total_names + 1);
        }
    }
    // ANCHOR_END: generate_trait

    // Free function
    fn get_total_names_storage_address(self: @ContractState) -> felt252 {
        //ANCHOR: total_names_address
        self.total_names.__base_address__
        //ANCHOR_END:total_names_address
    }
```

> 等等，这个 `#[generate_trait]` 属性是什么？这个实现的 trait 定义在哪里？嗯，`#[generate_trait]` 属性是一个特殊的属性，它告诉编译器为实现块生成 trait 定义。这允许你摆脱定义带有泛型参数的 trait 并为实现块实现它的样板代码。有了这个属性，我们可以简单地直接定义实现块，无需任何泛型参数，并在我们的函数中使用 `self: ContractState`。

`#[generate_trait]` 属性主要用于定义私有 impl 块。它也可以与 `#[abi(per_item)]` 一起使用来定义合约的各种入口点（见 [下一节][abi per item section]）。

> 注意：对于公共 impl 块，不建议在 `#[abi(embed_v0)]` 属性之外使用 `#[generate_trait]`，因为这将导致无法生成相应的 ABI。只有当公共函数被定义在同时用 `#[abi(per_item)]` 属性注释的 impl 块中时，该块才应用 `#[generate_trait]` 注释。

[abi per item section]: ./ch101-02-contract-functions.md#4-abiper_item-attribute

## `[abi(per_item)]` 属性

你也可以在 impl 块内使用 `#[abi(per_item)]` 属性单独定义函数的入口点类型。它通常与 `#[generate_trait]` 属性一起使用，因为它允许你在没有显式接口的情况下定义入口点。在这种情况下，函数不会在 ABI 中归类在 impl 下。注意，当使用 `#[abi(per_item)]` 属性时，公共函数需要用 `#[external(v0)]` 属性注释 - 否则，它们将不会被公开，并将被视为私有函数。

这是一个简短的例子：

```cairo,noplayground
#[starknet::contract]
mod ContractExample {
    #[storage]
    struct Storage {}

    #[abi(per_item)]
    #[generate_trait]
    impl SomeImpl of SomeTrait {
        #[constructor]
        // this is a constructor function
        fn constructor(ref self: ContractState) {}

        #[external(v0)]
        // this is a public function
        fn external_function(ref self: ContractState, arg1: felt252) {}

        #[l1_handler]
        // this is a l1_handler function
        fn handle_message(ref self: ContractState, from_address: felt252, arg: felt252) {}

        // this is an internal function
        fn internal_function(self: @ContractState) {}
    }
}
```

在不使用 `#[generate_trait]` 而使用 `#[abi(per_item)]` 属性的情况下，只能在 trait 实现中包含 `constructor`、`l1-handler` 和 `internal` 函数。实际上，`#[abi(per_item)]` 仅适用于未定义为 Starknet 接口的 trait。因此，必须创建另一个定义为接口的 trait 来实现公共函数。
