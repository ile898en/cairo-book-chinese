# 与另一个合约交互

在上一节中，我们介绍了用于合约交互的分发器模式。本章将深入探讨这种模式并演示如何使用它。

分发器模式允许我们通过使用一个包装了合约地址并没有实现了由编译器从合约类 ABI 生成的分发器 trait 的结构体来调用另一个合约上的函数。这利用了 Cairo 的 trait 系统，提供了一种清晰且类型安全的方式与其他合约进行交互。

当定义了 [合约接口][interfaces] 时，编译器会自动生成并导出多个分发器。例如，对于 `IERC20` 接口，编译器将生成以下分发器：

- _合约分发器_：`IERC20Dispatcher` 和 `IERC20SafeDispatcher`
- _库分发器_：`IERC20LibraryDispatcher` 和 `IERC20SafeLibraryDispatcher`

这些分发器有不同的用途：

- 合约分发器包装合约地址，用于调用其他合约上的函数。
- 库分发器包装类哈希，用于调用类上的函数。库分发器将在下一章 [“从另一个类执行代码”][library dispatcher] 中讨论。
- _'Safe'_ 分发器允许调用者处理调用执行期间可能发生的错误。

在底层，这些分发器使用低级的 [`contract_call_syscall`][syscalls]，它允许我们通过传递合约地址、函数选择器和函数参数来调用其他合约上的函数。分发器抽象了这个系统调用的复杂性，提供了一种清晰且类型安全的方式与其他合约进行交互。

为了有效地分解涉及的概念，我们将使用 `ERC20` 接口作为示例。

[interfaces]:
  ./ch100-00-introduction-to-smart-contracts.md#the-interface-the-contracts-blueprint
[syscalls]: ./appendix-08-system-calls.md
[library dispatcher]: ./ch102-03-executing-code-from-another-class.md

## 分发器模式 (The Dispatcher Pattern)

我们提到编译器会自动为给定的接口生成分发器结构体和分发器 trait。清单 {{#ref expanded-ierc20dispatcher}} 显示了为公开 `name` 视图函数和 `transfer` 外部函数的 `IERC20` 接口生成的项目的示例：

```cairo,noplayground
use starknet::ContractAddress;

trait IERC20DispatcherTrait<T> {
    fn name(self: T) -> felt252;
    fn transfer(self: T, recipient: ContractAddress, amount: u256);
}

#[derive(Copy, Drop, starknet::Store, Serde)]
struct IERC20Dispatcher {
    pub contract_address: starknet::ContractAddress,
}

impl IERC20DispatcherImpl of IERC20DispatcherTrait<IERC20Dispatcher> {
    fn name(self: IERC20Dispatcher) -> felt252 {
        let mut __calldata__ = core::traits::Default::default();

        let mut __dispatcher_return_data__ = starknet::syscalls::call_contract_syscall(
            self.contract_address, selector!("name"), core::array::ArrayTrait::span(@__calldata__),
        );
        let mut __dispatcher_return_data__ = starknet::SyscallResultTrait::unwrap_syscall(
            __dispatcher_return_data__,
        );
        core::option::OptionTrait::expect(
            core::serde::Serde::<felt252>::deserialize(ref __dispatcher_return_data__),
            'Returned data too short',
        )
    }
    fn transfer(self: IERC20Dispatcher, recipient: ContractAddress, amount: u256) {
        let mut __calldata__ = core::traits::Default::default();
        core::serde::Serde::<ContractAddress>::serialize(@recipient, ref __calldata__);
        core::serde::Serde::<u256>::serialize(@amount, ref __calldata__);

        let mut __dispatcher_return_data__ = starknet::syscalls::call_contract_syscall(
            self.contract_address,
            selector!("transfer"),
            core::array::ArrayTrait::span(@__calldata__),
        );
        let mut __dispatcher_return_data__ = starknet::SyscallResultTrait::unwrap_syscall(
            __dispatcher_return_data__,
        );
        ()
    }
}
```

{{#label expanded-ierc20dispatcher}} <span class="caption">清单 {{#ref expanded-ierc20dispatcher}}: `IERC20Dispatcher` 及其关联 trait 和 impl 的简化示例</span>

如你所见，合约分发器是一个简单的结构体，它包装了一个合约地址并实现了编译器生成的 `IERC20DispatcherTrait`。对于每个函数，trait 的实现将包含以下元素：

- 将函数参数序列化为 `felt252` 数组 `__calldata__`。
- 使用 `contract_call_syscall` 进行低级合约调用，包含合约地址、函数选择器和 `__calldata__` 数组。
- 将返回值反序列化为预期的返回类型。

## 使用合约分发器调用合约

为了说明合约分发器的使用，让我们创建一个简单的合约，它与 ERC20 合约交互。这个包装器合约将允许我们调用 ERC20 合约上的 `name` 和 `transfer_from` 函数，如清单 {{#ref contract-dispatcher}} 所示：

```cairo,noplayground
//**** Specify interface here ****//
#[starknet::contract]
mod TokenWrapper {
    //ANCHOR: import
    use starknet::{ContractAddress, get_caller_address};
    //ANCHOR_END: import
    use super::ITokenWrapper;
    use super::{IERC20Dispatcher, IERC20DispatcherTrait};

    #[storage]
    struct Storage {}

    #[abi(embed_v0)]
    impl TokenWrapper of ITokenWrapper<ContractState> {
        fn token_name(self: @ContractState, contract_address: ContractAddress) -> felt252 {
            IERC20Dispatcher { contract_address }.name()
        }

        fn transfer_token(
            ref self: ContractState,
            address: ContractAddress,
            recipient: ContractAddress,
            amount: u256,
        ) -> bool {
            let erc20_dispatcher = IERC20Dispatcher { contract_address: address };
            erc20_dispatcher.transfer_from(get_caller_address(), recipient, amount)
        }
    }
}
```

{{#label contract-dispatcher}} <span class="caption">清单 {{#ref contract-dispatcher}}: 使用分发器模式调用另一个合约的示例合约</span>

在这个合约中，我们导入了 `IERC20Dispatcher` 结构体和 `IERC20DispatcherTrait` trait。然后我们将 ERC20 合约的地址包装在 `IERC20Dispatcher` 结构体的一个实例中。这允许我们在 ERC20 合约上调用 `name` 和 `transfer` 函数。

调用 `transfer_token` 外部函数将修改部署在 `contract_address` 的合约的状态。

## 使用安全分发器处理错误

如前所述，像 `IERC20SafeDispatcher` 这样的 'Safe' 分发器允许调用合约优雅地处理在执行被调用函数期间可能发生的潜在错误。

当通过安全分发器调用的函数发生 panic 时，执行将返回到调用者合约，并且安全分发器返回包含 panic 原因的 `Result::Err`。这允许开发人员在其合约中实现自定义错误处理逻辑。

考虑以下使用假设的 `IFailableContract` 接口的示例：

```cairo,noplayground
#[starknet::interface]
pub trait IFailableContract<TState> {
    fn can_fail(self: @TState) -> u32;
}

#[feature("safe_dispatcher")]
fn interact_with_failable_contract() -> u32 {
    let contract_address = 0x123.try_into().unwrap();
    // Use the Safe Dispatcher
    let faillable_dispatcher = IFailableContractSafeDispatcher { contract_address };
    let response: Result<u32, Array<felt252>> = faillable_dispatcher.can_fail();

    // Match the result to handle success or failure
    match response {
        Result::Ok(x) => x, // Return the value on success
        Result::Err(_panic_reason) => {
            // Handle the error, e.g., log it or return a default value
            // The panic_reason is an array of felts detailing the error
            0 // Return 0 in case of failure
        },
    }
}
```

{{#label safe-dispatcher}} <span class="caption">清单 {{#ref safe-dispatcher}}: 使用安全分发器处理错误</span>

在此代码中，我们首先获取目标合约地址的 `IFailableContractSafeDispatcher` 实例。使用此安全分发器调用 `can_fail()` 函数返回 `Result<u32, Array<felt252>>`，它封装了成功的 `u32` 结果或失败信息。然后我们可以正确处理此结果，如 [第 {{#chap error-handling}} 章：错误处理][error-handling] 中所示。

> 重要的是要注意，某些情况仍会导致立即的交易回滚，这意味着错误无法被调用者使用安全分发器捕获。这些包括：
>
> - Cairo Zero 合约调用失败。
> - 使用不存在的类哈希进行库调用。
> - 对不存在的合约地址进行合约调用。
> - `deploy` 系统调用内部失败（例如，构造函数中的 panic，部署到现有地址）。
> - 使用不存在的类哈希使用 `deploy` 系统调用。
> - 使用不存在的类哈希使用 `replace_class` 系统调用。
>
> 预计在未来的 Starknet 版本中处理这些情况。

## 使用低级调用调用合约

调用其他合约的另一种方法是直接使用 `call_contract_syscall`。虽然不如使用分发器模式方便，但此系统调用提供了对序列化和反序列化过程的更多控制，并允许更自定义的错误处理。

清单 {{#ref syscalls}} 显示了一个示例，演示如何使用低级 `call_contract_sycall` 系统调用调用 `ERC20` 合约的 `transfer_from` 函数：

```cairo,noplayground
use starknet::ContractAddress;

#[starknet::interface]
trait ITokenWrapper<TContractState> {
    fn transfer_token(
        ref self: TContractState,
        address: ContractAddress,
        recipient: ContractAddress,
        amount: u256,
    ) -> bool;
}

#[starknet::contract]
mod TokenWrapper {
    use starknet::{ContractAddress, SyscallResultTrait, get_caller_address, syscalls};
    use super::ITokenWrapper;

    #[storage]
    struct Storage {}

    impl TokenWrapper of ITokenWrapper<ContractState> {
        fn transfer_token(
            ref self: ContractState,
            address: ContractAddress,
            recipient: ContractAddress,
            amount: u256,
        ) -> bool {
            let mut call_data: Array<felt252> = array![];
            Serde::serialize(@get_caller_address(), ref call_data);
            Serde::serialize(@recipient, ref call_data);
            Serde::serialize(@amount, ref call_data);

            let mut res = syscalls::call_contract_syscall(
                address, selector!("transfer_from"), call_data.span(),
            )
                .unwrap_syscall();

            Serde::<bool>::deserialize(ref res).unwrap()
        }
    }
}
```

{{#label syscalls}} <span class="caption">清单 {{#ref syscalls}}: 使用 `call_contract_sycall` 系统调用的示例合约</span>

要使用此系统调用，我们传入合约地址、我们想要调用的函数的选择器和调用参数。调用参数必须作为参数数组提供，序列化为 `Span<felt252>`。要序列化参数，我们可以简单地使用 `Serde` trait，前提是要序列化的类型实现了此 trait。调用返回一个序列化值的数组，我们需要自己反序列化！

[error-handling]: ./ch09-00-error-handling.md
