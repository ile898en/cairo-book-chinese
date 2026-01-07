# 简介

有没有想过你的 Cairo 程序是如何执行的？

首先，它们由 Cairo 编译器编译，然后由 Cairo 虚拟机 (Cairo Virtual Machine)，简称 _Cairo VM_ 执行，生成执行跟踪 (trace)，证明者 (Prover) 使用该跟踪生成该执行的 STARK 证明。此证明稍后可由验证者 (Verifier) 验证。

接下来的章节将深入探讨 Cairo VM 的内部工作原理。我们将介绍其架构、内存模型和执行模型。接下来，我们将探讨内置函数 (builtins) 和提示 (hints)，它们的目的以及它们是如何工作的。最后，我们将看看运行器 (runner)，它负责协调 Cairo 程序的执行。

但首先，我们所说的“虚拟机”是什么意思？

## 虚拟机

虚拟机 (VM) 是物理计算机的软件仿真。它们通过 API 提供完整的编程环境，其中包括正确执行其上程序所需的一切。

每个虚拟机 API 都包含一个指令集架构 (ISA) 来表达程序。它可以是与某些物理机器相同的指令集（例如 RISC-V），也可以是在 VM 中实现的专用指令集（例如 Cairo 汇编，CASM）。

那些模拟操作系统的被称为 _系统虚拟机_，例如 Xen 和 VMWare。我们这里对它们不感兴趣。

我们感兴趣的另一类是 _进程虚拟机_。它们提供单个用户级进程所需的环境。

最著名的进程 VM 可能是 Java 虚拟机 (JVM)。

- 给定一个 Java 程序 `prgm.java`，它被编译成包含 _Java 字节码_（JVM 指令和元数据）的类 `prgm.class`。
- JVM 验证字节码是否可以安全运行。
- 字节码要么被解释执行（慢），要么即时编译（JIT，快）为机器码。
- 如果使用 JIT，字节码在执行程序时被翻译成机器码。
- Java 程序也可以通过称为 _提前编译_ (AOT) 的过程直接编译为特定 CPU 架构（即机器码）。

Cairo VM 也是一个进程 VM，类似于 JVM，但有一个显著的区别：Java 及其 JVM 专为（平台无关的）通用计算而设计，而 Cairo 及其 Cairo VM 专为（平台无关的）_可证明_ 通用计算而设计。

- Cairo 程序 `prgm.cairo` 被编译成编译工件 `prgm.json`，其中包含 _Cairo 字节码_（编码的 CASM，Cairo 指令集和额外数据）。
- 如 [简介](ch00-00-introduction.md) 中所见，Cairo Zero 直接编译为 CASM，而 Cairo 首先编译为 _Sierra_，然后再编译为 CASM 的安全子集。
- Cairo VM _解释_ 提供的 CASM 并生成程序执行的跟踪。
- 获得的跟踪数据可以提供给 Cairo Prover 以生成 STARK 证明，从而允许证明程序的正确执行。创建此 _有效性证明_ 是 Cairo 的主要目的。

这是一个高级流程图，显示了 Java 程序和 Cairo 程序如何使用各自的编译器和 VM 执行。其中包括 Cairo 程序的证明生成。

<div align="center">
  <img src="java-cairo-execution-flow.png" alt="Java and Cairo execution flow" width="800px"/>
</div>
<div align="center">
  <span class="caption">Java 和 Cairo 程序高级执行流程图</span>
</div>

一个正在进行的项目，[Cairo Native][cairo-native] 致力于提供 Sierra 到机器码的编译，包括 JIT 和 AOT，用于执行 Cairo 程序。

尽管两个 VM 的高级流程相似，但它们的实际架构却截然不同：指令集、内存模型、Cairo 的非确定性和输出。

[cairo-native]: https://github.com/lambdaclass/cairo_native

## 参考资料

Michael L. Scott, in Programming Language Pragmatics, 2015
