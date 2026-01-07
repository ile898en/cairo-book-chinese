# 范围检查内置函数 (Range Check Builtin)

_范围检查_ 内置函数验证 field 元素是否落在特定范围内。此内置函数是 Cairo 整数类型和比较的基础，确保存值满足有界约束。

此内置函数存在两种变体：

- 标准范围检查：验证范围 \\([0, 2^{128}-1]\\) 内的值
- 范围检查 96：验证范围 \\([0, 2^{96}-1]\\) 内的值

本节重点介绍标准变体，但相同的原则适用于两者。

## 目的和重要性

虽然可以使用纯 Cairo 代码实现范围检查（例如，通过将数字分解为其二进制表示并验证每个位），但使用内置函数的效率要高得多。纯 Cairo 实现至少需要 384 条指令来验证单个范围检查，而内置函数通过相当于约 1.5 条指令的计算成本即可实现相同的结果。这种效率使得范围检查内置函数对于实现有界整数算术和其他需要值范围验证的操作至关重要。

## 单元格组织

范围检查内置函数在具有验证属性模式的专用内存段上运行：

| 特征     | 描述                                     |
| -------- | ---------------------------------------- |
| 有效值   | 范围 \\([0, 2^{128}-1]\\) 内的 field 元素 |
| 错误条件 | 值 ≥ 2^128 或可重定位地址                |
| 验证时机 | 立即（在单元格写入时）                   |

与具有推导属性的内置函数不同，范围检查内置函数在写入时而不是读取时验证值。这种立即验证为超出范围的值提供了早期错误检测。

### 有效操作示例

<div align="center">
  <img src="range-check-builtin-valid.png" alt="valid range_check builtin segment" width="300px"/>
</div>
<div align="center">
  <span class="caption">具有有效值的范围检查内置函数段</span>
</div>

在此示例中，程序成功向范围检查段写入三个值：

- `0`：允许的最小值
- `256`：典型的小整数值
- `2^128-1`：允许的最大值

这三个值都在允许的范围 \\([0, 2^{128}-1]\\) 内，因此操作成功。

### 超出范围错误示例

<div align="center">
  <img src="range-check-builtin-error1.png" alt="invalid range_check builtin segment" width="300px"/>
</div>
<div align="center">
  <span class="caption">范围检查错误：值超出最大范围</span>
</div>

在此示例中，程序尝试将 `2^128` 写入单元格 `2:2`，这超出了允许的最大值。VM 立即抛出超出范围错误并中止执行。

### 无效类型错误示例

<div align="center">
  <img src="range-check-builtin-error2.png" alt="invalid range_check builtin segment" width="300px"/>
</div>
<div align="center">
  <span class="caption">范围检查错误：值是可重定位地址</span>
</div>

在此示例中，程序尝试将可重定位地址（指向单元格 `1:7` 的指针）写入范围检查段。由于内置函数仅接受 field 元素，VM 抛出错误并中止执行。

## 实现参考

以下是各种 Cairo VM 实现中范围检查内置函数的实现参考：

- [TypeScript Range Check Builtin](https://github.com/kkrt-labs/cairo-vm-ts/blob/main/src/builtins/rangeCheck.ts)
- [Python Range Check Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/builtins/range_check/range_check_builtin_runner.py)
- [Rust Range Check Builtin](https://github.com/lambdaclass/cairo-vm/blob/41476335884bf600b62995f0c005be7d384eaec5/vm/src/vm/runners/builtin_runner/range_check.rs)
- [Zig Range Check Builtin](https://github.com/keep-starknet-strange/ziggy-starkdust/blob/55d83e61968336f6be93486d7acf8530ba868d7e/src/vm/builtins/builtin_runner/range_check.zig)

## 关于范围检查的资源

如果你对范围检查内置函数的工作原理及其在 Cairo VM 中的使用感兴趣：

- Starknet,
  [CairoZero documentation, Range Checks section of Builtins and implicit arguments](https://docs.cairo-lang.org/how_cairo_works/builtins.html#range-checks)
- Lior G., Shahar P., Michael R.,
  [Cairo Whitepaper, Sections 2.8 and 8](https://eprint.iacr.org/2021/1063.pdf)
- [StarkWare Range Check implementation](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/cairo/common/math.cairo)
