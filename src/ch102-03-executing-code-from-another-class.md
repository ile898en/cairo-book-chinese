# 从另一个类执行代码

在前面的章节中，我们探讨了如何调用外部 _合约_ 来执行其逻辑并更新其状态。但是如果我们想执行来自另一个类的代码而不更新另一个合约的状态呢？Starknet 通过 _库调用 (library calls)_ 使这成为可能，它允许合约在自己的上下文中执行另一个类的逻辑，更新其自己的状态。

## 库调用 (Library calls)

_合约调用_ 和 _库调用_ 之间的主要区别在于类中定义的逻辑的执行上下文。虽然合约调用用于调用已部署 **合约** 中的函数，但库调用用于在调用者的上下文中调用无状态 **类**。

为了说明这一点，让我们考虑两个合约 _A_ 和 _B_。

当 A 对 **合约** B 执行 _合约调用_ 时，B 中定义的逻辑的执行上下文是 B 的上下文。因此，B 中的 `get_caller_address()` 将返回 A 的地址，B 中的 `get_contract_address()` 将返回 B 的地址，并且 B 中的任何存储更新都将更新 B 的存储。

然而，当 A 使用 _库调用_ 来调用 B 的 **类** 时，B 中定义的逻辑的执行上下文是 A 的上下文。这意味着 B 中 `get_caller_address()` 返回的值将是 A 的调用者的地址，B 的类中 `get_contract_address()` 将返回 A 的地址，并且在 B 的类中更新存储变量将更新 A 的存储。

可以使用前一章介绍的分发器模式执行库调用，只是使用类哈希而不是合约地址。

清单 {{#ref expanded-ierc20-library}} 使用相同的 `IERC20` 示例描述了库分发器及其关联的 `IERC20DispatcherTrait` trait 和 impl：

```cairo,noplayground
//TAG: does_not_compile
use starknet::ContractAddress;

trait IERC20DispatcherTrait<T> {
    fn name(self: T) -> felt252;
    fn transfer(self: T, recipient: ContractAddress, amount: u256);
}

#[derive(Copy, Drop, starknet::Store, Serde)]
struct IERC20LibraryDispatcher {
    class_hash: starknet::ClassHash,
}

impl IERC20LibraryDispatcherImpl of IERC20DispatcherTrait<IERC20LibraryDispatcher> {
    fn name(
        self: IERC20LibraryDispatcher,
    ) -> felt252 { // starknet::syscalls::library_call_syscall  is called in here
    }
    fn transfer(
        self: IERC20LibraryDispatcher, recipient: ContractAddress, amount: u256,
    ) { // starknet::syscalls::library_call_syscall  is called in here
    }
}
```

{{#label expanded-ierc20-library}} <span class="caption">清单 {{#ref expanded-ierc20-library}}: `IERC20LibraryDispatcher` 及其关联 trait 和 impl 的简化示例</span>

与合约分发器的一个显著区别是，库分发器使用 `library_call_syscall` 而不是 `call_contract_syscall`。否则，过程是相似的。

让我们看看如何使用库调用在当前合约的上下文中执行另一个类的逻辑。

## 使用库分发器

清单 {{#ref library-dispatcher}} 定义了两个合约：`ValueStoreLogic`，它定义了我们示例的逻辑，以及 `ValueStoreExecutor`，它只是执行 `ValueStoreLogic` 类的逻辑。

我们首先需要导入 `IValueStoreDispatcherTrait` 和 `IValueStoreLibraryDispatcher`，它们是由编译器从我们的接口生成的。然后，我们可以创建一个 `IValueStoreLibraryDispatcher` 的实例，传入我们想要进行库调用的类的 `class_hash`。从那里，我们可以调用在该类中定义的函数，在我们的合约上下文中执行其逻辑。

```cairo,noplayground
#[starknet::interface]
trait IValueStore<TContractState> {
    fn set_value(ref self: TContractState, value: u128);
    fn get_value(self: @TContractState) -> u128;
}

#[starknet::contract]
mod ValueStoreLogic {
    use starknet::ContractAddress;
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};

    #[storage]
    struct Storage {
        value: u128,
    }

    #[abi(embed_v0)]
    impl ValueStore of super::IValueStore<ContractState> {
        fn set_value(ref self: ContractState, value: u128) {
            self.value.write(value);
        }

        fn get_value(self: @ContractState) -> u128 {
            self.value.read()
        }
    }
}

#[starknet::contract]
mod ValueStoreExecutor {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use starknet::{ClassHash, ContractAddress};
    use super::{IValueStoreDispatcherTrait, IValueStoreLibraryDispatcher};

    #[storage]
    struct Storage {
        logic_library: ClassHash,
        value: u128,
    }

    #[constructor]
    fn constructor(ref self: ContractState, logic_library: ClassHash) {
        self.logic_library.write(logic_library);
    }

    #[abi(embed_v0)]
    impl ValueStoreExecutor of super::IValueStore<ContractState> {
        fn set_value(ref self: ContractState, value: u128) {
            IValueStoreLibraryDispatcher { class_hash: self.logic_library.read() }
                .set_value((value));
        }

        fn get_value(self: @ContractState) -> u128 {
            IValueStoreLibraryDispatcher { class_hash: self.logic_library.read() }.get_value()
        }
    }

    #[external(v0)]
    fn get_value_local(self: @ContractState) -> u128 {
        self.value.read()
    }
}
```

{{#label library-dispatcher}} <span class="caption">清单 {{#ref library-dispatcher}}: 使用库分发器的示例合约</span>

当我们调用 `ValueStoreExecutor` 上的 `set_value` 函数时，它将对 `ValueStoreLogic` 中定义的 `set_value` 函数进行库调用。因为我们使用的是库调用，`ValueStoreExecutor` 的存储变量 `value` 将被更新。同样，当我们调用 `get_value` 函数时，它将对 `ValueStoreLogic` 中定义的 `get_value` 函数进行库调用，返回存储变量 `value` 的值 —— 仍然是在 `ValueStoreExecutor` 的上下文中。

因此，`get_value` 和 `get_value_local` 都返回相同的值，因为它们读取相同的存储槽。

## 使用低级调用调用类

调用类的另一种方法是直接使用 `library_call_syscall`。虽然不如使用分发器模式方便，但此系统调用提供了对序列化和反序列化过程的更多控制，并允许更自定义的错误处理。

清单 {{#ref library_syscall}} 显示了一个示例，演示如何使用 `library_call_syscall` 调用 `ValueStore` 合约的 `set_value` 函数：

```cairo,noplayground
#[starknet::contract]
mod ValueStore {
    use starknet::storage::{StoragePointerReadAccess, StoragePointerWriteAccess};
    use starknet::{ClassHash, SyscallResultTrait, syscalls};

    #[storage]
    struct Storage {
        logic_library: ClassHash,
        value: u128,
    }

    #[constructor]
    fn constructor(ref self: ContractState, logic_library: ClassHash) {
        self.logic_library.write(logic_library);
    }

    #[external(v0)]
    fn set_value(ref self: ContractState, value: u128) -> bool {
        let mut call_data: Array<felt252> = array![];
        Serde::serialize(@value, ref call_data);

        let mut res = syscalls::library_call_syscall(
            self.logic_library.read(), selector!("set_value"), call_data.span(),
        )
            .unwrap_syscall();

        Serde::<bool>::deserialize(ref res).unwrap()
    }

    #[external(v0)]
    fn get_value(self: @ContractState) -> u128 {
        self.value.read()
    }
}
```

{{#label library_syscall}} <span class="caption">清单 {{#ref library_syscall}}: 使用 `library_call_syscall` 系统调用的示例合约</span>

要使用此系统调用，我们传入类哈希、我们想要调用的函数的选择器和调用参数。调用参数必须作为参数数组提供，序列化为 `Span<felt252>`。要序列化参数，我们可以简单地使用 `Serde` trait，前提是要序列化的类型实现了此 trait。调用返回一个序列化值的数组，我们需要自己反序列化！

## 总结

祝贺你完成本章！你已经学到了很多新概念：

- _合约_ 与 _类_ 有何不同，以及 ABI 如何为外部源描述它们
- 如何使用 _分发器_ 模式调用其他合约和类中的函数
- 如何使用 _库调用_ 在调用者的上下文中执行另一个类的逻辑
- Starknet 提供的用于与合约和类交互的两个系统调用

你现在拥有开发跨多个合约和类分布逻辑的复杂应用程序所需的所有工具。在下一章中，我们将探索更高级的主题，这将帮助你释放 Starknet 的全部潜力。
