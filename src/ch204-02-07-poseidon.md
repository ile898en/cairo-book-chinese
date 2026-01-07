# Poseidon 内置函数

_Poseidon_ 内置函数使用 Poseidon 哈希函数计算加密哈希，该函数专门为零知识证明和代数电路中的高效计算而优化。作为 Cairo 加密操作的核心组件，它使用 Hades 排列策略，结合全轮和部分轮以在 STARK 证明中实现安全性和性能。

Poseidon 对 Cairo 应用程序特别重要，因为它提供：

- 比 Pedersen 处理多个输入更好的性能
- ZK 友好的设计（针对 ZK 证明系统中的约束进行了优化）
- 强大的加密安全属性

## 单元格组织

Poseidon 内置函数使用专用内存段运行，并遵循推导属性模式：

| 单元格             | 用途                                    |
| ------------------ | --------------------------------------- |
| 输入单元格 [0-2]   | 存储 Hades 排列的输入状态               |
| 输出单元格 [3-5]   | 存储计算出的排列结果                    |

每个操作处理 6 个连续的单元格——一个包含 3 个输入的块，后跟 3 个输出。当程序读取任何输出单元格时，VM 将 Hades 排列应用于输入单元格，并用结果填充所有三个输出单元格。

### 单值哈希示例

<div align="center">
  <img src="poseidon-builtin-valid.png" alt="valid poseidon builtin segment" width="600px"/>
</div>
<div align="center">
  <span class="caption">具有有效输入的 Poseidon 内置函数段</span>
</div>

对于在第一个实例中哈希单个值 (42)：

1. 程序将值写入第一个输入单元格（位置 3:0）
2. 其他输入单元格保持默认值 (0)
3. 读取输出单元格 (3:3) 时，VM：
   - 获取初始状态 (42, 0, 0)
   - 应用填充：(42+1, 0, 0) = (43, 0, 0)
   - 计算 Hades 排列
   - 将结果存储在输出单元格 3:3 中

当哈希单个值时，排列结果的第一个分量成为哈希输出。

### 序列哈希示例

对于在第二个实例中哈希值序列 (73, 91)：

1. 程序将值写入前两个输入单元格（位置 3:6 和 3:7）
2. 读取任何输出单元格时，VM：
   - 获取状态 (73, 91, 0)
   - 应用适当的填充：(73, 91+1, 0)
   - 计算 Hades 排列
   - 将所有三个结果存储在输出单元格 (3:9, 3:10, 3:11) 中

当哈希序列时，所有三个输出状态分量均可用于进一步计算或多轮哈希中的链接。

### 错误条件

<div align="center">
  <img src="poseidon-builtin-error.png" alt="invalid poseidon builtin segment" width="300px"/>
</div>
<div align="center">
  <span class="caption">具有无效输入的 Poseidon 内置函数段</span>
</div>

在此错误示例中：

- 程序尝试将可重定位值（指向单元格 7:1 的指针）写入输入单元格 3:0
- 尝试读取输出单元格 3:3 时，VM 抛出错误
- 发生错误是因为 Poseidon 内置函数只能对 field 元素进行操作，而不能对内存地址进行操作

输入验证发生在读取输出时，而不是写入输入时，这与推导属性模式一致。

## 实现参考

以下是各种 Cairo VM 实现中 Poseidon 内置函数的实现参考：

- [TypeScript Poseidon Builtin](https://github.com/kkrt-labs/cairo-vm-ts/blob/main/src/builtins/poseidon.ts)
- [Python Poseidon Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/builtins/poseidon/poseidon_builtin_runner.py)
- [Rust Poseidon Builtin](https://github.com/lambdaclass/cairo-vm/blob/052e7cef977b336305c869fccbf24e1794b116ff/vm/src/vm/runners/builtin_runner/poseidon.rs)
- [Zig Poseidon Builtin](https://github.com/keep-starknet-strange/ziggy-starkdust/blob/55d83e61968336f6be93486d7acf8530ba868d7e/src/vm/builtins/builtin_runner/poseidon.zig)

## 关于 Poseidon 哈希的资源

如果你对 Poseidon 哈希函数及其应用感兴趣：

- [StarkNet - Hash Functions: Poseidon Hash](https://docs.starknet.io/architecture-and-concepts/cryptography/hash-functions/#poseidon-hash)
- [StarkWare - Poseidon Implementation](https://github.com/starkware-industries/poseidon/tree/main)
- [Poseidon: A New Hash Function for Zero-Knowledge Proof Systems](https://eprint.iacr.org/2019/458.pdf)
  (original paper)
- [Poseidon: ZK-friendly Hashing](https://www.poseidon-hash.info/)
- [Poseidon Journal](https://autoparallel.github.io/overview/index.html)
