# 按位运算内置函数 (Bitwise Builtin)

在 Cairo VM 中，_按位运算内置函数_ 可以在 field 元素上进行按位运算——具体来说是 AND (`&`)、XOR (`^`) 和 OR (`|`)。作为内置函数，它是 VM 架构的组成部分，旨在支持需要位级操作的特定任务。在 Cairo 中，这些按位运算通过更轻松地执行诸如位掩码或在特定用例中组合值之类的任务来补充 VM 的基本指令集。

## 它是如何工作的

按位运算内置函数在专用内存段上运行。每个操作使用 5 个单元格的块：

| 偏移量 | 描述          | 角色   |
| ------ | ------------- | ------ |
| 0      | x 值          | 输入   |
| 1      | y 值          | 输入   |
| 2      | x & y 结果    | 输出   |
| 3      | x ^ y 结果    | 输出   |
| 4      | x \| y 结果   | 输出   |

例如，如果 `x = 5` (二进制 `101`) 且 `y = 3` (二进制 `011`)，则输出为：

- `5 & 3 = 1` (二进制 `001`)
- `5 ^ 3 = 6` (二进制 `110`)
- `5 | 3 = 7` (二进制 `111`)

这种结构确在需要时高效、原生计算按位运算，并进行严格验证以防止错误（例如，输入超过位限制）。

## 使用示例

这是一个使用按位运算内置函数的简单 Cairo 函数。我们使用 Cairo Zero 演示它，它更接近机器代码，并允许低级操作的可视化表示。

```cairo
from starkware.cairo.common.cairo_builtins import BitwiseBuiltin

func bitwise_ops{bitwise_ptr: BitwiseBuiltin*}(x: felt, y: felt) -> (and: felt, xor: felt, or: felt) {
    assert [bitwise_ptr] = x;        // Input x
    assert [bitwise_ptr + 1] = y;    // Input y
    let and = [bitwise_ptr + 2];     // x & y
    let xor = [bitwise_ptr + 3];     // x ^ y
    let or = [bitwise_ptr + 4];      // x | y
    let bitwise_ptr = bitwise_ptr + 5;
    return (and, xor, or);
}
```

## 实现参考

以下是各种 Cairo VM 实现中按位运算内置函数的实现参考：

- [Python Bitwise Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/builtins/bitwise/bitwise_builtin_runner.py)
