# 部署投票合约并与之交互

Starknet 中的 **`Vote`** 合约首先通过合约的构造函数注册选民。在这个阶段初始化了三个选民，并将他们的地址传递给内部函数 **`_register_voters`**。此函数将选民添加到合约的状态中，标记他们已注册并有资格投票。

在合约中，定义了常量 **`YES`** 和 **`NO`** 来代表投票选项（分别为 1 和 0）。这些常量通过标准化输入值来促进投票过程。

一旦注册，选民就可以使用 **`vote`** 函数投票，选择 1 (YES) 或 0 (NO) 作为他们的投票。投票时，合约的状态会更新，记录投票并标记选民已投票。这确保了选民无法要在同一提案中再次投票。投票触发 **`VoteCast`** 事件，记录该操作。

合约还监控未经授权的投票尝试。如果检测到未经授权的操作，例如未注册的用户试图投票或用户试图再次投票，则会发出 **`UnauthorizedAttempt`** 事件。

这些函数、状态、常量和事件共同创建了一个结构化的投票系统，在 Starknet 环境中管理从注册到投、事件记录和结果检索的投票生命周期。像 **`YES`** 和 **`NO`** 这样的常量有助于简化投票过程，而事件在确保透明度和可追溯性方面起着至关重要的作用。

清单 {{#ref voting_contract}} 详细显示了 `Vote` 合约：

```cairo,noplayground
/// Core Library Imports for the Traits outside the Starknet Contract
use starknet::ContractAddress;

/// Trait defining the functions that can be implemented or called by the Starknet Contract
#[starknet::interface]
trait VoteTrait<T> {
    /// Returns the current vote status
    fn get_vote_status(self: @T) -> (u8, u8, u8, u8);
    /// Checks if the user at the specified address is allowed to vote
    fn voter_can_vote(self: @T, user_address: ContractAddress) -> bool;
    /// Checks if the specified address is registered as a voter
    fn is_voter_registered(self: @T, address: ContractAddress) -> bool;
    /// Allows a user to vote
    fn vote(ref self: T, vote: u8);
}

/// Starknet Contract allowing three registered voters to vote on a proposal
#[starknet::contract]
mod Vote {
    use starknet::storage::{
        Map, StorageMapReadAccess, StorageMapWriteAccess, StoragePointerReadAccess,
        StoragePointerWriteAccess,
    };
    use starknet::{ContractAddress, get_caller_address};

    const YES: u8 = 1_u8;
    const NO: u8 = 0_u8;

    #[storage]
    struct Storage {
        yes_votes: u8,
        no_votes: u8,
        can_vote: Map<ContractAddress, bool>,
        registered_voter: Map<ContractAddress, bool>,
    }

    #[constructor]
    fn constructor(
        ref self: ContractState,
        voter_1: ContractAddress,
        voter_2: ContractAddress,
        voter_3: ContractAddress,
    ) {
        self._register_voters(voter_1, voter_2, voter_3);

        self.yes_votes.write(0_u8);
        self.no_votes.write(0_u8);
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        VoteCast: VoteCast,
        UnauthorizedAttempt: UnauthorizedAttempt,
    }

    #[derive(Drop, starknet::Event)]
    struct VoteCast {
        voter: ContractAddress,
        vote: u8,
    }

    #[derive(Drop, starknet::Event)]
    struct UnauthorizedAttempt {
        unauthorized_address: ContractAddress,
    }

    #[abi(embed_v0)]
    impl VoteImpl of super::VoteTrait<ContractState> {
        fn get_vote_status(self: @ContractState) -> (u8, u8, u8, u8) {
            let (n_yes, n_no) = self._get_voting_result();
            let (yes_percentage, no_percentage) = self._get_voting_result_in_percentage();
            (n_yes, n_no, yes_percentage, no_percentage)
        }

        fn voter_can_vote(self: @ContractState, user_address: ContractAddress) -> bool {
            self.can_vote.read(user_address)
        }

        fn is_voter_registered(self: @ContractState, address: ContractAddress) -> bool {
            self.registered_voter.read(address)
        }

        fn vote(ref self: ContractState, vote: u8) {
            assert!(vote == NO || vote == YES, "VOTE_0_OR_1");
            let caller: ContractAddress = get_caller_address();
            self._assert_allowed(caller);
            self.can_vote.write(caller, false);

            if (vote == NO) {
                self.no_votes.write(self.no_votes.read() + 1_u8);
            }
            if (vote == YES) {
                self.yes_votes.write(self.yes_votes.read() + 1_u8);
            }

            self.emit(VoteCast { voter: caller, vote: vote });
        }
    }

    #[generate_trait]
    impl InternalFunctions of InternalFunctionsTrait {
        fn _register_voters(
            ref self: ContractState,
            voter_1: ContractAddress,
            voter_2: ContractAddress,
            voter_3: ContractAddress,
        ) {
            self.registered_voter.write(voter_1, true);
            self.can_vote.write(voter_1, true);

            self.registered_voter.write(voter_2, true);
            self.can_vote.write(voter_2, true);

            self.registered_voter.write(voter_3, true);
            self.can_vote.write(voter_3, true);
        }
    }

    #[generate_trait]
    impl AssertsImpl of AssertsTrait {
        fn _assert_allowed(ref self: ContractState, address: ContractAddress) {
            let is_voter: bool = self.registered_voter.read((address));
            let can_vote: bool = self.can_vote.read((address));

            if (!can_vote) {
                self.emit(UnauthorizedAttempt { unauthorized_address: address });
            }

            assert!(is_voter, "USER_NOT_REGISTERED");
            assert!(can_vote, "USER_ALREADY_VOTED");
        }
    }

    #[generate_trait]
    impl VoteResultFunctionsImpl of VoteResultFunctionsTrait {
        fn _get_voting_result(self: @ContractState) -> (u8, u8) {
            let n_yes: u8 = self.yes_votes.read();
            let n_no: u8 = self.no_votes.read();

            (n_yes, n_no)
        }

        fn _get_voting_result_in_percentage(self: @ContractState) -> (u8, u8) {
            let n_yes: u8 = self.yes_votes.read();
            let n_no: u8 = self.no_votes.read();

            let total_votes: u8 = n_yes + n_no;

            if (total_votes == 0_u8) {
                return (0, 0);
            }
            let yes_percentage: u8 = (n_yes * 100_u8) / (total_votes);
            let no_percentage: u8 = (n_no * 100_u8) / (total_votes);

            (yes_percentage, no_percentage)
        }
    }
}
```

{{#label voting_contract}} <span class="caption">清单 {{#ref voting_contract}}: 一个投票智能合约</span>

## 部署、调用和 Invoke 投票合约

Starknet 体验的一部分是部署智能合约并与之交互。

一旦合约部署完毕，我们可以通过调用 (calling) 和 Invoke (invoking) 其函数与之交互：

- 调用合约：与仅从状态读取的外部函数交互。这些函数不会改变网络的状态，所以它们不需要费用或签名。
- Invoke 合约：与可以写入状态的外部函数交互。这些函数确实会改变网络的状态，并需要费用和签名。

我们将使用 `katana` 设置本地开发节点来部署投票合约。然后，我们将通过调用和 Invoke 其函数与合约进行交互。你也可以使用 Goerli 测试网代替 `katana`。但是，我们建议使用 `katana` 进行本地开发和测试。你可以在 Starknet 文档的 [“使用开发网络”][katana chapter] 一章中找到 `katana` 的完整教程。

[katana chapter]: https://docs.starknet.io/quick-start/using_devnet/

### `katana` 本地 Starknet 节点

`katana` 由 [Dojo 团队][dojo katana] 设计，旨在支持本地开发。它将允许你做所有你需要用 Starknet 做的事情，但是在本地。这是一个用于开发和测试的很棒的工具。

要从源代码安装 `katana`，请参阅 Dojo 引擎的 [“使用 Katana”][katana installation] 一章。

> 注意：请验证 `katana` 的版本是否与下面提供的指定版本匹配。
>
> ```bash
> $ katana --version
> katana 1.0.9-dev (38b3c2a6)
> ```
>
> 要升级 `katana` 版本，请参阅 Dojo 引擎的 [“使用 Katana”][katana installation] 一章。

一旦安装了 `katana`，你可以使用以下命令启动本地 Starknet 节点：

```bash
katana
```

此命令将启动一个带有预部署账户的本地 Starknet 节点。我们将使用这些账户来部署投票合约并与之交互：

```bash
...
PREFUNDED ACCOUNTS
==================

| Account address |  0x03ee9e18edc71a6df30ac3aca2e0b02a198fbce19b7480a63a0d71cbd76652e0
| Private key     |  0x0300001800000000300000180000000000030000000000003006001800006600
| Public key      |  0x01b7b37a580d91bc3ad4f9933ed61f3a395e0e51c9dd5553323b8ca3942bb44e

| Account address |  0x033c627a3e5213790e246a917770ce23d7e562baa5b4d2917c23b1be6d91961c
| Private key     |  0x0333803103001800039980190300d206608b0070db0012135bd1fb5f6282170b
| Public key      |  0x04486e2308ef3513531042acb8ead377b887af16bd4cdd8149812dfef1ba924d

| Account address |  0x01d98d835e43b032254ffbef0f150c5606fa9c5c9310b1fae370ab956a7919f5
| Private key     |  0x07ca856005bee0329def368d34a6711b2d95b09ef9740ebf2c7c7e3b16c1ca9c
| Public key      |  0x07006c42b1cfc8bd45710646a0bb3534b182e83c313c7bc88ecf33b53ba4bcbc
...
```

在我们与投票合约交互之前，我们需要在 Starknet 上准备好选民和管理员账户。每个选民账户必须注册并为投票提供足够的资金。要更详细地了解账户如何通过账户抽象进行操作，请参阅 Starknet 文档的 [“账户抽象”][aa chapter] 一章。

[dojo katana]: https://book.dojoengine.org/toolchain/katana
[katana installation]:
  https://book.dojoengine.org/toolchain/katana#getting-started
[aa chapter]:
  https://docs.starknet.io/architecture-and-concepts/accounts/introduction/#account_abstraction

### 用于投票的智能钱包

除了 Scarb 之外，你还需要安装 Starkli。Starkli 是一个命令行工具，允许你与 Starknet 交互。你可以在 Starknet 文档的 [“设置 Starkli”][starkli installation] 一章中找到安装说明。

> 注意：请验证 `starkli` 的版本是否与下面提供的指定版本匹配。
>
> ```bash
> $ starkli --version
> 0.3.6 (8d6db8c)
> ```
>
> 要升级 `starkli` 到 `0.3.6`，请使用 `starkliup -v 0.3.6` 命令，或者简单地使用 `starkliup` 安装最新的稳定版本。

你可以使用以下命令检索智能钱包类哈希（对于你所有的智能钱包都是一样的）。注意使用 `--rpc` 标志和 `katana` 提供的 RPC 端点：

```
starkli class-hash-at <SMART_WALLET_ADDRESS> --rpc http://0.0.0.0:5050
```

[starkli installation]: https://docs.starknet.io/quick-start/environment-setup/

### 合约部署

在部署之前，我们需要声明合约。我们可以使用 `starkli declare` 命令来完成：

```bash
starkli declare target/dev/listing_99_12_vote_contract_Vote.contract_class.json --rpc http://0.0.0.0:5050 --account katana-0
```

如果你使用的编译器版本早于 Starkli 使用的版本，并且在使用上述命令时遇到 `compiler-version` 错误，你可以通过添加 `--compiler-version x.y.z` 标志在命令中指定要使用的编译器版本。

如果你仍然遇到编译器版本问题，请尝试使用命令 `starkliup` 升级 Starkli，以确保你使用的是最新版本的 starkli。

合约的类哈希是：
`0x06974677a079b7edfadcd70aa4d12aac0263a4cda379009fca125e0ab1a9ba52`。你可以在 Sepolia 测试网上声明此合约，并查看类哈希是否对应。

`--rpc` 标志指定要使用的 RPC 端点（`katana` 提供的那个）。`--account` 标志指定用于签署交易的账户。

由于我们使用的是本地节点，交易将立即实现最终性。如果你使用的是 Goerli 测试网，你需要等待交易最终确认，这通常需要几秒钟。

以下命令部署投票合约并将 voter_0、voter_1 和 voter_2 注册为合格选民。这些是构造函数参数，所以添加一个你以后可以用来投票的选民账户。

```bash
starkli deploy <class_hash_of_the_contract_to_be_deployed> <voter_0_address> <voter_1_address> <voter_2_address> --rpc http://0.0.0.0:5050 --account katana-0
```

一个示例命令：

```bash
starkli deploy 0x06974677a079b7edfadcd70aa4d12aac0263a4cda379009fca125e0ab1a9ba52 0x03ee9e18edc71a6df30ac3aca2e0b02a198fbce19b7480a63a0d71cbd76652e0 0x033c627a3e5213790e246a917770ce23d7e562baa5b4d2917c23b1be6d91961c 0x01d98d835e43b032254ffbef0f150c5606fa9c5c9310b1fae370ab956a7919f5 --rpc http://0.0.0.0:5050 --account katana-0
```

在这种情况下，合约已部署在特定地址：
`0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349`。对你来说，这个地址会有所不同。我们将使用此地址与合约进行交互。

### 选民资格验证

在我们的投票合约中，我们有两个函数来验证选民资格，`voter_can_vote` 和 `is_voter_registered`。这些是外部读取函数，这意味着它们不会改变合约的状态，而只是读取当前状态。

`is_voter_registered` 函数检查特定地址是否在合约中注册为合格选民。另一方面，`voter_can_vote` 函数检查特定地址的选民当前是否有资格投票，即他们已注册且尚未投票。

你可以使用 `starkli call` 命令调用这些函数。注意，`call` 命令用于读取函数，而 `invoke` 命令用于也可以写入存储的函数。`call` 命令不需要签名，而 `invoke` 命令需要。

```bash+
starkli call 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 voter_can_vote 0x03ee9e18edc71a6df30ac3aca2e0b02a198fbce19b7480a63a0d71cbd76652e0 --rpc http://0.0.0.0:5050
```

首先我们添加了合约的地址，然后是我们想要调用的函数，最后是函数的输入。在这种情况下，我们正在检查地址为
`0x03ee9e18edc71a6df30ac3aca2e0b02a198fbce19b7480a63a0d71cbd76652e0` 的选民是否可以投票。

由于我们提供了一个已注册的选民地址作为输入，结果为 1（布尔值真），表明选民有资格投票。

接下来，让我们使用未注册的帐户地址调用 `is_voter_registered` 函数以观察输出：

```bash
starkli call 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 is_voter_registered 0x44444444444444444 --rpc http://0.0.0.0:5050
```

使用未注册的帐户地址，终端输出为 0（即 false），确该帐户没有资格投票。

### 投票

既然我们已经确定了如何验证选民资格，我们就可以投票了！要投票，我们与 `vote` 函数交互，该函数被标记为 external，必须使用 `starknet invoke` 命令。

`invoke` 命令语法类似于 `call` 命令，但对于投票，我们提交 `1`（赞成）或 `0`（反对）作为我们的输入。当我们调用 `vote` 函数时，我们将被收取费用，并且交易必须由选民签署；我们正在写入合约的存储。

```bash
//投赞成票
starkli invoke 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 vote 1 --rpc http://0.0.0.0:5050 --account katana-0

//投反对票
starkli invoke 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 vote 0 --rpc http://0.0.0.0:5050 --account katana-0
```

系统将提示你输入签名者的密码。输入密码后，交易将被签名并提交到 Starknet 网络。你将收到交易哈希作为输出。使用 starkli 交易命令，你可以获得有关交易的更多详细信息：

```bash
starkli transaction <TRANSACTION_HASH> --rpc http://0.0.0.0:5050
```

这返回：

```bash
{
  "transaction_hash": "0x5604a97922b6811060e70ed0b40959ea9e20c726220b526ec690de8923907fd",
  "max_fee": "0x430e81",
  "version": "0x1",
  "signature": [
    "0x75e5e4880d7a8301b35ff4a1ed1e3d72fffefa64bb6c306c314496e6e402d57",
    "0xbb6c459b395a535dcd00d8ab13d7ed71273da4a8e9c1f4afe9b9f4254a6f51"
  ],
  "nonce": "0x3",
  "type": "INVOKE",
  "sender_address": "0x3ee9e18edc71a6df30ac3aca2e0b02a198fbce19b7480a63a0d71cbd76652e0",
  "calldata": [
    "0x1",
    "0x5ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349",
    "0x132bdf85fc8aa10ac3c22f02317f8f53d4b4f52235ed1eabb3a4cbbe08b5c41",
    "0x0",
    "0x1",
    "0x1",
    "0x1"
  ]
}
```

如果你尝试使用同一个签名者投票两次，你会得到一个错误：

```bash
Error: code=ContractError, message="Contract error"
```

这个错误信息不是很有用，但是你可以查看你启动 `katana`（我们的本地 Starknet 节点）的终端中的输出以获取更多详细信息：

```bash
...
Transaction execution error: "Error in the called contract (0x03ee9e18edc71a6df30ac3aca2e0b02a198fbce19b7480a63a0d71cbd76652e0):
    Error at pc=0:81:
    Got an exception while executing a hint: Custom Hint Error: Execution failed. Failure reason: \"USER_ALREADY_VOTED\".
    ...
```

错误的键是 `USER_ALREADY_VOTED`。

```bash
assert!(can_vote, "USER_ALREADY_VOTED");
```

我们可以重复此过程，为我们想要用来投票的帐户创建签名者和帐户描述符。请记住，每个签名者必须从私钥创建，每个帐户描述符必须从公钥、智能钱包地址和智能钱包类哈希（对于每个选民都是相同的）创建。

```bash
starkli invoke 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 vote 0 --rpc http://0.0.0.0:5050 --account katana-0

starkli invoke 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 vote 1 --rpc http://0.0.0.0:5050 --account katana-0
```

### 可视化投票结果

要检查投票结果，我们通过 `starknet call` 命令调用 `get_vote_status` 函数，这是另一个视图函数。

```bash
starkli call 0x05ea3a690be71c7fcd83945517f82e8861a97d42fca8ec9a2c46831d11f33349 get_vote_status --rpc http://0.0.0.0:5050
```

输出显示了“赞成”和“反对”票的统计以及它们的相对百分比。
