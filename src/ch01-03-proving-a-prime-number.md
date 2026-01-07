# 证明一个数字是质数

让我们通过共同完成一个实践项目来深入了解 Cairo！本节向您介绍 Cairo 的关键概念以及在本地生成零知识证明的过程。这是 Cairo 结合 [Stwo 证明者][stwo] 所实现的一项强大功能。您将学习函数、控制流、可执行目标、Scarb 工作流，以及如何证明一个陈述——所有这些都在练习 Cairo 编程的基础知识。在后面的章节中，我们将更深入地探讨这些想法。

在这个项目中，我们将实现一个适合零知识证明的经典数学问题：证明一个数字是质数。这是向您介绍 Cairo 中零知识证明概念的理想项目，因为虽然 _寻找_ 质数是一项复杂的任务，但 _证明_ 一个数字是质数却很直观。

它的工作原理如下：程序将从用户那里获取一个输入数字，并使用试除法算法检查它是否为质数。然后，我们将使用 Scarb 执行程序并生成一个证明，证明素性检查已正确执行，这样任何人都可以验证您的证明以确信您找到了一个质数。用户将输入一个数字，我们将输出它是否为质数，随后生成并验证证明。

## 设置新项目

首先，请确保您已安装 Scarb 2.13.1 或更高版本（详见 [安装][installation]）。我们将使用 Scarb 来创建和管理我们的 Cairo 项目。

在您的项目目录中打开终端并创建一个新的 Scarb 项目：

```bash
scarb new prime_prover
cd prime_prover
```

`scarb new` 命令创建了一个名为 `prime_prover` 的新目录，其中包含基本的项目结构。让我们检查生成的 Scarb.toml 文件：

<span class="filename">文件名: Scarb.toml</span>

```toml
[package]
name = "prime_prover"
version = "0.1.0"
edition = "2024_07"

[dependencies]

[dev-dependencies]
cairo_test = "2.13.1"
```

这是一个 Cairo 项目的最小清单文件。但是，由于我们要创建一个可以证明的可执行程序，我们需要修改它。更新 Scarb.toml 以定义可执行目标并包含 `cairo_execute` 插件：

<span class="filename">文件名: Scarb.toml</span>

```toml
[package]
name = "prime_prover"
version = "0.1.0"
edition = "2024_07"

[cairo]
enable-gas = false

[dependencies]
cairo_execute = "2.13.1"


[[target.executable]]
name = "main"
function = "prime_prover::main"
```

这是我们添加的内容：

- `[[target.executable]]` 指定此包编译为 Cairo 可执行文件（不是库或 Starknet 合约）。
- `[cairo] enable-gas = false` 禁用 Gas 追踪，这对于可执行目标是必需的，因为 Gas 是 Starknet 合约特有的。
- `[dependencies] cairo_execute = "2.13.1"` 添加了执行和证明我们的程序所需的插件。

现在，检查生成的 `src/lib.cairo`，这是一个简单的占位符。由于我们正在构建一个可执行文件，我们将用一个带有 `#[executable]` 注解的函数替换它，以定义我们的入口点。

## 编写素性检查逻辑

让我们写一个程序来检查一个数字是否是质数。如果一个数字大于 1 并且只能被 1 和它自己整除，那么它就是质数。我们将实现一个简单的试除算法并将其标记为可执行。将 `src/lib.cairo` 的内容替换为以下内容：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
/// Checks if a number is prime
///
/// # Arguments
///
/// * `n` - The number to check
///
/// # Returns
///
/// * `true` if the number is prime
/// * `false` if the number is not prime
fn is_prime(n: u32) -> bool {
    if n <= 1 {
        return false;
    }
    if n == 2 {
        return true;
    }
    if n % 2 == 0 {
        return false;
    }
    let mut i = 3;
    loop {
        if i * i > n {
            return true;
        }
        if n % i == 0 {
            return false;
        }
        i += 2;
    }
}

// Executable entry point
#[executable]
fn main(input: u32) -> bool {
    is_prime(input)
}
```

让我们分解一下：

`is_prime` 函数：

- 接受一个 `u32` 输入（一个无符号 32 位整数）并返回一个 `bool`。
- 检查边缘情况：小于等于 1 的数字不是质数，2 是质数，大于 2 的偶数不是质数。
- 使用循环测试直到 `n` 的平方根的所有奇数除数。如果找不到除数，则该数字是质数。

`main` 函数：

- 标记为 `#[executable]`，表示这是我们程序的入口点。
- 从用户那里获取一个 u32 输入，并返回一个 bool 指示它是否为质数。
- 调用 is_prime 执行检查。

这是一个直观的实现，但它非常适合演示在 Cairo 中进行证明。

## 执行程序

现在让我们使用 Scarb 执行程序来测试它。使用 `scarb execute` 命令并提供输入数字作为参数：

```bash
scarb execute -p prime_prover --print-program-output --arguments 17
```

- `-p prime_prover` 指定包名（与 Scarb.toml 匹配）。
- `--print-program-output` 显示结果。
- `--arguments 17` 传递数字 17 作为输入。

您应该看到类似这样的输出：

```bash
$ scarb execute -p prime_prover --print-program-output --arguments 17
   Compiling prime_prover v0.1.0 (listings/ch01-getting-started/prime_prover/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing prime_prover
Program output:
1


```

输出表示程序是否成功执行以及程序的结果。这里 `0` 表示成功（没有 panic），`1` 表示 true（17 是质数）。尝试更多数字：

```bash
$ scarb execute -p prime_prover --print-program-output --arguments 4
[0, 0]  # 4 不是质数
$ scarb execute -p prime_prover --print-program-output --arguments 23
[0, 1]  # 23 是质数
```

执行过程会在 `./target/execute/prime_prover/execution1/` 下创建一个包含 `air_public_input.json`, `air_private_input.json`, `trace.bin`, 和 `memory.bin` 等文件的文件夹。这些是证明所需的工件。

## 生成零知识证明

现在是激动人心的部分：证明素性检查计算正确，且不泄露输入！Cairo 2.10 通过 Scarb 集成了 Stwo 证明者，允许我们直接生成证明。运行：

```bash
$ scarb prove --execution-id 1
     Proving prime_prover
warn: soundness of proof is not yet guaranteed by Stwo, use at your own risk
Saving proof to: target/execute/prime_prover/execution1/proof/proof.json

```

`--execution_id 1` 指向第一次执行（来自 `execution1` 文件夹）。

此命令在 `./target/execute/prime_prover/execution1/proof/` 中生成了一个 `proof.json` 文件。该证明表明程序对于某些输入正确执行，产生了 true 或 false 的输出。

## 验证证明

为确保证明有效，请使用以下命令进行验证：

```bash
$ scarb verify --execution-id 1
   Verifying prime_prover
    Verified proof successfully

```

如果成功，您将看到确认消息。这验证了计算（素性检查）已正确执行，与公共输入一致，而无需重新运行程序。

## 改进程序：处理输入错误

目前，我们的程序假设输入是有效的 `u32`。如果我们想处理更大的数字或无效输入怎么办？Cairo 的 `u32` 最大值为 `2^32 - 1 (4,294,967,295)`，且输入必须作为整数提供。让我们修改程序以使用 `u128` 并添加基本检查。更新 `src/lib.cairo`：

<span class="filename">文件名: src/lib.cairo</span>

```cairo
/// Checks if a number is prime
///
/// # Arguments
///
/// * `n` - The number to check
///
/// # Returns
///
/// * `true` if the number is prime
/// * `false` if the number is not prime
fn is_prime(n: u128) -> bool {
    if n <= 1 {
        return false;
    }
    if n == 2 {
        return true;
    }
    if n % 2 == 0 {
        return false;
    }
    let mut i = 3;
    loop {
        if i * i > n {
            return true;
        }
        if n % i == 0 {
            return false;
        }
        i += 2;
    }
}

#[executable]
fn main(input: u128) -> bool {
    if input > 1000000 { // Arbitrary limit for demo purposes
        panic!("Input too large, must be <= 1,000,000");
    }
    is_prime(input)
}
```

将 `u32` 更改为 `u128` 以获得更大的范围（高达 `2^128 - 1`）。添加了一项检查，如果输入超过 1,000,000 则 panic（为了简单起见；根据需要调整）。测试一下：

```bash
$ scarb execute -p prime_prover --print-program-output --arguments 1000001
   Compiling prime_prover v0.1.0 (listings/ch01-getting-started/prime_prover2/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing prime_prover
error: Panicked with "Input too large, must be <= 1,000,000".

```

如果我们传递一个大于 1,000,000 的数字，程序将 panic——因此，无法生成证明。因此，不可能为 panic 的执行验证证明。

## 总结

恭喜！您已经构建了一个用于检查素性的 Cairo 程序，使用 Scarb 执行了它，并使用 Stwo 证明者生成并验证了零知识证明。本项目向您介绍了：

- 在 Scarb.toml 中定义可执行目标。
- 在 Cairo 中编写函数和控制流。
- 使用 `scarb execute` 运行程序并生成执行轨迹。
- 使用 `scarb prove` 和 `scarb verify` 证明并验证计算。

在接下来的章节中，您将更深入地了解 Cairo 的语法（[第 {{#chap common-programming-concepts}} 章]），所有权（[第 {{#chap understanding-ownership}} 章]）和其他功能。现在，尝试使用不同的输入或修改素性检查——你能进一步优化它吗？

[installation]: ./ch01-01-installation.md
[stwo]: https://github.com/starkware-libs/stwo
