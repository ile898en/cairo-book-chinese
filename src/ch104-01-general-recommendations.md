# 一般建议

到目前为止，我们一直专注于学习如何编写 Cairo 代码，这是让你的程序焕发生机的最低要求；但编写 _安全_ 的代码同样重要。本章提炼自大量真实的 Cairo/Starknet 审计资料，编译成具体的说明，你可以在编码、测试和审查合约时使用。

我们将重点关注：

- 访问控制和升级
- 安全的 ERC20 代币集成
- 可能导致漏洞的 Cairo 特定陷阱
- 跨域/桥接安全
- Starknet 上的经济/DoS 必备知识

## 访问控制、升级和初始化器

Starknet 审计中最常见的关键问题仍然是“谁可以调用此函数？”和“此函数可以（重新）初始化吗？”。Cairo 拥有很棒的简单构建块，你应该重用它们的逻辑来专注于程序的核心安全方面。

### 控制你的特权路径

始终确保升级只能由授权角色完成。如果未经授权的用户可以升级你的合约，它可以将类替换为任何东西，并获得对合约的完全控制权。这同样适用于暂停/恢复函数、桥接处理程序（谁可以从 L1 调用此合约）和元执行 (meta-execution)。所有这些关键函数都应使用 OpenZeppelin 的 OwnableComponent 进行保护。

```cairo, noplayground
// components
component!(path: OwnableComponent, storage: ownable);
component!(path: UpgradeableComponent, storage: upgradeable, event: UpgradeableEvent);

#[abi(embed_v0)]
impl OwnableImpl = OwnableComponent::OwnableImpl<ContractState>;
impl InternalUpgradeableImpl = UpgradeableComponent::InternalImpl<ContractState>;

#[event]
fn Upgraded(new_class_hash: felt252) {}

fn upgrade(ref self: ContractState, new_class_hash: ClassHash) {
    self.ownable.assert_only_owner();
    self.upgradeable._upgrade(new_class_hash);
    Upgraded(new_class_hash); // emit explicit upgrade event
}
```

**为什么要发出事件？** 事故响应和索引器依赖于它们。为升级、配置更改、暂停、清算和任何特权操作发出事件；包括地址（例如，代币）以消除歧义。

### 初始化器应该只被调用一次

一个常见的漏洞向量是公开暴露的初始化器，可以在部署后调用。初始化器的目的是解耦部署和合约的初始化。但是，如果初始化器可以被多次调用，它可能会产生意想不到的后果。确保行为是幂等的。

```cairo, noplayground
#[storage]
struct Storage {
    _initialized: u8,
    // ...
}

fn initializer(ref self: ContractState, owner: ContractAddress) {
    assert!(self._initialized.read() == 0, "ALREADY_INIT");
    self._initialized.write(1);
    self.ownable.initialize(owner);
    // init the rest…
}
```

> 规则：如果在部署期间它 _必须_ 是外部的，请确保它只能被调用一次；如果它不需要是外部的，请将其保留为内部。

## 代币集成

### 始终检查布尔返回值

虽然 OpenZeppelin ERC20 实现会在失败时 revert，但这并非所有 ERC-20 实现都会这样做。有些可能会返回 `false` 而不 panic。`transfer` 和 `transfer_from` 返回布尔标志；验证它们以确保转账成功。

### CamelCase / snake_case 双接口

Starknet 上的大多数 ERC20 代币应该使用 `snake_case` 命名风格。然而，由于遗留原因，一些旧的 ERC20 代币具有 `camelCase` 入口点，如果你合约调用它们期望找到 `snake_case`，这可能会导致问题。处理这两种命名风格很麻烦；但你应该至少确保你将与之交互的大多数代币使用 `snake_case` 命名风格，或者调整你的合约。

## Cairo 特定陷阱

Cairo 语言本身并没有非常复杂的语义可能引入漏洞，但有一些常规的编程模式可能导致不需要的行为。

### 表达式中的运算符优先级

在 Cairo 中，`&&` 的优先级高于 `||`。确保组合表达式被正确地用括号括起来，以强制运算符之间的优先级。

```cairo, noplayground
// ❌ buggy: ctx.coll_ok and ctx.debt_ok are only required in Recovery
assert!(
    mode == Mode::None || mode == Mode::Recovery && ctx.coll_ok && ctx.debt_ok,
    "EMERGENCY_MODE"
);

// ✅ fixed
assert!(
    (mode == Mode::None || mode == Mode::Recovery) && (ctx.coll_ok && ctx.debt_ok),
    "EMERGENCY_MODE"
);
```

### 无符号循环下溢

使用 `u32` 作为循环计数器可能会导致下溢 panic，如果该计数器减至 0 以下。如果计数器应该处理负值，请改用 `i32`。

```cairo, noplayground
// ✅ prefer signed counters or explicit break
let mut i: i32 = (n.try_into().unwrap()) - 1;
while i >= 0 { // This would never trigger if `i` was a u32.
    // ...
    i -= 1;
}
```

### 位打包到 `felt252`

将多个字段打包到一个 `felt252` 中非常适合优化，但如果没有严格的边界，这也常见且危险。在将字段打包到 `felt252` 之前，请确检查字段的范围。值得注意的是，打包值的大小的总和不应超过 251 位。

```cairo, noplayground
fn pack_order(book_id: u256, tick_u24: u256, index_u40: u256) -> felt252 {
    // width checks
    assert!(book_id < (1_u256 * POW_2_187), "BOOK_OVER");
    assert!(tick_u24 < (1_u256 * POW_2_24),  "TICK_OVER");
    assert!(index_u40 < (1_u256 * POW_2_40), "INDEX_OVER");

    let packed: u256 =
        (book_id * POW_2_64) + (tick_u24 * POW_2_40) + index_u40;
    packed.try_into().expect("PACK_OVERFLOW")
}
```

<span class="caption"> 如果值太大，位打包可能会失败。</span>

### `deploy_syscall(deploy_from_zero=true)` 冲突

从零开始的确定性部署可能会导致冲突，如果尝试使用相同的 calldata 部署两个合约。确保将 `deploy_from_zero` 设置为 `false`，除非你确定要从零开始部署。

### 不要检查 `get_caller_address().is_zero()`

从 Solidity 继承而来的是零地址检查。在 Starknet 上，`get_caller_address()` 永远不零地址。因此，这些检查是无用的。

## 跨域 / 桥接安全

L1-L2 交互是 Starknet 工作方式所特有的，并且可能是错误的来源。

### L1 处理程序必须验证调用者地址

`#[l1_handler]` 属性将入口点标记为可从 L1 上的合约调用。在大多数情况下，你会希望确保该调用的来源是受信任的 L1 合约 —— 因此，你应该验证调用者地址。

```cairo, noplayground
#[l1_handler]
fn handle_deposit(
    ref self: ContractState,
    from_address: ContractAddress,
    account: ContractAddress,
    amount: u256
) {
    let l1_bridge = self._l1_bridge.read();
    assert!(!l1_bridge.is_zero(), 'UNINIT_BRIDGE');
    assert!(from_address == l1_bridge, 'ONLY_L1_BRIDGE');
    // credit account…
}
```

## 经济/DoS 和恶意破坏 (Griefing)

### 无界循环

用户控制的迭代（领取、批量提款、订单扫描）可能会超过 Starknet 步骤限制。确保限制迭代次数和/或使用分页模式将工作拆分多个交易。

值得注意的是，想象你正在实现一个系统，该系统被调用时，函数将遍历存储中的项目列表并处理它们。如果列表没有边界，攻击者可以增加该列表中的项目数量，导致函数永远不会终止，因为它将达到 Starknet 的执行步骤限制。

在这种情况下，合约就会变砖：_任何人_ 都无法再与之交互，因为任何交互都会触发步骤限制。

为了绕过这个问题，例如你可以使用分页模式，其中函数一次处理最大数量的项目，并将下一个游标返回给调用者。然后调用者可以使用下一个游标再次调用该函数以处理下一批项目。

```cairo, noplayground
fn claim_withdrawals(ref self: ContractState, start: u64, max: u64) -> u64 {
    let mut i = start;
    let end = core::cmp::min(self.pending_count.read(), start + max);
    while i < end {
        self._process(i);
        i += 1;
    }
    end // next cursor
}
```
