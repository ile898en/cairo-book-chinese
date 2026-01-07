# Mod 内置函数

在 Cairo 虚拟机 (VM) 中，_Mod 内置函数_ 是一个专门的内置函数，用于处理给定模数内 field 元素（或 felts）的模运算操作——具体来说是加法和乘法。它旨在高效计算像 `a + b (mod p)` 和 `a * b (mod p)` 这样的表达式。

为此，它有两个派生：用于加法的 `AddModBuiltin` 和用于乘法的 `MulModBuiltin`。

此内置函数是加密应用程序或大量使用模运算的计算（尤其是在处理像 `UInt384` 这样的大整数时）的必需品。

## 为什么我们需要它

模运算是许多加密协议和零知识证明系统的核心。如果你尝试用纯 Cairo 代码实现这些操作，你很快就会遇到计算开销的阻碍——重复的除法和余数检查会大大增加步骤数。Mod 内置函数提供了一种优化的解决方案来高效处理这些操作。实际上，在操作 [算术电路](./ch12-10-arithmetic-circuits.md) 时，你在底层使用的就是 Mod 内置函数。

## 它的结构是如何的

每个实例都从七个输入单元格开始。其中四个——`p0`、`p1`、`p2` 和 `p3`——将模数 `p` 定义为一个多字整数，通常为了 `UInt384` 兼容性而分成四个 96 位的块。然后，操作数和结果存储在 `values_ptr` 中，`offsets_ptr` 指向另一个表，告诉内置函数相对于 `values_ptr` 在哪里读取或写入这些值。最后，`n` 指定要执行多少次操作，它需要是批处理大小的倍数。

内置函数的核心技巧是推导。给它三元组的两个部分——比如 `a` 和 `b`——它会根据操作和模数计算出第三个部分 `c`。或者，如果你提供 `c` 和一个操作数，它可以求解另一个。它分批处理这些三元组（通常一次只处理一个），确保 `op(a, b) = c + k * p` 成立，其中 `k` 是特定范围内的商——加法较小，乘法较大。实际上，你是使用 `run_mod_p_circuit` 函数设置此项，该函数将所有内容联系在一起。值表通常与 `range_check96_ptr` 段重叠，以保持每个字在 `2^96` 以下，并且偏移量是使用 `dw` 指令定义为程序字面量的。

## 投入使用：模加法

让我们通过一个使用 `AddMod` 内置函数计算两个 `UInt384` 值的 `x + y (mod p)` 的示例来演示。这段 Cairo Zero 代码展示了 Mod 内置函数如何融入实际程序。

```cairo
from starkware.cairo.common.cairo_builtins import UInt384, ModBuiltin
from starkware.cairo.common.modulo import run_mod_p_circuit
from starkware.cairo.lang.compiler.lib.registers import get_fp_and_pc

func add{range_check96_ptr: felt*, add_mod_ptr: ModBuiltin*, mul_mod_ptr: ModBuiltin*}(
    x: UInt384*, y: UInt384*, p: UInt384*
) -> UInt384* {
    let (_, pc) = get_fp_and_pc();

    // Define pointers to the offsets tables, which come later in the code
    pc_label:
    let add_mod_offsets_ptr = pc + (add_offsets - pc_label);
    let mul_mod_offsets_ptr = pc + (mul_offsets - pc_label);

    // Load x and y into the range_check96 segment, which doubles as our values table
    // x takes slots 0-3, y takes 4-7—each UInt384 is 4 words of 96 bits
    assert [range_check96_ptr + 0] = x.d0;
    assert [range_check96_ptr + 1] = x.d1;
    assert [range_check96_ptr + 2] = x.d2;
    assert [range_check96_ptr + 3] = x.d3;
    assert [range_check96_ptr + 4] = y.d0;
    assert [range_check96_ptr + 5] = y.d1;
    assert [range_check96_ptr + 6] = y.d2;
    assert [range_check96_ptr + 7] = y.d3;

    // Fire up the modular circuit: 1 addition, no multiplications
    // The builtin deduces c = x + y (mod p) and writes it to offsets 8-11
    run_mod_p_circuit(
        p=[p],
        values_ptr=cast(range_check96_ptr, UInt384*),
        add_mod_offsets_ptr=add_mod_offsets_ptr,
        add_mod_n=1,
        mul_mod_offsets_ptr=mul_mod_offsets_ptr,
        mul_mod_n=0,
    );

    // Bump the range_check96_ptr forward: 8 input words + 4 output words = 12 total
    let range_check96_ptr = range_check96_ptr + 12;

    // Return a pointer to the result, sitting in the last 4 words
    return cast(range_check96_ptr - 4, UInt384*);

    // Offsets for AddMod: point to x (0), y (4), and the result (8)
    add_offsets:
    dw 0;  // x starts at offset 0
    dw 4;  // y starts at offset 4
    dw 8;  // result c starts at offset 8

    // No offsets needed for MulMod here
    mul_offsets:
}
```

在这个函数中，我们获取两个 `UInt384` 值 `x` 和 `y`，以及一个模数 `p`。我们将 `x` 和 `y` 写入从 `range_check96_ptr` 开始的值表，然后使用偏移量——`[0, 4, 8]`——告诉内置函数 `x`、`y` 和结果 `c` 所在的位置。`run_mod_p_circuit` 调用触发 `AddMod` 内置函数计算 `x + y (mod p)` 并将结果存储在偏移量 8 处。调整指针后，我们返回指向该结果的指针。这是一个简单的设置，但内置函数为我们处理模归约，节省了大量的手动工作。

想象一个 `n_words = 1` 且 `batch_size = 1` 的简单情况。如果 `p = 5`，`x = 3`，且 `y = 4`，值表可能包含 `[3, 4, 2]`。偏移量 `[0, 1, 2]` 指向 `a = 3`，`b = 4`，和 `c = 2`。内置函数检查：`3 + 4 = 7`，且 `7 mod 5 = 2`，这与 `c` 匹配。一切都通过检查。

如果出现问题——比如说 `x` 缺少一个字——内置函数会标记 `MissingOperand` 错误。对于 `MulMod`，如果 `b` 和 `p` 互质且 `a` 未知，你会得到 `ZeroDivisor` 错误。如果任何字超过 `2^96`，范围检查将失败。这些保障措施保持操作可靠。

## 幕后花絮

让我们仔细看看 Mod 内置函数是如何设计的，以及为什么它是这样构建的。这里的目标是效率——使模运算在 Cairo 虚拟机中快速可靠——同时保持其在现实世界使用中的实用性，如加密程序。为了理解这一点，我们将探索其结构的一些关键部分以及其背后的思想。

首先，为什么要将数字分成 96 位的块，对于 `UInt384` 来说通常是四个？这不是随意的选择。Cairo VM 已经有一个内置系统来检查数字是否在特定范围内，称为 `range_check96`，它处理 96 位的值。通过将 Mod 内置函数的字大小与该系统对齐，它可以依靠 VM 现有的机制来确保数字的每个块都保持在 `2^96` 以下。对于 `UInt384`，四个 96 位字加起来是 384 位，这对于大多数加密应用程序来说足够大了。

现在，考虑 `AddMod` 和 `MulMod` 之间的区别。当你将两个接近模数 `p` 的数字相加时，结果最多为 `2p - 2`。这就是为什么 `AddMod` 将其商 `k`——减去 `p` 以通过结果的次数——限制为仅 2。如果 `a + b` 是，比如说，`1.5p`，那么 `k = 1` 且 `c = a + b - p` 保持一切正常。这是一个严格的约束，因为加法不会产生非常大的数字。但是，乘法不同。使用 `MulMod`，`a * b` 可能是巨大的——想想 `p * p` 或更多——所以默认的商界限设置得更高，高达 `2^384`。这确保了它甚至可以处理最大的乘积，而不会耗尽调整空间。

真正的巧妙之处在于内置函数如何找出缺失的值，这个过程称为推导。它不是总是从头开始，而是利用内存中已有的内容。对于 `AddMod`，如果你给它 `a` 和 `b`，它会计算 `c = a + b (mod p)`。如果你给它 `c` 和 `b`，它会求解 `a + b = c + k * p` 来找到 `a`，测试 `k = 0` 或 `1`。

对于 `MulMod`，它更棘手——乘法不像加法那样直接可逆。在这里，它使用一种称为扩展 GCD 算法的数学工具来求解 `a * b = c (mod p)`。如果 `b` 和 `p` 有公因数（它们的最大公约数不是 1），则没有唯一解，它会标记 `ZeroDivisor` 错误。否则，它会找到适合的最小 `a`。

另一个设计选择是批处理大小，实际上通常仅为 1。内置函数可以一次处理多个操作——`batch_size` 个 `a`、`b` 和 `c` 的三元组——但在大多数情况下将其保持为 1 可以简化事情。这就像一次处理一个加法或乘法，这对于许多程序来说已经足够了，尽管如果你需要，扩展的选项就在那里。

为什么要将值表绑定到 `range_check96_ptr`？这又是关于效率。VM 的范围检查系统已经设置为监视该段，因此将其用于内置函数的值——如 `a`、`b` 和 `c`——意味着这些数字会自动得到验证。

## 实现参考

以下是各种 Cairo VM 实现中 Mod 内置函数的实现参考：

- [Python Mod Builtin](https://github.com/starkware-libs/cairo-lang/blob/0e4dab8a6065d80d1c726394f5d9d23cb451706a/src/starkware/cairo/lang/builtins/modulo/mod_builtin_runner.py)
