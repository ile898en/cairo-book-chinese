# 附录 A - 系统调用

本章基于 Starknet 文档，可在此处获取：
[Starknet Docs](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/system-calls-cairo1/)。

编写智能合约需要各种相关操作，例如调用另一个合约或访问合约的存储，而独立程序不需要这些操作。

Starknet 合约语言通过使用系统调用来支持这些操作。系统调用使合约能够请求 Starknet OS 的服务。你可以在函数中使用系统调用来获取依赖于 Starknet 更广泛状态的信息（否则无法访问），而不是函数作用域中出现的局部变量。

以下是 Cairo 1.0 中可用的系统调用列表：

- [get_block_hash](#get_block_hash)
- [get_execution_info](#get_execution_info)
- [call_contract](#call_contract)
- [deploy](#deploy)
- [emit_event](#emit_event)
- [library_call](#library_call)
- [send_message_to_L1](#send_message_to_l1)
- [get_class_hash_at](#get_class_hash_at)
- [replace_class](#replace_class)
- [storage_read](#storage_read)
- [storage_write](#storage_write)
- [keccak](#keccak)
- [sha256_process_block](#sha256_process_block)

## `get_block_hash`

#### 语法

```cairo,noplayground
pub extern fn get_block_hash_syscall(
    block_number: u64,
) -> SyscallResult<felt252> implicits(GasBuiltin, System) nopanic;
```

#### 描述

获取特定 Starknet 区块的哈希值，范围在 `[first_v0_12_0_block, current_block - 10]` 之间。

#### 返回值

返回给定区块的哈希值。

#### 错误信息

- `Block number out of range`: `block_number` 大于 _`current_block`_`- 10`。
- `0`: `block_number` 小于 v0.12.0 的第一个区块号。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/0c882679fdb24a818cad19f2c18decbf6ef66153/corelib/src/starknet/syscalls.cairo#L37)

## `get_execution_info`

#### 语法

```cairo,noplayground
pub extern fn get_execution_info_syscall() -> SyscallResult<
    Box<starknet::info::ExecutionInfo>,
> implicits(GasBuiltin, System) nopanic;
```

#### 描述

获取有关原始交易的信息。

在 Cairo 1.0 中，所有区块/交易/执行上下文的 getter 都被批处理到这一个系统调用中。

#### 参数

无。

#### 返回值

返回一个包含执行信息的 [结构体](https://github.com/starkware-libs/cairo/blob/efbf69d4e93a60faa6e1363fd0152b8fcedbb00a/corelib/src/starknet/info.cairo#L8)。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L35)

## `call_contract`

#### 语法

```cairo,noplayground
pub extern fn call_contract_syscall(
    address: ContractAddress, entry_point_selector: felt252, calldata: Span<felt252>,
) -> SyscallResult<Span<felt252>> implicits(GasBuiltin, System) nopanic;
```

#### 描述

调用给定的合约。此系统调用需要被调用合约的地址、该合约内函数的选择器以及调用参数。

> **注意：**
>
> 内部调用不能返回 Err(\_)，因为排序器和 Starknet OS 不处理这种情况。
>
> 如果 call_contract_syscall 失败，这无法被捕获，因此将导致整个交易被回滚。

#### 参数

- _`address`_: 你想要调用的合约地址。
- _`entry_point_selector`_: 该合约内函数的选择器，可以使用 `selector!` 宏计算。
- _`calldata`_: calldata 数组。

#### 返回值

调用响应，类型为 `SyscallResult<Span<felt252>>`。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L10)

> **注意：** 这被认为是调用合约的较低级语法。如果被调用合约的接口可用，那么你可以使用更直接的语法。

## `deploy`

#### 语法

```cairo,noplayground
pub extern fn deploy_syscall(
    class_hash: ClassHash,
    contract_address_salt: felt252,
    calldata: Span<felt252>,
    deploy_from_zero: bool,
) -> SyscallResult<(ContractAddress, Span<felt252>)> implicits(GasBuiltin, System) nopanic;
```

#### 描述

部署先前声明的类的新实例。

#### 参数

- _`class_hash`_: 要部署的合约的类哈希。
- _`contract_address_salt`_: 盐，发送者提供的任意值。它用于计算合约地址。
- _`calldata`_: 构造函数的 calldata。一个 field 数组。
- _`deploy_from_zero`_: 用于合约地址计算的标志。如果未设置，则调用者地址将用作新合约的部署者地址，否则使用 0。

#### 返回值

一个包含在 SyscallResult 中的元组，其中：

- 第一个元素是已部署合约的地址，类型为 `ContractAddress`。

- 第二个元素是合约构造函数的响应数组，类型为 `Span::<felt252>`。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/4821865770ac9e57442aef6f0ce82edc7020a4d6/corelib/src/starknet/syscalls.cairo#L22)

## `emit_event`

#### 语法

```cairo,noplayground
pub extern fn emit_event_syscall(
    keys: Span<felt252>, data: Span<felt252>,
) -> SyscallResult<()> implicits(GasBuiltin, System) nopanic;
```

#### 描述

发出具有给定键和数据的事件。

有关发出事件的更多信息和更高级的语法，请参阅 [Starknet 事件](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/starknet-events/)。

#### 参数

- _`keys`_: 事件的键。这些类似于以太坊的事件 topics，你可以使用 starknet_getEvents 方法通过这些键进行过滤。

- _`data`_: 事件的数据。

#### 返回值

无。

#### 示例

以下示发出一个具有两个键（字符串 `status` 和 `deposit`）和三个数据元素（`1`、`2` 和 `3`）的事件。

```cairo,noplayground
let keys = ArrayTrait::new();
keys.append('key');
keys.append('deposit');
let values = ArrayTrait::new();
values.append(1);
values.append(2);
values.append(3);
emit_event_syscall(keys, values).unwrap_syscall();
```

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L30)

## `library_call`

#### 语法

```cairo,noplayground
pub extern fn library_call_syscall(
    class_hash: ClassHash, function_selector: felt252, calldata: Span<felt252>,
) -> SyscallResult<Span<felt252>> implicits(GasBuiltin, System) nopanic;
```

#### 描述

调用任何先前声明的类中请求的函数。该类仅用于其逻辑。

此系统调用取代了以太坊中已知的 delegate call 功能，重要的区别在于只涉及一个合约。

#### 参数

- _`class_hash`_: 你想要使用的类的哈希。

- _`function_selector`_: 该类中函数的选择器，可以使用 `selector!` 宏计算。

- _`calldata`_: calldata。

#### 返回值

调用响应，类型为 `SyscallResult<Span<felt252>>`。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L43)

## `send_message_to_L1`

#### 语法

```cairo,noplayground
pub extern fn send_message_to_l1_syscall(
    to_address: felt252, payload: Span<felt252>,
) -> SyscallResult<()> implicits(GasBuiltin, System) nopanic;
```

#### 描述

向 L1 发送消息。

此系统调用将消息参数包含为证明输出的一部分，并在收到状态更新（包括交易）后将这些参数公开给 L1 上的 `StarknetCore` 合约。

有关更多信息，请参阅 Starknet 的 [消息传递机制](https://docs.starknet.io/documentation/architecture_and_concepts/Network_Architecture/messaging-mechanism/)。

#### 参数

- _`to_address`_: 接收者的 L1 地址。

- _`payload`_: 包含消息有效负载的数组。

#### 返回值

无。

#### 示例

以下示例向地址为 `3423542542364363` 的 L1 合约发送内容为 `(1,2)` 的消息。

```cairo,noplayground
let payload = ArrayTrait::new();
payload.append(1);
payload.append(2);
send_message_to_l1_syscall(payload).unwrap_syscall();
```

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L51)

## `get_class_hash_at`

#### 语法

```cairo,noplayground
pub extern fn get_class_hash_at_syscall(
    contract_address: ContractAddress,
) -> SyscallResult<ClassHash> implicits(GasBuiltin, System) nopanic;
```

#### 描述

获取给定地址的合约的类哈希。

#### 参数

- _`contract_address`_: 已部署合约的地址。

#### 返回值

合约源代码的类哈希。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/67c6eff9c276d11bd1cc903d7a3981d8d0eb2fa2/corelib/src/starknet/syscalls.cairo#L99)

## `replace_class`

#### 语法

```cairo,noplayground
pub extern fn replace_class_syscall(
    class_hash: ClassHash,
) -> SyscallResult<()> implicits(GasBuiltin, System) nopanic;
```

#### 描述

一旦调用 `replace_class`，调用合约（即调用系统调用时 `get_contract_address` 返回其地址的合约）的类将被 class_hash 参数给出的类替换。

> **注意：**
>
> 调用 `replace_class` 后，当前从旧类执行的代码将完成运行。
>
> 新类将从下一个交易开始使用，或者如果在同一交易中（替换后）通过 `call_contract` 系统调用调用该合约。

#### 参数

- _`class_hash`_: 你想要用作替换的类的哈希。

#### 返回值

无。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L77)

## `storage_read`

#### 语法

```cairo,noplayground
pub extern fn storage_read_syscall(
    address_domain: u32, address: StorageAddress,
) -> SyscallResult<felt252> implicits(GasBuiltin, System) nopanic;
```

#### 描述

获取调用合约存储中键的值。

与能够读取显式定义在合约中的存储变量的 `var.read()` 相比，此系统调用提供了对应存储中任何可能键的直接访问。

有关使用存储变量访问存储的信息，请参阅 [存储变量](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/contract-storage/#storage_variables)。

#### 参数

- _`address_domain`_: 键的域，用于区分不同的数据可用性模式。此分离用于在 Starknet 中提供不同的数据可用性模式。目前，仅支持由域 `0` 指示的链上模式（所有更新都转到 L1）。将来引入的其他地址域在发布方面将有不同的行为（特别是，它们不会发布在 L1 上，从而在成本和安全性之间进行权衡）。

- _`address`_: 请求的存储地址。

#### 返回值

键的值，类型为 `SyscallResult<felt252>`。

#### 示例

```cairo,noplayground
use starknet::storage_access::storage_base_address_from_felt252;

...

let storage_address = storage_base_address_from_felt252(3534535754756246375475423547453)
storage_read_syscall(0, storage_address).unwrap_syscall()
```

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L60)

## `storage_write`

#### 语法

```cairo,noplayground
pub extern fn storage_write_syscall(
    address_domain: u32, address: StorageAddress, value: felt252,
) -> SyscallResult<()> implicits(GasBuiltin, System) nopanic;
```

#### 描述

设置调用合约存储中键的值。

与能够写入显式定义在合约中的存储变量的 `var.write()` 相比，此系统调用提供了对应存储中任何可能键的直接访问。

有关使用存储变量访问存储的信息，请参阅 [存储变量](https://docs.starknet.io/documentation/architecture_and_concepts/Smart_Contracts/contract-storage/#storage_variables)。

#### 参数

- _`address_domain`_: 键的域，用于区分不同的数据可用性模式。此分离用于在 Starknet 中提供不同的数据可用性模式。目前，仅支持由域 `0` 指示的链上模式（所有更新都转到 L1）。将来引入的其他地址域在发布方面将有不同的行为（特别是，它们不会发布在 L1 上，从而在成本和安全性之间进行权衡）。

- _`address`_: 请求的存储地址。

- _`value`_: 要写入键的值。

#### 返回值

无。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/cca08c898f0eb3e58797674f20994df0ba641983/corelib/src/starknet/syscalls.cairo#L70)

## `keccak`

#### 语法

```cairo,noplayground
pub extern fn keccak_syscall(
    input: Span<u64>,
) -> SyscallResult<u256> implicits(GasBuiltin, System) nopanic;
```

#### 描述

计算给定输入的 Keccak-256 哈希。

#### 参数

- _`input`_: 一个 `Span<u64>` Keccak-256 输入。

#### 返回值

返回哈希结果，类型为 `u256`。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/67c6eff9c276d11bd1cc903d7a3981d8d0eb2fa2/corelib/src/starknet/syscalls.cairo#L107)

## `sha256_process_block`

#### 语法

```cairo,noplayground
pub extern fn sha256_process_block_syscall(
    state: core::sha256::Sha256StateHandle, input: Box<[u32; 16]>
) -> SyscallResult<core::sha256::Sha256StateHandle> implicits(GasBuiltin, System) nopanic;
```

#### 描述

使用给定状态计算输入的下一个 SHA-256 状态。

此系统调用通过将当前 `state` 与 512 位 `input` 数据块组合来计算下一个 SHA-256 状态。

#### 参数

- _`state`_: 当前 SHA-256 状态。
- _`input`_: 要处理成 SHA-256 的值。

#### 返回值

返回 `input` 数据的新 SHA-256 状态。

#### 公共库

- [syscalls.cairo](https://github.com/starkware-libs/cairo/blob/3540731e5b0e78f2f5b1a51d3611418121c19e54/corelib/src/starknet/syscalls.cairo#L106)
