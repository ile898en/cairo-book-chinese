# Keccak 内置函数

_Keccak_ 内置函数实现了 SHA-3 加利福尼亚哈希函数系列的核心功能。它通过对输入状态 `s` 应用 24 轮 keccak-f1600 排列来计算新状态 `s'`。此内置函数对于以太坊兼容性特别重要，因为以太坊使用 Keccak-256 进行各种加密操作。

## 单元格组织

Keccak 内置函数使用一个专用内存段，按 16 个连续单元格的块进行组织：

| 单元格范围   | 用途            | 描述                                       |
| ------------ | --------------- | ------------------------------------------ |
| 前 8 个单元格 | 输入状态 `s`    | 每个单元格存储 1600 位输入状态中的 200 位  |
| 后 8 个单元格 | 输出状态 `s'`   | 每个单元格存储 1600 位输出状态中的 200 位  |

内置函数独立处理每个块，应用以下规则：

1. **输入验证**：每个输入单元格必须包含不超过 200 位 (0 ≤ value < 2^200) 的有效 field 元素
2. **惰性计算**：仅当访问任何输出单元格时才计算输出状态
3. **缓存**：一旦计算，结果将被缓存，以避免在访问同一块中的其他输出单元格时进行冗余计算

### 操作示例

<div align="center">
  <img src="keccak-segment.png" alt="keccak builtin segment"/>
</div>
<div align="center">
  <span class="caption">具有完整操作的 Keccak 内置函数段</span>
</div>

在此示例中：

- 程序已将输入值 [0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8] 写入前 8 个单元格
- 读取任何输出单元格后，VM 将对整个状态计算 keccak-f1600 排列
- 结果输出状态存储在接下来的 8 个单元格中
- 每个块仅计算一次并缓存

### 错误条件

在以下情况下，Keccak 内置函数将抛出错误：

- 如果任何输入单元格包含超过 200 位 (≥ 2^200) 的值
- 如果任何输入单元格包含可重定位值（指针）而不是 field 元素
- 如果在初始化所有八个输入单元格之前读取输出单元格

## 实现参考

以下是各种 Cairo VM 实现中 Keccak 内置函数的实现参考：

- [TypeScript Keccak Builtin](https://github.com/kkrt-labs/cairo-vm-ts/blob/main/src/builtins/keccak.ts)
- [Python Keccak Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/builtins/keccak/keccak_builtin_runner.py)
- [Rust Keccak Builtin](https://github.com/lambdaclass/cairo-vm/blob/41476335884bf600b62995f0c005be7d384eaec5/vm/src/vm/runners/builtin_runner/keccak.rs)
- [Zig Keccak Builtin](https://github.com/keep-starknet-strange/ziggy-starkdust/blob/55d83e61968336f6be93486d7acf8530ba868d7e/src/vm/builtins/builtin_runner/keccak.zig)

## 关于 Keccak 哈希的资源

如果你对 Keccak 哈希函数及其应用感兴趣：

- StarkNet,
  [Hash Functions - Starknet Keccak](https://docs.starknet.io/architecture-and-concepts/cryptography/hash-functions/#starknet_keccak)
- NIST,
  [SHA-3 Standard: Permutation-Based Hash and Extendable-Output Functions](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.202.pdf)
- Wikipedia,
  [SHA-3 (Secure Hash Algorithm 3)](https://en.wikipedia.org/wiki/SHA-3)
- Keccak Team, [Keccak Reference](https://keccak.team/keccak_specs_summary.html)
