# 价格喂送 (Price Feeds)

由预言机启用的价格喂送充当现实世界数据源和区块链之间的桥梁。它们将来自多个可信外部来源（例如加密货币交易所、金融数据提供商等）的实时定价数据聚合到区块链网络。

对于本书本节中的示例，我们将使用 Pragma Oracle 读取 `ETH/USD` 资产对的价格喂送，并展示一个利用此喂送的迷你应用程序。

[Pragma Oracle](https://www.pragma.build/) 是一种领先的零知识预言机，它提供在 Starknet 区块链上以可验证的方式访问链下数据。

## 设置你的合约以进行价格喂送

### 添加 Pragma 作为项目依赖项

要开始在 Cairo 智能合约上集成 Pragma 以获取价格喂送数据，请编辑项目的 `Scarb.toml` 文件以包含使用 Pragma 的路径。

```toml
[dependencies]
pragma_lib = { git = "https://github.com/astraly-labs/pragma-lib" }
```

### 创建价格喂送合约

为项目添加必要的依赖项后，你需要定义一个包含所需 Pragma 价格喂送入口点的合约接口。

```cairo,noplayground
#[starknet::interface]
pub trait IPriceFeedExample<TContractState> {
    fn buy_item(ref self: TContractState);
    fn get_asset_price(self: @TContractState, asset_id: felt252) -> u128;
}
```

在 `IPriceFeedExample` 中公开的两个公共函数中，与 Pragma 价格喂送预言机交互所需的是 `get_asset_price` 函数，这是一个接收 `asset_id` 参数并返回 `u128` 值的视图函数。

### 导入 Pragma 依赖项

```cairo,noplayground
    use pragma_lib::abi::{IPragmaABIDispatcher, IPragmaABIDispatcherTrait};
    use pragma_lib::types::{DataType, PragmaPricesResponse};
    use starknet::contract_address::contract_address_const;
    use starknet::get_caller_address;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use super::{ContractAddress, IPriceFeedExample};

    const ETH_USD: felt252 = 19514442401534788;
    const EIGHT_DECIMAL_FACTOR: u256 = 100000000;

    #[storage]
    struct Storage {
        pragma_contract: ContractAddress,
        product_price_in_usd: u256,
    }

    #[constructor]
    fn constructor(ref self: ContractState, pragma_contract: ContractAddress) {
        self.pragma_contract.write(pragma_contract);
        self.product_price_in_usd.write(100);
    }

    #[abi(embed_v0)]
    impl PriceFeedExampleImpl of IPriceFeedExample<ContractState> {
        fn buy_item(ref self: ContractState) {
            let caller_address = get_caller_address();
            let eth_price = self.get_asset_price(ETH_USD).into();
            let product_price = self.product_price_in_usd.read();

            // Calculate the amount of ETH needed
            let eth_needed = product_price * EIGHT_DECIMAL_FACTOR / eth_price;

            let eth_dispatcher = ERC20ABIDispatcher {
                contract_address: contract_address_const::<
                    0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7,
                >() // ETH Contract Address
            };

            // Transfer the ETH to the caller
            eth_dispatcher
                .transfer_from(
                    caller_address,
                    contract_address_const::<
                        0x0237726d12d3c7581156e141c1b132f2db9acf788296a0e6e4e9d0ef27d092a2,
                    >(),
                    eth_needed,
                );
        }

        //ANCHOR: price_feed_impl
        fn get_asset_price(self: @ContractState, asset_id: felt252) -> u128 {
            // Retrieve the oracle dispatcher
            let oracle_dispatcher = IPragmaABIDispatcher {
                contract_address: self.pragma_contract.read(),
            };

            // Call the Oracle contract, for a spot entry
            let output: PragmaPricesResponse = oracle_dispatcher
                .get_data_median(DataType::SpotEntry(asset_id));

            return output.price;
        }
        //ANCHOR_END: price_feed_impl
    }
}
//ANCHOR_END: here


```

上面的代码片段显示了为了与 Pragma 预言机交互，你需要添加到合约模块的必要导入。

### 合约中所需的价格喂送函数实现

```cairo,noplayground
        fn get_asset_price(self: @ContractState, asset_id: felt252) -> u128 {
            // Retrieve the oracle dispatcher
            let oracle_dispatcher = IPragmaABIDispatcher {
                contract_address: self.pragma_contract.read(),
            };

            // Call the Oracle contract, for a spot entry
            let output: PragmaPricesResponse = oracle_dispatcher
                .get_data_median(DataType::SpotEntry(asset_id));

            return output.price;
        }
```

`get_asset_price` 函数负责从 Pragma Oracle 检索由 `asset_id` 参数指定的资产价格。通过传递 `DataType::SpotEntry(asset_id)` 作为参数从 `IPragmaDispatcher` 实例调用的 `get_data_median` 方法，其输出被分配给名为 `output` 的 `PragmaPricesResponse` 类型变量。最后，该函数以 `u128` 形式返回请求资产的价格。

## 使用 Pragma 价格喂送的示例应用程序

```cairo,noplayground
#[starknet::contract]
mod PriceFeedExample {
    //ANCHOR_END: pragma_lib
    use openzeppelin::token::erc20::interface::{ERC20ABIDispatcher, ERC20ABIDispatcherTrait};
    //ANCHOR: pragma_lib
    use pragma_lib::abi::{IPragmaABIDispatcher, IPragmaABIDispatcherTrait};
    use pragma_lib::types::{DataType, PragmaPricesResponse};
    use starknet::contract_address::contract_address_const;
    use starknet::get_caller_address;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use super::{ContractAddress, IPriceFeedExample};

    const ETH_USD: felt252 = 19514442401534788;
    const EIGHT_DECIMAL_FACTOR: u256 = 100000000;

    #[storage]
    struct Storage {
        pragma_contract: ContractAddress,
        product_price_in_usd: u256,
    }

    #[constructor]
    fn constructor(ref self: ContractState, pragma_contract: ContractAddress) {
        self.pragma_contract.write(pragma_contract);
        self.product_price_in_usd.write(100);
    }

    #[abi(embed_v0)]
    impl PriceFeedExampleImpl of IPriceFeedExample<ContractState> {
        fn buy_item(ref self: ContractState) {
            let caller_address = get_caller_address();
            let eth_price = self.get_asset_price(ETH_USD).into();
            let product_price = self.product_price_in_usd.read();

            // Calculate the amount of ETH needed
            let eth_needed = product_price * EIGHT_DECIMAL_FACTOR / eth_price;

            let eth_dispatcher = ERC20ABIDispatcher {
                contract_address: contract_address_const::<
                    0x049d36570d4e46f48e99674bd3fcc84644ddd6b96f7c741b1562b82f9e004dc7,
                >() // ETH Contract Address
            };

            // Transfer the ETH to the caller
            eth_dispatcher
                .transfer_from(
                    caller_address,
                    contract_address_const::<
                        0x0237726d12d3c7581156e141c1b132f2db9acf788296a0e6e4e9d0ef27d092a2,
                    >(),
                    eth_needed,
                );
        }

        //ANCHOR: price_feed_impl
        fn get_asset_price(self: @ContractState, asset_id: felt252) -> u128 {
            // Retrieve the oracle dispatcher
            let oracle_dispatcher = IPragmaABIDispatcher {
                contract_address: self.pragma_contract.read(),
            };

            // Call the Oracle contract, for a spot entry
            let output: PragmaPricesResponse = oracle_dispatcher
                .get_data_median(DataType::SpotEntry(asset_id));

            return output.price;
        }
        //ANCHOR_END: price_feed_impl
    }
}
```

> **注意**：Pragma 使用 6 或 8 的小数位数因子返回不同代币对的值。你可以通过将值除以 \\( {10^{n}} \\) 来将值转换为所需的小数位数因子，其中 `n` 是小数位数因子。

上面的代码是一个使用来自 Pragma 预言机的价格喂送的应用程序的实现示例。合约导入了必要的模块和接口，包括用于与 Pragma 预言机合约交互的 `IPragmaABIDispatcher` 和用于与 ETH ERC20 代币合约交互的 `ERC20ABIDispatcher`。

该合约有一个存储 `ETH/USD` 代币对 ID 的 `const`，和一个包含两个字段 `pragma_contract` 和 `product_price_in_usd` 的 `Storage` 结构体。构造函数初始化 `pragma_contract` 地址并将 `product_price_in_usd` 设置为 100。

`buy_item` 函数是以太坊购买产品的主要入口点。它检索调用者的地址。它调用 `get_asset_price` 函数使用 `ETH_USD` 资产 ID 获取以美元计价的当前 ETH 价格。它根据相应 ETH 价格下的美元产品价格计算购买产品所需的 ETH 数量。然后，它通过调用 ERC20 ETH 合约上的 `balance_of` 方法来检查调用者是否有足够的 ETH。如果调用者有足够的 ETH，它会调用 `eth_dispatcher` 实例的 `transfer_from` 方法，将所需数量的 ETH 从调用者转移到另一个合约地址。

`get_asset_price` 函数是与 Pragma 预言机交互的入口点，在上一节已经解释过。

你可以从他们的 [文档](https://docs.pragma.build/Resources/Starknet/data-feeds/consuming-data) 中获得有关使用 Pragma 价格喂送消费数据的详细指南。
