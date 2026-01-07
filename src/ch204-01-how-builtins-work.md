# 内置函数如何工作

内置函数对 Cairo 内存强制执行某些约束以执行特定任务，例如计算哈希。

每个内置函数都在专用的内存段上工作，该内存段最终代表固定的地址范围。这种通信方法称为 _内存映射 I/O_：专用于内置函数的特定内存地址范围。

为了让 Cairo 程序与内置函数交互，它只需要读取或写入相应的内存单元。

我们称之为 _验证属性_ 和 _推导属性_ 的内置函数约束主要有两种类型。具有推导属性的内置函数通常被分成通过验证属性约束某些单元的单元块。

如果定义的属性不成立，则 Cairo VM 将 panic。

## 验证属性

验证属性定义了值必须满足的约束，才能将其写入内置函数内存单元。

例如，_范围检查 (Range Check)_ 内置函数仅接受 felt，并验证该 felt 是否在 `[0, 2**128)` 范围内。只有当这两个约束成立时，程序才能将值写入范围检查内置函数。这两个约束代表范围检查内置函数的验证属性。

<div align="center">
  <img src="range-check-validation-property.png" alt="Diagram snapshot Cairo memory using the Range Check builtin" width="800px"/>
</div>
<div align="center">
  <span class="caption">使用范围检查内置函数的 Cairo VM 内存图</span>
</div>

## 推导属性

推导属性在读取或写入单元时定义对单元块的约束。

一个单元块有两类单元：

- _输入单元_ - 程序可以写入的单元，它们的约束类似于验证属性。
- _输出单元_ - 程序必须读取的单元，其值是基于推导属性和输入单元值计算的。

仅写入输入单元而从未读取输出单元的程序，只要这些单元上的约束成立，就是有效的。虽然，它是无用的。

例如，_Pedersen_ 内置函数使用三元组单元：

- 两个输入单元存储两个 felt，`a` 和 `b`。
- 一个输出单元将存储 `Pedersen(a, b)`。

要计算 `a` 和 `b` 的 Pedersen 哈希，程序必须：

- 将 `a` 写入第一个单元
- 将 `b` 写入第二个单元
- 读取第三个单元，它将计算并将 `Pedersen(a, b)` 写入其中。

在下图中，使用了 Pedersen 内置函数，突出了其推导属性：在将其值写入单元 `1:5` 时读取输出单元 `2:2`。

<div align="center">
  <img src="pedersen-deduction-property.png" alt="Diagram of Cairo VM memory Pedersen builtins" width="800px"/>
</div>
<div align="center">
  <span class="caption">使用 Pedersen 内置函数的 Cairo VM 内存图</span>
</div>
