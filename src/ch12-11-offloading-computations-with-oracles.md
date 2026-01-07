# 使用预言机卸载计算 (Offloading Computations with Oracles)

在本章中，我们将构建一个小的 Cairo 可执行文件，它请求外部助手（“预言机”）做一些工作，然后在 Cairo 内部约束返回值，使它们成为证明的一部分。在此过程中，你将学习什么是预言机，为什么它们自然地适合 Cairo 的非确定性机器，以及如何安全地使用它们。

预言机是 Scarb 的一个实验性功能，可供使用 `--experimental-oracles` 执行的 Cairo 可执行文件使用。它们在 Starknet 合约中不可用。

> 注意：这个“预言机”系统与智能合约预言机无关；但概念相似：使用外部进程在受限系统中查询数据。

## 为什么要使用预言机？

当 Cairo VM 运行时，预言机允许证明者为内存单元分配任意值。这种非确定性让我们“注入”来自外部世界的值。例如，要证明我们知道 \\(\sqrt{25}\\)，我们不需要在 Cairo 中实现平方根算法；我们可以从预言机获得 \\(5\\) 并简单地断言 \\(5 \cdot 5 = 25\\)。

如果你对底层内存模型感到好奇，请参阅 [非确定性只读存储器](./ch202-01-non-deterministic-read-only-memory.md)。

## 我们将构建什么

我们将创建两个协同工作的部分：

- 一个 Cairo 可执行文件，它调用两个预言机端点：一个返回数字的整数平方根，另一个将数字分解为小端字节。每次调用后，我们将断言必须成立的属性，以保持程序的可靠性属性。
- 一个 Rust 进程，实现这些端点并通过标准输入/输出通过 JSON-RPC（Scarb 执行器支持的 `stdio:` 协议）与 Cairo 通信。

我们在 `listing_oracles/` 下准备了一个完整的示例。我们将浏览重要文件然后运行它。

## Cairo 包

首先，让我们看看清单。我们声明一个可执行包，依赖于 `cairo_execute` 以便我们可以使用 Scarb 运行，并添加 `oracle` crate 以访问 `oracle::invoke`。

<span class="caption">文件名: listing_oracles/Scarb.toml</span>

```toml
[package]
name = "example"
version = "0.1.0"
edition = "2024_07"
publish = false

[dependencies]
cairo_execute = "2.13.1"
oracle = "0.1.0-dev.4"

[executable]

[cairo]
enable-gas = false

[dev-dependencies]
cairo_test = "2"
```

现在让我们看看我们如何从 Cairo 调用预言机。我们定义两个辅助函数，使用形式为 `stdio:...` 的连接字符串转发到 Rust 预言机，然后我们断言使返回值成为证明一部分的关系。

<span class="caption">文件名: listing_oracles/src/lib.cairo</span>

```cairo
use core::num::traits::Pow;

// Call into the Rust oracle to get the square root of an integer.
fn sqrt_call(x: u64) -> oracle::Result<u64> {
    oracle::invoke("stdio:cargo -q run --manifest-path ./src/my_oracle/Cargo.toml", 'sqrt', (x,))
}

// Call into the Rust oracle to convert an integer to little-endian bytes.
fn to_le_bytes(val: u64) -> oracle::Result<Array<u8>> {
    oracle::invoke(
        "stdio:cargo -q run --manifest-path ./src/my_oracle/Cargo.toml", 'to_le_bytes', (val,),
    )
}

fn oracle_calls(x: u64) -> Result<(), oracle::Error> {
    let sqrt = sqrt_call(x)?;
    // CONSTRAINT: sqrt * sqrt == x
    assert!(sqrt * sqrt == x, "Expected sqrt({x}) * sqrt({x}) == x, got {sqrt} * {sqrt} == {x}");
    println!("Computed sqrt({x}) = {sqrt}");

    let bytes = to_le_bytes(x)?;
    // CONSTRAINT: sum(bytes_i * 256^i) == x
    let mut recomposed_val = 0;
    for (i, byte) in bytes.span().into_iter().enumerate() {
        recomposed_val += (*byte).into() * 256_u64.pow(i.into());
    }
    assert!(
        recomposed_val == x,
        "Expected recomposed value {recomposed_val} == {x}, got {recomposed_val}",
    );
    println!("le_bytes decomposition of {x}) = {:?}", bytes.span());

    Ok(())
}

#[executable]
fn main(x: u64) -> bool {
    match oracle_calls(x) {
        Ok(()) => true,
        Err(e) => panic!("Oracle call failed: {e:?}"),
    }
}
```

这里有两个重要的想法：

1. 我们用 `oracle::invoke(connection, selector, inputs_tuple)` 调用预言机。`connection` 告诉 Scarb 如何生成进程（这里是通过 `stdio:` 的 Cargo 命令），`selector` 按名称选择端点，元组包含输入。返回类型是 `oracle::Result<T>`，所以我们显式处理错误。
2. 我们立即约束从预言机返回的任何内容。对于平方根，我们断言 `sqrt * sqrt == x`。对于字节分解，我们从其字节重新计算该值并断言它等于原始数字。这些断言是将注入的值转化为可靠见证数据的原因。正确约束返回值非常重要；否则，恶意证明者可以将任意值注入内存，并伪造任意有效的 ZK-Proofs。

## Rust 预言机

在 Rust 方面，我们实现端点并让辅助 crate (`cairo_oracle_server`) 处理管道。输入自动解码；输出被编码回 Cairo。

<span class="caption">文件名: listing_oracles/src/my_oracle/Cargo.toml</span>

```toml
[package]
name = "my_oracle"
version = "0.1.0"
edition = "2021"
publish = false

[dependencies]
anyhow = "1"
cairo-oracle-server = "0.1"
starknet-core = "0.11"
```

<span class="caption">文件名: listing_oracles/src/my_oracle/src/main.rs</span>

```rust, noplayground
use anyhow::ensure;
use cairo_oracle_server::Oracle;
use std::process::ExitCode;

fn main() -> ExitCode {
    Oracle::new()
        .provide("sqrt", |value: u64| {
            let sqrt = (value as f64).sqrt() as u64;
            ensure!(
                sqrt * sqrt == value,
                "Cannot compute integer square root of {value}"
            );
            Ok(sqrt)
        })
        .provide("to_le_bytes", |value: u64| {
            let value_bytes = value.to_le_bytes();
            Ok(value_bytes.to_vec())
        })
        .run()
}
```

`sqrt` 端点返回整数平方根并拒绝没有精确平方根的值。`to_le_bytes` 端点返回 `u64` 的小端字节分解。

## 运行示例

从示例目录中，在启用预言机的情况下执行程序：

```bash
scarb execute --experimental-oracles --print-program-output --arguments 25000000
```

你会看到程序返回 `1`，表示成功。Cairo 代码向预言机询问 `sqrt(25000000)`，验证 `5000 * 5000 == 25000000`，然后将 `25000000` 分解为字节并验证重新组合它们等于原始输入。

尝试一个不是完全平方数的值，例如 `27`：

```bash
scarb execute --experimental-oracles --print-program-output --arguments 27
```

`sqrt` 端点将返回错误，因为 `27` 没有整数平方根，这会传播回 Cairo。我们的程序会 panic。

## 快速浏览 API

所有预言机交互都通过 Cairo 侧的一个函数进行：

```text
oracle::invoke(connection: felt252*, selector: felt252*, inputs: (..)) -> oracle::Result<T>
```

连接字符串选择传输和要运行的进程（这里是 `stdio:` 加上 Cargo 命令）。选择器是你在 Rust 中提供的端点名称（例如 `'sqrt'`）。输入是与 Rust 处理程序参数匹配的 Cairo 元组。返回类型是 `oracle::Result<T>`，所以你可以用 `match`、`unwrap_or` 或自定义逻辑处理错误。

## 总结

你现在有一个工作示例，展示了如何将工作卸载到外部进程并将结果作为 Cairo 证明的一部分。当你想在客户端证明期间使用快速、灵活的助手时，请使用此模式，并记住：预言机是实验性的，仅限运行器，并且来自它们的所有内容必须由你的 Cairo 代码验证。
