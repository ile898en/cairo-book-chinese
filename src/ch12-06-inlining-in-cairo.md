# Cairo 中的内联 (Inlining)

内联是大多数编译器支持的一种常见的代码优化技术。它涉及用被调用函数的实际代码替换调用点处的函数调用，从而消除与函数调用本身相关的开销。这可以通过减少执行的指令数量来提高性能，但可能会增加程序的总大小。每当你考虑是否内联函数时，请考虑它的大小、它有什么参数、它被调用的频率以及它可能如何影响编译代码的大小等因素。

## `inline` 属性

在 Cairo 中，`inline` 属性建议是否应将归属于该函数的 Sierra 代码直接注入调用者函数的上下文中，而不是使用 `function_call` libfunc 来执行该代码。

你可以使用 `inline` 属性的三种变体：

- `#[inline]` 建议执行内联展开。
- `#[inline(always)]` 建议总是执行内联展开。
- `#[inline(never)]` 建议从不执行内联展开。

> 注意：每种形式的 `inline` 属性都是一个提示，语言不要求将归属函数的副本放置在调用者中。这意味着编译器可能会忽略该属性。实际上，`#[inline(always)]` 会在除最例外的情况外的所有情况下导致内联。

许多 Cairo corelib 函数都是内联的。用户定义的函数也可以用 `inline` 属性进行注释。用 `#[inline(always)]` 属性注释函数可以减少调用这些归属函数时所需的总步骤数。实际上，在调用者站点注入 Sierra 代码避免了调用函数和获取其参数所涉及的步骤成本。

然而，内联也会导致代码大小增加。每当一个函数被内联时，调用点都包含该函数的 Sierra 代码的副本，这可能导致编译代码中的代码重复。

因此，应谨慎应用内联。不加选择地使用 `#[inline]` 或 `#[inline(always)]` 会导致编译时间增加。内联小函数特别有用，最好是带有许多参数的函数。这是因为内联大函数会增加程序的代码长度，而处理许多参数会增加执行这些函数的步骤数。

函数被调用的频率越高，内联在性能方面就越有利。通过这样做，执行的步骤数将更少，而代码长度不会增长那么多，甚至在指令总数方面可能会减少。

> 内联通​​常是步骤数和代码长度之间的权衡。在适当的地方谨慎使用 `inline` 属性。

## 内联决策过程

Cairo 编译器遵循 `inline` 属性，但对于没有显式内联指令的函数，它将使用一种启发式方法。是否内联函数的决定将取决于归属函数的复杂性，并且主要依赖于阈值 `DEFAULT_INLINE_SMALL_FUNCTIONS_THRESHOLD`。

编译器使用 `ApproxCasmInlineWeight` 结构体计算函数的“权重”，该结构体估计函数将生成的 Cairo 汇编 (CASM) 语句的数量。这种权重计算提供了比简单的语句计数更细致的函数复杂性视图。如果函数的权重低于阈值，它将被内联。

除了基于权重的方法外，编译器还考虑原始语句计数。语句数少于阈值的函数通常会被内联，从而促进小的、经常调用的函数的优化。

内联过程还考虑了特殊情况。非常简单的函数，例如仅调用另一个函数或返回常量的函数，无论其他因素如何，总是被内联。相反，具有复杂控制流结构（如 `Match`）或以 `Panic` 结尾的函数通常不被内联。

## 内联示例

让我们介绍一个简短的示例来说明 Cairo 中内联的机制。清单 {{#ref inlining}} 显示了一个基本程序，允许比较内联和非内联函数。

```cairo
#[executable]
fn main() -> felt252 {
    inlined() + not_inlined()
}

#[inline(always)]
fn inlined() -> felt252 {
    1
}

#[inline(never)]
fn not_inlined() -> felt252 {
    2
}
```

{{#label inlining}} <span class="caption">清单 {{#ref inlining}}: 一个小的 Cairo 程序，它添加 2 个函数的返回值，其中一个被内联</span>

让我们看看相应的 Sierra 代码，看看内联在底层是如何工作的：

```cairo,noplayground
// type declarations
type felt252 = felt252 [storable: true, drop: true, dup: true, zero_sized: false]

// libfunc declarations
libfunc function_call<user@main::main::not_inlined> = function_call<user@main::main::not_inlined>
libfunc felt252_const<1> = felt252_const<1>
libfunc store_temp<felt252> = store_temp<felt252>
libfunc felt252_add = felt252_add
libfunc felt252_const<2> = felt252_const<2>

// statements
00 function_call<user@main::main::not_inlined>() -> ([0])
01 felt252_const<1>() -> ([1])
02 store_temp<felt252>([1]) -> ([1])
03 felt252_add([1], [0]) -> ([2])
04 store_temp<felt252>([2]) -> ([2])
05 return([2])
06 felt252_const<1>() -> ([0])
07 store_temp<felt252>([0]) -> ([0])
08 return([0])
09 felt252_const<2>() -> ([0])
10 store_temp<felt252>([0]) -> ([0])
11 return([0])

// funcs
main::main::main@0() -> (felt252)
main::main::inlined@6() -> (felt252)
main::main::not_inlined@9() -> (felt252)
```

Sierra 文件结构分为三个部分：

- 类型和 libfunc 声明。
- 构成程序的语句。
- 程序函数的声明。

Sierra 代码语句总是与 Cairo 程序中函数声明的顺序相匹配。实际上，程序函数的声明告诉我们：

- `main` 函数从第 0 行开始，并在第 5 行返回一个 `felt252`。
- `inlined` 函数从第 6 行开始，并在第 8 行返回一个 `felt252`。
- `not_inlined` 函数从第 9 行开始，并在第 11 行返回一个 `felt252`。

对应于 `main` 函数的所有语句位于第 0 行和第 5 行之间：

```cairo,noplayground
00 function_call<user@main::main::not_inlined>() -> ([0])
01 felt252_const<1>() -> ([1])
02 store_temp<felt252>([1]) -> ([1])
03 felt252_add([1], [0]) -> ([2])
04 store_temp<felt252>([2]) -> ([2])
05 return([2])
```

第 0 行调用 `function_call` libfunc 来执行 `not_inlined` 函数。这将执行从第 9 行到第 10 行的代码，并将返回值存储在 id 为 `0` 的变量中。

```cairo,noplayground
09	felt252_const<2>() -> ([0])
10	store_temp<felt252>([0]) -> ([0])
```

此代码使用单一数据类型 `felt252`。它使用两个库函数 —— `felt252_const<2>`，它返回常量 `felt252` 2，以及 `store_temp<felt252>`，它将常量值推送到内存。第一行调用 `felt252_const<2>` libfunc 来创建一个 id 为 `0` 的变量。然后，第二行将此变量推送到内存以供以后使用。

之后，从第 1 行到第 2 行的 Sierra 语句是 `inlined` 函数的实际主体：

```cairo,noplayground
06	felt252_const<1>() -> ([0])
07	store_temp<felt252>([0]) -> ([0])
```

唯一的区别是内联代码将把 `felt252_const` 值存储在 id 为 `1` 的变量中，因为 `[0]` 引用的是之前分配的变量：

```cairo,noplayground
01	felt252_const<1>() -> ([1])
02	store_temp<felt252>([1]) -> ([1])
```

> 注意：在两种情况下（内联或不内联），被调用函数的 `return` 指令都不会被执行，因为这将导致过早结束 `main` 函数的执行。相反，`inlined` 和 `not_inlined` 的返回值将被相加并返回结果。

第 3 到 5 行包含 Sierra 语句，它们将把包含在 id 为 `0` 和 `1` 的变量中的值相加，将结果存储在内存中并返回它：

```cairo,noplayground
03	felt252_add([1], [0]) -> ([2])
04	store_temp<felt252>([2]) -> ([2])
05	return([2])
```

现在，让我们看看对应于此程序的 Casm 代码，以真正了解内联的好处。

## Casm 代码解释

这是我们之前程序示例的 Casm 代码：

```cairo,noplayground
1	call rel 3
2	ret
3	call rel 9
4	[ap + 0] = 1, ap++
5	[ap + 0] = [ap + -1] + [ap + -2], ap++
6	ret
7	[ap + 0] = 1, ap++
8	ret
9	[ap + 0] = 2, ap++
10	ret
11	ret
```

不要犹豫，使用 [cairovm.codes](https://cairovm.codes/) playground 来跟随并查看所有的执行痕迹。

每条指令和任何指令的每个参数都会使程序计数器（称为 PC）增加 1。这意味着第 2 行上的 `ret` 实际上是 `PC = 3` 处的指令，因为参数 `3` 对应于 `PC = 2`。

`call` 和 `ret` 指令允许实现函数栈：

- `call` 指令像跳转指令一样行动，将 PC 更新为给定值，无论是使用 `rel` 相对于当前值还是使用 `abs` 绝对值。
- `ret` 指令跳回到 `call` 指令之后并在那里继续执行代码。

我们现在可以分解这些指令是如何执行的，以理解这段代码做了什么：

- `call rel 3`: 这条指令将 PC 增加 3 并执行位于此位置的指令，即 `PC = 4` 处的 `call rel 9`。
- `call rel 9` 将 PC 增加 9 并执行 `PC = 13` 处的指令，实际上是第 9 行。
- `[ap + 0] = 2, ap++`: `ap` 代表 Allocation Pointer（分配指针），它指向程序目前尚未使用的第一个内存单元。这意味着我们将值 `2` 存储在 `ap` 当前值指示的下一个空闲内存单元中，之后我们将 `ap` 增加 1。然后，我们去下一行，即 `ret`。
- `ret`: 跳回到 `call rel 9` 之后的行，所以我们去第 4 行。
- `[ap + 0] = 1, ap++`: 我们将值 `1` 存储在 `[ap]` 中并应用 `ap++`，使得 `[ap - 1] = 1`。这意味着我们现在有 `[ap-1] = 1, [ap-2] = 2`，我们去下一行。
- `[ap + 0] = [ap + -1] + [ap + -2], ap++`: 我们将值 `1` 和 `2` 相加并将结果存储在 `[ap]` 中，我们应用 `ap++`，所以结果是 `[ap-1] = 3, [ap-2] = 1, [ap-3]=2`。
- `ret`: 跳回到 `call rel 3` 之后的行，所以我们去第 2 行。
- `ret`: 执行的最后一条指令，因为没有更多的 `call` 指令可以在之后跳回。这是 Cairo `main` 函数的实际返回指令。

总结一下：

- `call rel 3` 对应于 `main` 函数，它显然没有被内联。
- `call rel 9` 触发对 `not_inlined` 函数的调用，该函数返回 `2` 并将其存储在最终位置 `[ap-3]`。
- 第 4 行是 `inlined` 函数的内联代码，它返回 `1` 并将其存储在最终位置 `[ap-2]`。我们可以清楚地看到在这种情况下没有 `call` 指令，因为函数体被插入并直接执行。
- 之后，计算总和，我们最终回到第 2 行，其中包含最终的 `ret` 指令，返回总和，对应于 `main` 函数的返回值。

有趣的是，在 Sierra 代码和 Casm 代码中，`not_inlined` 函数将在 `inlined` 函数的主体之前被调用和执行，即使 Cairo 程序执行 `inlined() + not_inlined()`。

> 我们程序的 Casm 代码清楚地表明，对于 `not_inlined` 函数有一个函数调用，而 `inlined` 函数被正确地内联了。

## 额外的优化

让我们研究另一个程序，它展示了内联有时可能提供的其他好处。清单 {{#ref code_removal}} 显示了一个调用 2 个函数且不返回任何内容的 Cairo 程序：

```cairo
#[executable]
fn main() {
    inlined();
    not_inlined();
}

#[inline(always)]
fn inlined() -> felt252 {
    'inlined'
}

#[inline(never)]
fn not_inlined() -> felt252 {
    'not inlined'
}
```

{{#label code_removal}} <span class="caption">清单 {{#ref code_removal}}: 一个小的 Cairo 程序，它调用 `inlined` 和 `not_inlined` 且不返回任何值。</span>

这是相应的 Sierra 代码：

```cairo,noplayground
// type declarations
type felt252 = felt252 [storable: true, drop: true, dup: true, zero_sized: false]
type Unit = Struct<ut@Tuple> [storable: true, drop: true, dup: true, zero_sized: true]

// libfunc declarations
libfunc function_call<user@main::main::not_inlined> = function_call<user@main::main::not_inlined>
libfunc drop<felt252> = drop<felt252>
libfunc struct_construct<Unit> = struct_construct<Unit>
libfunc felt252_const<29676284458984804> = felt252_const<29676284458984804>
libfunc store_temp<felt252> = store_temp<felt252>
libfunc felt252_const<133508164995039583817065828> = felt252_const<133508164995039583817065828>

// statements
00 function_call<user@main::main::not_inlined>() -> ([0])
01 drop<felt252>([0]) -> ()
02 struct_construct<Unit>() -> ([1])
03 return([1])
04 felt252_const<29676284458984804>() -> ([0])
05 store_temp<felt252>([0]) -> ([0])
06 return([0])
07 felt252_const<133508164995039583817065828>() -> ([0])
08 store_temp<felt252>([0]) -> ([0])
09 return([0])

// funcs
main::main::main@0() -> (Unit)
main::main::inlined@4() -> (felt252)
main::main::not_inlined@7() -> (felt252)
```

在这个特定案例中，我们可以观察到编译器对我们代码的 `main` 函数应用了额外的优化：`inlined` 函数的代码，它被注释为 `#[inline(always)]` 属性，实际上没有被复制到 `main` 函数中。相反，`main` 函数以 `function_call` libfunc 开始来调用 `not_inlined` 函数，完全省略了 `inlined` 函数的代码。

> 因为 `inlined` 返回值从未被使用，编译器通过跳过 `inlined` 函数代码来优化 `main` 函数。这实际上会减少代码长度，同时减少执行 `main` 所需的步骤数。

相比之下，第 0 行使用 `function_call` libfunc 来正常执行 `not_inlined` 函数。这意味着从第 7 行到第 8 行的所有代码都将被执行：

```cairo,noplayground
07 felt252_const<133508164995039583817065828>() -> ([0])
08 store_temp<felt252>([0]) -> ([0])
```

这个存储在 id 为 `0` 的变量中的值随后在第 1 行被丢弃，因为它在 `main` 函数中没被使用：

```cairo,noplayground
01 drop<felt252>([0]) -> ()
```

最后，由于 `main` 函数不返回任何值，因此创建并返回一个单元类型的变量 `()`：

```cairo,noplayground
02 struct_construct<Unit>() -> ([1])
03 return([1])
```

## 总结

内联是一种编译器优化技术，在各种情况下都非常有用。内联函数允许通过将 Sierra 代码直接注入调用者函数的上下文中来摆脱使用 `function_call` libfunc 调用函数的开销，同时可能优化执行的 Sierra 代码以减少步骤数。如果使用得当，内联甚至可以减少代码长度，如前一个示例所示。

然而，将 `inline` 属性应用于具有大量代码和少量参数的函数可能会导致代码大小增加，特别是如果内联函数在代码库中被多次使用。仅在有意义的地方使用内联，并注意编译器默认处理内联。因此，在大多数情况下不建议手动应用内联，但这可以帮助改进和微调代码的行为。
