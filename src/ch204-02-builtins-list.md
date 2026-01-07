# 内置函数列表

下表列出了 Cairo VM 中实现的不同内置函数，并简要说明了它们的用途。对于每个内置函数，都有一个特定部分详细介绍其工作原理、单元格组织（如果有）以及它们在 Cairo VM 的不同实现中的实际实现的参考。

如果相关，还提供了与内置函数执行的操作相关的其他资源。

| 内置函数                 | 描述                                                                                                       |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| [Output][output]         | 存储生成 STARK 证明所需的所有公共内存（输入和输出值、内置指针...）                                         |
| [Pedersen][pedersen]     | 计算两个 felt `a` 和 `b` 的 Pedersen 哈希 `h`。`h = Pedersen(a, b)`                                        |
| [Range Check][rc]        | 验证 felt `x` 是否在边界 `[0, 2**128)` 内。                                                                |
| [ECDSA][ecdsa]           | 验证给定公钥 `pub` 对消息 `m` 的 ECDSA 签名是否等于之前存储的 `sig`。仅由 Cairo Zero 使用。                |
| [Bitwise][bitwise]       | 计算两个 felt `a` 和 `b` 的按位与、异或和或。`a & b`、`a ^ b` 和 `a \| b`。                                |
| [EC OP][ec_op]           | 执行椭圆曲线操作 - 对于 STARK 曲线上的两个点 `P`、`Q` 和一个标量 `m`，计算 `R = P + mQ`。                  |
| [Keccak][keccak]         | 在给定状态 `s` 上应用 24 轮 keccak-f1600 块排列后计算新状态 `s'`。                                         |
| [Poseidon][poseidon]     | 在给定状态 `s` 上应用 91 轮 hades 块排列后计算新状态 `s'`。                                                |
| [Range Check96][rc96]    | 验证 felt `x` 是否在边界 `[0, 2**96)` 内。                                                                 |
| [AddMod][add_mod]        | 算术电路支持 - 分批计算两个 felt `a`、`b` 的模加法 `c`。`c ≡ a + b mod(p)`                                 |
| [MulMod][mul_mod]        | 算术电路支持 - 分批计算两个 felt `a`、`b` 的模乘法 `c`。`c ≡ a * b mod(p)`                                 |
| [Segment Arena][seg_are] | 管理 Cairo 字典。在 Cairo Zero 中不使用。                                                                  |
| [Gas][gas]               | 管理运行期间的可用 gas。Starknet 用它来处理其 gas 使用并避免 DoS。                                         |
| [System][system]         | 管理 Starknet 系统调用和作弊码 (cheatcodes)。                                                              |

[output]: ch204-02-00-output.md
[pedersen]: ch204-02-01-pedersen.md
[rc]: ch204-02-02-range-check.md
[ecdsa]: ch204-02-03-ecdsa.md
[bitwise]: ch204-02-04-bitwise.md
[ec_op]: ch204-02-05-ec-op.md
[keccak]: ch204-02-06-keccak.md
[poseidon]: ch204-02-07-poseidon.md
[rc96]: ch204-02-08-range-check-96.md
[add_mod]: ch204-02-09-add-mod.md
[mul_mod]: ch204-02-10-mul-mod.md
[seg_are]: ch204-02-11-segment-arena.md
[gas]: ch204-02-12-gas.md
[system]: ch204-02-13-system.md
