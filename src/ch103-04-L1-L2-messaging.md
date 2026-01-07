# L1-L2 消息传递

Layer 2 的一个关键特性是其与 Layer 1 交互的能力。

Starknet 拥有自己的 `L1-L2` 消息传递系统，该系统与其共识机制和 L1 上状态更新的提交不同。消息传递是 L1 上的智能合约与 L2 上的智能合约（反之亦然）交互的一种方式，允许我们进行“跨链”交易。例如，我们可以在一条链上进行一些计算，并在另一条链上使用该计算的结果。

Starknet 上的桥接器都使用 `L1-L2` 消息传递。假设你想将代币从以太坊桥接到 Starknet。你只需将代币存入 L1 桥接合约，该合约将自动触发在 L2 上铸造相同的代币。`L1-L2` 消息传递的另一个好用例是 [DeFi 池 (DeFi pooling)][defi pooling doc]。

在 Starknet 上，重要的是要注意消息传递系统是 **异步** 和 **不对称** 的。

- **异步**：这意味着在你的合约代码中（无论是 Solidity 还是 Cairo），你无法在合约代码执行中等待另一条链上发送消息的结果。
- **不对称**：从以太坊向 Starknet 发送消息 (`L1->L2`) 是由 Starknet 定序器完全自动化的，这意味着消息会自动传递到 L2 上的目标合约。但是，当从 Starknet 向以太坊发送消息 (`L2->L1`) 时，只有消息的哈希值由 Starknet 定序器发送到 L1。然后，你必须通过 L1 上的交易手动消费该消息。

让我们深入了解细节。

[defi pooling doc]: https://starkware.co/resource/defi-pooling/

## StarknetMessaging 合约

`L1-L2` 消息传递系统的关键组件是 [`StarknetCore`][starknetcore etherscan] 合约。它是部署在以太坊上的一组 Solidity 合约，允许 Starknet 正常运行。`StarknetCore` 的合约之一名为 `StarknetMessaging`，它是负责在 Starknet 和 Ethereum 之间传递消息的合约。`StarknetMessaging` 遵循一个 [接口][IStarknetMessaging]，该接口具有允许向 L2 发送消息、在 L1 上接收来自 L2 的消息以及取消消息的功能。

```js
interface IStarknetMessaging is IStarknetMessagingEvents {

    function sendMessageToL2(
        uint256 toAddress,
        uint256 selector,
        uint256[] calldata payload
    ) external returns (bytes32);

    function consumeMessageFromL2(uint256 fromAddress, uint256[] calldata payload)
        external
        returns (bytes32);

    function startL1ToL2MessageCancellation(
        uint256 toAddress,
        uint256 selector,
        uint256[] calldata payload,
        uint256 nonce
    ) external;

    function cancelL1ToL2Message(
        uint256 toAddress,
        uint256 selector,
        uint256[] calldata payload,
        uint256 nonce
    ) external;
}
```

<span class="caption"> Starknet 消息传递合约接口</span>

在 `L1->L2` 消息的情况下，Starknet 定序器不断监听以太坊上 `StarknetMessaging` 合约发出的日志。一旦在日志中检测到消息，定序器就会准备并执行 `L1HandlerTransaction` 以调用目标 L2 合约上的函数。这需要 1-2 分钟才能完成（以太坊区块被挖掘需要几秒钟，然后定序器必须构建并执行交易）。

`L2->L1` 消息是由 L2 上的合约执行准备的，并且是生成的区块的一部分。确定序器生成一个区块时，它将合约执行准备的每条消息的哈希发送到 L1 上的 `StarknetCore` 合约，一旦它们所属的区块在以太坊上得到证明和验证（目前大约需要 3-4 小时），就可以在那里消费它们。

[starknetcore etherscan]:
  https://etherscan.io/address/0xc662c410C0ECf747543f5bA90660f6ABeBD9C8c4
[IStarknetMessaging]:
  https://github.com/starkware-libs/cairo-lang/blob/4e233516f52477ad158bc81a86ec2760471c1b65/src/starkware/starknet/eth/IStarknetMessaging.sol#L6

## 从以太坊向 Starknet 发送消息

如果你想从以太坊向 Starknet 发送消息，你的 Solidity 合约必须调用 `StarknetMessaging` 合约的 `sendMessageToL2` 函数。要在 Starknet 上接收这些消息，你需要用 `#[l1_handler]` 属性注释可以从 L1 调用的函数。

让我们来看一个取自 [本教程][messaging contract] 的简单合约，我们要向 Starknet 发送一条消息。`_snMessaging` 是一个状态变量，已经用 `StarknetMessaging` 合约的地址初始化。你可以 [在这里][starknet addresses] 查看所有 Starknet 合约和定序器地址。

```js
// Sends a message on Starknet with a single felt.
function sendMessageFelt(
    uint256 contractAddress,
    uint256 selector,
    uint256 myFelt
)
    external
    payable
{
    // We "serialize" here the felt into a payload, which is an array of uint256.
    uint256[] memory payload = new uint256[](1);
    payload[0] = myFelt;

    // msg.value must always be >= 20_000 wei.
    _snMessaging.sendMessageToL2{value: msg.value}(
        contractAddress,
        selector,
        payload
    );
}
```

该函数向 `StarknetMessaging` 合约发送一条包含单个 felt 值的消息。请注意，你的 Cairo 合约只能理解 `felt252` 数据类型，因此如果你想发送更复杂的数据，你必须确保将数据序列化为 `uint256` 数组遵循 Cairo 序列化方案。

重要的是要注意我们有 `{value: msg.value}`。事实上，我们这里必须发送的最小值为 `20k wei`，这是因为 `StarknetMessaging` 合约将在以太坊的存储中注册我们消息的哈希。

除了这 `20k wei` 之外，由于定序器执行的 `L1HandlerTransaction` 不绑定到任何账户（消息源自 L1），你还必须确保在 L1 上支付足够的费用，以便你的消息在 L2 上被反序列化和处理。

`L1HandlerTransaction` 的费用以常规方式计算，就像 `Invoke` 交易一样。为此，你可以使用 `starkli` 或 `snforge` 分析 gas 消耗，以估算消息执行的成本。

`sendMessageToL2` 的签名是：

```js
function sendMessageToL2(
        uint256 toAddress,
        uint256 selector,
        uint256[] calldata payload
    ) external override returns (bytes32);
```

参数如下：

- `toAddress`：将被调用的 L2 上的合约地址。
- `selector`：此合约在 `toAddress` 处的函数的选择器。此选择器（函数）必须具有 `#[l1_handler]` 属性才能被调用。
- `payload`：有效负载始终是 `felt252` 数组（在 Solidity 中用 `uint256` 表示）。由于这个原因，我们将输入 `myFelt` 插入到数组中。这就是为什么我们需要将输入数据插入到数组中的原因。

在 Starknet 端，为了接收这条消息，我们有：

```cairo,noplayground
    #[l1_handler]
    fn msg_handler_felt(ref self: ContractState, from_address: felt252, my_felt: felt252) {
        assert!(from_address == self.allowed_message_sender.read(), "Invalid message sender");

        // You can now use the data, automatically deserialized from the message payload.
        assert!(my_felt == 123, "Invalid value");
    }
```

我们需要将 `#[l1_handler]` 属性添加到我们的函数中。L1 处理程序是只能由 `L1HandlerTransaction` 执行的特殊函数。接收来自 L1 的交易没有什么特别要做的，因为消息由定序器自动中继。在你的 `#[l1_handler]` 函数中，验证 L1 消息的发送者非常重要，以确保我们的合约只能接收来自受信任的 L1 合约的消息。

[messaging contract]:
  https://github.com/glihm/starknet-messaging-dev/blob/main/solidity/src/ContractMsg.sol
[starknet addresses]:
  https://docs.starknet.io/documentation/tools/important_addresses/

## 从 Starknet 向以太坊发送消息

当从 Starknet 向以太坊发送消息时，你必须在你的 Cairo 合约中使用 `send_message_to_l1` 系统调用。此系统调用允许你向 L1 上的 `StarknetMessaging` 合约发送消息。与 `L1->L2` 消息不同，`L2->L1` 消息必须手动消费，这意味着你需要你的 Solidity 合约显式调用 `StarknetMessaging` 合约的 `consumeMessageFromL2` 函数才能消费该消息。

要将消息从 L2 发送到 L1，我们在 Starknet 上要做的是：

```cairo,noplayground
        fn send_message_felt(ref self: ContractState, to_address: EthAddress, my_felt: felt252) {
            // Note here, we "serialize" my_felt, as the payload must be
            // a `Span<felt252>`.
            syscalls::send_message_to_l1_syscall(to_address.into(), array![my_felt].span())
                .unwrap();
        }
```

我们只需构建有效负载并将其与 L1 合约地址一起传递给系统调用函数。

在 L1 上，重要的部分是构建 L2 发送的相同有效负载。然后在你的 Solidity 合约中，你可以通过传递 L2 合约地址和有效负载来调用 `consumeMessageFromL2`。请注意，`consumeMessageFromL2` 期望的 L2 合约地址是通过调用 `send_message_to_l1_syscall` 在 L2 上发送消息的合约地址。

```js
function consumeMessageFelt(
    uint256 fromAddress,
    uint256[] calldata payload
)
    external
{
    let messageHash = _snMessaging.consumeMessageFromL2(fromAddress, payload);

    // You can use the message hash if you want here.

    // We expect the payload to contain only a felt252 value (which is a uint256 in Solidity).
    require(payload.length == 1, "Invalid payload");

    uint256 my_felt = payload[0];

    // From here, you can safely use `my_felt` as the message has been verified by StarknetMessaging.
    require(my_felt > 0, "Invalid value");
}
```

正如你所看到的，在这个上下文中，我们不必验证哪个 L2 合约正在发送消息（就像我们在 L2 上验证哪个 L1 合约正在发送消息一样）。但我们实际上正在使用 `StarknetCore` 合约的 `consumeMessageFromL2` 来验证输入（L2 上的合约地址和有效负载），以确保我们只消费有效的消息。

> **注意：** `StarknetCore` 合约的 `consumeMessageFromL2` 函数应该从 Solidity 合约中调用，而不是直接在 `StarknetCore` 合约上调用。其原因是因为 `StarknetCore` 合约正在使用 `msg.sender` 来实际计算消息的哈希值。而这个 `msg.sender` 必须对应于在 Starknet 上调用的 `send_message_to_l1_syscall` 函数中给出的 `to_address` 字段。

## Cairo 序列化

在 L1 和 L2 之间发送消息之前，你必须记住，用 Cairo 编写的 Starknet 合约只能理解序列化数据。序列化数据始终是 `felt252` 数组。在 Solidity 中，我们有 `uint256` 类型，而 `felt252` 比 `uint256` 小大约 4 位。所以我们要特别注意我们正在发送的消息的有效负载中包含的值。如果在 L1 上，我们构建了一条包含超过最大 `felt252` 的值的消息，则该消息将被卡住，永远无法在 L2 上消费。

例如，Cairo 中的实际 `uint256` 值由如下结构体表示：

```cairo,does_not_compile
struct u256 {
    low: u128,
    high: u128,
}
```

它将被序列化为 **两个** felt，一个用于 `low`，一个用于 `high`。这意味着要向 Cairo 仅发送一个 `u256`，你需要从 L1 发送具有 **两个** 值的有效负载。

```js
uint256[] memory payload = new uint256[](2);
// Let's send the value 1 as a u256 in cairo: low = 1, high = 0.
payload[0] = 1;
payload[1] = 0;
```

如果你想了解有关消息传递机制的更多信息，可以访问 [Starknet 文档][starknet messaging doc]。

你也可以 [在这里找到详细指南][glihm messaging guide] 以在本地测试消息传递系统。

[starknet messaging doc]:
  https://docs.starknet.io/documentation/architecture_and_concepts/Network_Architecture/messaging-mechanism/
[glihm messaging guide]: https://github.com/glihm/starknet-messaging-dev
