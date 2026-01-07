# Pedersen 内置函数

_Pedersen_ 内置函数专门用于计算两个 field 元素 (felts) 的 Pedersen 哈希。它在 Cairo VM 中提供了此加密哈希函数的高效原生实现。有关在 Cairo 程序中使用哈希的指南，请参阅第 11.4 节 [使用哈希](ch12-04-hash.md)。

## 单元格组织

在 Cairo VM 运行期间，Pedersen 内置函数拥有自己专用的段。它遵循推导属性模式，并以 _单元格三元组_ 的形式组织——两个输入单元格和一个输出单元格：

- **输入单元格**：必须存储 field 元素 (felts)；禁止使用可重定位值（指针）。此限制是有道理的，因为在这种情况下，计算内存地址的哈希没有明确定义。
- **输出单元格**：该值是从输入单元格推导出来的。当指令尝试读取此单元格时，VM 会计算两个输入单元格的 Pedersen 哈希，并将结果写入输出单元格。

让我们检查 Cairo VM 执行程序期间 Pedersen 段的两个快照：

在第一个快照中，我们看到两个处于不同状态的三元组：

<div align="center">
  <img src="pedersen-builtin-valid.png" alt="valid pedersen builtin segment" width="300px"/>
</div>
<div align="center">
  <span class="caption">快照 1 - 具有有效输入的 Pedersen 内置函数段</span>
</div>

- **第一个三元组** (单元格 3:0, 3:1, 3:2)：所有三个单元格都包含 felt。单元格 3:2 (输出) 中的值已被计算，因为在程序执行期间读取了该单元格，这触发了输入 15 和 35 的 Pedersen 哈希计算。
- **第二个三元组** (单元格 3:3, 3:4, 3:5)：只有输入单元格被值 93 和 5 填充。输出单元格 3:5 仍然为空，因为它尚未被读取，因此尚未计算 93 和 5 的 Pedersen 哈希。

在第二个快照中，我们看到两种在尝试读取输出单元格时会导致错误的情况：

<div align="center">
  <img src="pedersen-builtin-error.png" alt="Invalid pedersen builtin segment" width="300px"/>
</div>
<div align="center">
  <span class="caption">快照 2 - 具有无效输入的 Pedersen 内置函数段</span>
</div>

1. **第一个三元组**：读取单元格 3:2 会抛出错误，因为其中一个输入单元格 (3:0) 为空。VM 无法在缺少输入数据的情况下计算哈希。

2. **第二个三元组**：读取单元格 3:5 会抛出错误，因为其中一个输入单元格 (3:4) 包含指向单元格 1:7 的可重定位值。Pedersen 内置函数只能对 field 元素进行哈希，而不能对内存地址进行哈希。

由于这些错误仅在读取输出单元格时才会显现。对于第二种情况，更健壮的实现可以在写入输入单元格时对其进行验证，立即拒绝可重定位值，而不是等到尝试计算哈希时。

## 实现参考

以下是各种 Cairo VM 实现中 Pedersen 内置函数的实现参考：

- [TypeScript Pedersen Builtin](https://github.com/kkrt-labs/cairo-vm-ts/blob/main/src/builtins/pedersen.ts#L4)
- [Python Pedersen Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/builtins/hash/hash_builtin_runner.py)
- [Rust Pedersen Builtin](https://github.com/lambdaclass/cairo-vm/blob/41476335884bf600b62995f0c005be7d384eaec5/vm/src/vm/runners/builtin_runner/hash.rs)
- [Zig Pedersen Builtin](https://github.com/keep-starknet-strange/ziggy-starkdust/blob/55d83e61968336f6be93486d7acf8530ba868d7e/src/vm/builtins/builtin_runner/hash.zig)

## 关于 Pedersen 哈希的资源

如果你对 Pedersen 哈希函数及其在密码学中的应用感兴趣：

- StarkNet,
  [Hash Functions - Pedersen Hash](https://docs.starknet.io/architecture-and-concepts/cryptography/hash-functions/#pedersen-hash)
- nccgroup,
  [Breaking Pedersen Hashes in Practice](https://research.nccgroup.com/2023/03/22/breaking-pedersen-hashes-in-practice/),
  2023, March 22
- Ryan S.,
  [Pedersen Hash Function Overview](https://rya-sge.github.io/access-denied/2024/05/07/pedersen-hash-function/),
  2024, May 07
