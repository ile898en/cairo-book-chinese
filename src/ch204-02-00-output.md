# 输出内置函数 (Output Builtin)

在 Cairo 虚拟机 (VM) 中，**输出内置函数** 是一个内置组件，它通过 `output_ptr` 管理内存的输出段。它被用作 Cairo 程序执行与外部世界之间的桥梁，使用 **公共内存** 来产生可验证的输出。

以 `output_ptr` 表示，输出内置函数处理程序写入其输出的专用内存区域。它的主要作用是存储任何必须在证明系统中可用于验证的值，我们称之为 **公共内存**。随着程序写入值，该段会增长。

## 内存组织

输出段是从基地址开始的连续单元块。输出段中的所有单元本质上都是公开的，验证者可以访问。它的交互非常简单：它可以被写入和读取，没有任何特定要求。

## 在 STARK 证明中的作用

输出内置函数与公共内存的集成对于 STARK 证明的构造和验证是必需的：

1. **公共承诺 (Public Commitment)**：写入 `output_ptr` 的值成为公共内存的一部分，在证明中承诺程序输出这些值。
2. **证明结构**：输出段包含在跟踪的公共输入中，其边界被跟踪（例如，`begin_addr` 和 `stop_ptr`）以进行验证。
3. **验证过程**：验证者提取并哈希输出段。通常，输出段中的所有单元被一起哈希，创建一个承诺，允许在不重新执行的情况下进行有效验证。

## 实现参考

以下是各种 Cairo VM 实现中输出内置函数的实现参考：

- [TypeScript Output Builtin](https://github.com/kkrt-labs/cairo-vm-ts/blob/main/src/builtins/output.ts#L4)
- [Python Output Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/vm/output_builtin_runner.py)
