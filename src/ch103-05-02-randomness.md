# 随机性 (Randomness)

由于所有区块链在根本上都是确定性的，并且大多数是公共账本，因此在链上生成真正不可预测的随机性是一个挑战。这种随机性对于游戏、彩票和 NFT 的唯一生成中的公平结果至关重要。为了解决这个问题，预言机提供的可验证随机函数 (VRF) 提供了一种解决方案。VRF 保证随机性无法被预测或篡改，从而确保这些应用程序中的信任和透明度。

## VRF 概述

VRF 使用密钥和 nonce（唯一的输入）来生成看似随机的输出。虽然技术上是“伪随机”，但如果没有密钥，另一方实际上不可能预测结果。

VRF 不仅产生随机数，还产生一个证明，任何人都可以使用该证明来独立验证结果是否根据函数的参数正确生成。

## 使用 Cartridge VRF 生成随机性

[Cartridge VRF](https://github.com/cartridge-gg/vrf) 提供了专为 Starknet 上的游戏设计的同步、链上可验证随机性 —— 尽管它也可以用于其他目的。它使用一个简单的流程：交易在对 VRF 提供者的 `request_random` 调用之前加上前缀，然后你的合约调用 `consume_random` 在同一交易中获取已验证的随机值。

### 添加 Cartridge VRF 作为依赖项

编辑你的 Cairo 项目的 `Scarb.toml` 文件以包含 Cartridge VRF。

```toml
[dependencies]
cartridge_vrf = { git = "https://github.com/cartridge-gg/vrf" }
```

### 定义合约接口

```cairo,noplayground
use starknet::ContractAddress;

#[starknet::interface]
pub trait IVRFGame<TContractState> {
    fn get_last_random_number(self: @TContractState) -> felt252;
    fn settle_random(ref self: TContractState);
    fn set_vrf_provider(ref self: TContractState, new_vrf_provider: ContractAddress);
}

#[starknet::interface]
pub trait IDiceGame<TContractState> {
    fn guess(ref self: TContractState, guess: u8);
    fn toggle_play_window(ref self: TContractState);
    fn get_game_window(self: @TContractState) -> bool;
    fn process_game_winners(ref self: TContractState);
}
```

{{#label cartridge_vrf_interface}} <span class="caption">清单 {{#ref cartridge_vrf_interface}} 展示了将 Cartridge VRF 与简单骰子游戏集成的接口。</span>

### Cartridge VRF 流程和关键入口点

Cartridge VRF 在单个交易中使用两个调用工作：

1. `request_random(caller, source)` — 必须是交易 multicall 中的第一个调用。它表明你在 `caller` 处的合约将使用指定的 `source` 消费随机值。
2. `consume_random(source)` — 由你的游戏合约调用以同步检索随机值。VRF 证明在链上验证，该值立即可用。

常见的 `source` 选择：

- `Source::Nonce(ContractAddress)` — 对提供的地址使用提供者的内部 nonce，确保每个请求都有唯一的随机值。
- `Source::Salt(felt252)` — 使用静态盐。使用相同的盐将返回相同的随机值。

## 骰子游戏合约

这个骰子游戏合约允许玩家在活动游戏窗口期间猜测 1 到 6 之间的数字。合约所有者可以切换游戏窗口以禁用新猜测。为了确定中奖号码，合约所有者调用 `settle_random`，它从 Cartridge VRF 提供者那里消费一个随机值并将其存储在 `last_random_number` 中。每个玩家然后调用 `process_game_winners` 来确定他们赢了还是输了。存储的 `last_random_number` 被归约为 1 到 6 之间的数字，并与玩家的猜测进行比较，发出 `GameWinner` 或 `GameLost`。

```cairo,noplayground
#[starknet::contract]
mod DiceGame {
    use cartridge_vrf::Source;

    // Cartridge VRF consumer component and types
    use cartridge_vrf::vrf_consumer::vrf_consumer_component::VrfConsumerComponent;
    use openzeppelin::access::ownable::OwnableComponent;
    use starknet::storage::{
        Map, StoragePathEntry, StoragePointerReadAccess, StoragePointerWriteAccess,
    };
    use starknet::{ContractAddress, get_caller_address, get_contract_address};

    component!(path: OwnableComponent, storage: ownable, event: OwnableEvent);
    component!(path: VrfConsumerComponent, storage: vrf_consumer, event: VrfConsumerEvent);

    #[abi(embed_v0)]
    impl OwnableImpl = OwnableComponent::OwnableImpl<ContractState>;
    impl InternalImpl = OwnableComponent::InternalImpl<ContractState>;

    // Expose VRF consumer helpers
    #[abi(embed_v0)]
    impl VrfConsumerImpl = VrfConsumerComponent::VrfConsumerImpl<ContractState>;
    impl VrfConsumerInternalImpl = VrfConsumerComponent::InternalImpl<ContractState>;

    #[storage]
    struct Storage {
        user_guesses: Map<ContractAddress, u8>,
        game_window: bool,
        last_random_number: felt252,
        #[substorage(v0)]
        ownable: OwnableComponent::Storage,
        #[substorage(v0)]
        vrf_consumer: VrfConsumerComponent::Storage,
    }

    #[event]
    #[derive(Drop, starknet::Event)]
    enum Event {
        GameWinner: ResultAnnouncement,
        GameLost: ResultAnnouncement,
        #[flat]
        OwnableEvent: OwnableComponent::Event,
        #[flat]
        VrfConsumerEvent: VrfConsumerComponent::Event,
    }

    #[derive(Drop, starknet::Event)]
    struct ResultAnnouncement {
        caller: ContractAddress,
        guess: u8,
        random_number: u256,
    }

    #[constructor]
    fn constructor(ref self: ContractState, vrf_provider: ContractAddress, owner: ContractAddress) {
        self.ownable.initializer(owner);
        self.vrf_consumer.initializer(vrf_provider);
        self.game_window.write(true);
    }

    #[abi(embed_v0)]
    impl DiceGame of super::IDiceGame<ContractState> {
        fn guess(ref self: ContractState, guess: u8) {
            assert!(self.game_window.read(), "GAME_INACTIVE");
            assert!(guess >= 1 && guess <= 6, "INVALID_GUESS");

            let caller = get_caller_address();
            self.user_guesses.entry(caller).write(guess);
        }

        fn toggle_play_window(ref self: ContractState) {
            self.ownable.assert_only_owner();

            let current: bool = self.game_window.read();
            self.game_window.write(!current);
        }

        fn get_game_window(self: @ContractState) -> bool {
            self.game_window.read()
        }

        fn process_game_winners(ref self: ContractState) {
            assert!(!self.game_window.read(), "GAME_ACTIVE");
            assert!(self.last_random_number.read() != 0, "NO_RANDOM_NUMBER_YET");

            let caller = get_caller_address();
            let user_guess: u8 = self.user_guesses.entry(caller).read();
            let reduced_random_number: u256 = self.last_random_number.read().into() % 6 + 1;

            if user_guess == reduced_random_number.try_into().unwrap() {
                self
                    .emit(
                        Event::GameWinner(
                            ResultAnnouncement {
                                caller: caller,
                                guess: user_guess,
                                random_number: reduced_random_number,
                            },
                        ),
                    );
            } else {
                self
                    .emit(
                        Event::GameLost(
                            ResultAnnouncement {
                                caller: caller,
                                guess: user_guess,
                                random_number: reduced_random_number,
                            },
                        ),
                    );
            }
        }
    }

    #[abi(embed_v0)]
    impl VRFGame of super::IVRFGame<ContractState> {
        fn get_last_random_number(self: @ContractState) -> felt252 {
            self.last_random_number.read()
        }

        // Settle randomness for the current round using Cartridge VRF.
        // Requires the caller to prefix the multicall with:
        //   VRF.request_random(caller: <this contract>, source: Source::Nonce(<this contract>))
        fn settle_random(ref self: ContractState) {
            self.ownable.assert_only_owner();
            // Consume a random value tied to this contract's own nonce
            let random = self.vrf_consumer.consume_random(Source::Nonce(get_contract_address()));
            self.last_random_number.write(random);
        }

        fn set_vrf_provider(ref self: ContractState, new_vrf_provider: ContractAddress) {
            self.ownable.assert_only_owner();
            self.vrf_consumer.set_vrf_provider(new_vrf_provider);
        }
    }
}
```

{{#label dice_game_vrf}} <span class="caption">清单 {{#ref dice_game_vrf}}: 使用 Cartridge VRF 的简单骰子游戏合约。</span>

#### Cartridge VRF 的调用模式

当你从帐户调用 `settle_random` 入口点时，在交易的 multicall 前加上对 VRF 提供者的 `request_random` 的调用，使用与合约将传递给 `consume_random` 相同的 `source`（在此示例中，为 `Source::Nonce(<dice_contract>)`）。例如：

1. `VRF.request_random(caller: <dice_contract>, source: Source::Nonce(<dice_contract>))`
2. `<dice_contract>.settle_random()`

这确保了 VRF 服务器可以在链上提交并验证证明，并且随机值在执行期间可供你的合约使用。

#### 部署

- Mainnet
  - 类哈希:
    https://voyager.online/class/0x00be3edf412dd5982aa102524c0b8a0bcee584c5a627ed1db6a7c36922047257
  - 合约:
    https://voyager.online/contract/0x051fea4450da9d6aee758bdeba88b2f665bcbf549d2c61421aa724e9ac0ced8f
- Sepolia
  - 类哈希:
    https://sepolia.voyager.online/class/0x00be3edf412dd5982aa102524c0b8a0bcee584c5a627ed1db6a7c36922047257
  - 合约:
    https://sepolia.voyager.online/contract/0x051fea4450da9d6aee758bdeba88b2f665bcbf549d2c61421aa724e9ac0ced8f

在示例合约中使用网络的 VRF 提供者地址作为 `vrf_provider` 构造函数参数（或通过 `set_vrf_provider`）。

更多详细信息和更新：请参阅 [Cartridge VRF 存储库](https://github.com/cartridge-gg/vrf)。
