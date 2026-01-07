# 附录 E - 常见错误信息

在编写 Cairo 代码时，你可能会遇到错误信息。其中一些非常频繁地发生，这就是为什么我们将在这个附录中列出最常见的错误信息，以帮助你解决常见问题。

- `Variable not dropped.`: 这个错误信息意味着你正试图让一个具有未实现 `Drop` trait 的类型的变量离开作用域，而没有销毁它。确保在函数执行结束时需要被丢弃的变量实现了 `Drop` trait 或 `Destruct` trait。请参阅 [所有权](ch04-01-what-is-ownership.md#destroying-values---example-with-feltdict) 章节。

- `Variable was previously moved.`: 这个错误信息意味着你正试图使用一个所有权已经转移给另一个函数的变量。当变量没有实现 `Copy` trait 时，它按值传递给函数，变量的所有权转移给函数。在所有权转移后，该变量不能再在当前上下文中使用。在这种情况下，使用 `clone` 方法通常很有用。

- `error: Trait has no implementation in context: core::fmt::Display::<package_name::struct_name>`:
  如果尝试在 `print!` 或 `println!` 宏中使用 `{}` 占位符打印自定义数据类型的实例，则会遇到此错误消息。为了解决这个问题，你要么需要为你的类型手动实现 `Display` trait，要么使用 `Debug` trait，通过对你的类型应用 `derive(Debug)`，允许通过在 `{}` 占位符中添加 `:?` 来打印你的实例。

- `Got an exception while executing a hint: Hint Error: Failed to deserialize param #x.`:
  这个错误意味着执行失败，因为调用了一个入口点而没有提供预期的参数。确保调用入口点时提供的参数是正确的。`u256` 变量有一个经典问题，它实际上是 2 个 `u128` 的结构体。因此，当调用接受 `u256` 作为参数的函数时，你需要传递 2 个值。

- `Item path::item is not visible in this context.`: 这个错误信息让我们知道引入项到作用域的路径是正确的，但是存在可见性问题。在 Cairo 中，默认情况下所有项对父模块都是私有的。要解决此问题，请确保通往项的路径上的所有模块以及项本身都声明为 `pub(crate)` 或 `pub` 以便可以访问它们。

- `Identifier not found.`: 这个错误信息有点不具体，但可能表明：
  - 变量在声明之前就被使用了。确保使用 `let` 关键字声明变量。
  - 将项引入作用域的路径定义错误。确保使用有效路径。

## Starknet 组件相关错误信息

在尝试实现组件时，你可能会遇到一些错误。不幸的是，其中一些缺乏有意义的错误信息来帮助调试。本节旨在为你提供一些指针来帮助你调试代码。

- `Trait not found. Not a trait.`: 当你没有在合约中正确导入组件的 impl 块时，可能会发生此错误。确保遵守以下语法：

  ```cairo,noplayground
  #[abi(embed_v0)]
  impl IMPL_NAME = PATH_TO_COMPONENT::EMBEDDED_NAME<ContractState>
  ```

- `Plugin diagnostic: name is not a substorage member in the contract's Storage. Consider adding to Storage: (...)`:
  编译器通过为你提供有关要采取的操作的建议来帮助你调试此问题。基本上，你忘记了将组件的存储添加到合约的存储中。确保将带有 `#[substorage(v0)]` 属性注释的组件存储路径添加到合约的存储中。

- `Plugin diagnostic: name is not a nested event in the contract's Event enum. Consider adding to the Event enum:`
  类似于前一个错误，编译器告诉你忘记了将组件的事件添加到合约的事件中。确保将组件事件的路径添加到合约的事件中。
