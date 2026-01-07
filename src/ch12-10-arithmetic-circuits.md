# 算术电路 (Arithmetic Circuits)

算术电路是用于表示多项式计算的数学模型。它们定义在域上（通常是有限域 \\(F_p\\)，其中 \\(p\\) 是素数），并由以下部分组成：

- 输入信号（范围 \\([0, p-1]\\) 内的值）
- 算术运算（加法和乘法门）
- 输出信号

Cairo 支持模数高达 384 位的模拟算术电路。

这对于以下方面特别有用：

- 为其他证明系统实现验证
- 实现密码学原语
- 创建更低级的程序，与标准 Cairo 构造相比可能具有更低的开销

## 在 Cairo 中实现算术电路

Cairo 的电路构造在 corelib 的 `core::circuit` 模块中可用。

算术电路包括：

- 模 \\(p\\) 加法：`AddMod` builtin
- 模 \\(p\\) 乘法：`MulMod` builtin

由于模属性，我们可以构建四个基本算术门：

- 加法：`AddModGate`
- 减法：`SubModGate`
- 乘法：`MulModGate`
- 逆元：`InvModGate`

让我们创建一个计算 \\(a \cdot (a + b)\\) 的电路，在 BN254 素数域上。

我们从空的结构体 `CircuitElement<T>` 开始。

我们电路的输入定义为 `CircuitInput`：

```cairo, noplayground
    let a = CircuitElement::<CircuitInput<0>> {};
    let b = CircuitElement::<CircuitInput<1>> {};
```

我们可以组合电路输入和门：`CircuitElement<a>` 和 `CircuitElement<b>` 与加法门组合得到 `CircuitElement<AddModGate<a, b>>`。

我们可以使用 `circuit_add`、`circuit_sub`、`circuit_mul` 和 `circuit_inverse` 直接组合电路元素。对于 \\(a \* (a + b)\\)，我们的电路描述是：

```cairo, noplayground
    let add = circuit_add(a, b);
    let mul = circuit_mul(a, add);
```

注意 `a`、`b` 和 `add` 是中间电路元素，不是专门的输入或门，这就是为什么我们需要区分空结构体 `CircuitElement<T>` 和由类型 `T` 指定的电路描述。

电路的输出定义为电路元素的元组。可以添加我们电路的任何中间门，但我们必须添加所有度为 0 的门（输出信号未被用作任何其他门的输入的门）。在我们的例子中，我们将只添加最后一个门 `mul`：

```cairo, noplayground
    let output = (mul,);
```

我们现在有了我们电路及其输出的完整描述。我们现在需要为每个输入分配一个值。由于电路是用 384 位模数定义的，单个 `u384` 值可以表示为四个 `u96` 的固定数组。我们可以将 \\(a\\) 和 \\(b\\) 分别初始化为 \\(10\\) 和 \\(20\\)：

```cairo, noplayground
    let mut inputs = output.new_inputs();
    inputs = inputs.next([10, 0, 0, 0]);
    inputs = inputs.next([20, 0, 0, 0]);

    let instance = inputs.done();
```

由于输入数量可以变化，Cairo 使用累加器，并且 `new_inputs` 和 `next` 函数返回 `AddInputResult` 枚举的一个变体。

```cairo, noplayground
pub enum AddInputResult<C> {
    /// All inputs have been filled.
    Done: CircuitData<C>,
    /// More inputs are needed to fill the circuit instance's data.
    More: CircuitInputAccumulator<C>,
}
```

我们必须为每个输入分配一个值，通过对每个 `CircuitInputAccumulator` 变体调用 `next`。输入初始化后，通过调用 `done` 函数，我们得到完整的电路 `CircuitData<C>`，其中 `C` 是编码整个电路实例的长类型。

然后我们需要定义我们的电路使用什么模数（高达 384 位模数），通过定义一个 `CircuitModulus`。我们想使用 BN254 素数域模数：

```cairo, noplayground
    let bn254_modulus = TryInto::<
        _, CircuitModulus,
    >::try_into([0x6871ca8d3c208c16d87cfd47, 0xb85045b68181585d97816a91, 0x30644e72e131a029, 0x0])
        .unwrap();
```

最后一部分是电路的评估，即通过我们电路描述的每个门正确传递输入信号并获取每个输出门的值的实际过程。我们可以如下评估并获得给定模数的结果：

```cairo, noplayground
    let res = instance.eval(bn254_modulus).unwrap();
```

要检索特定输出的值，我们可以对我们的结果使用 `get_output` 函数以及我们想要的输出门的 `CircuitElement` 实例。我们也可以检索任何中间门的值。

```cairo, noplayground
    let add_output = res.get_output(add);
    let circuit_output = res.get_output(mul);

    assert!(add_output == u384 { limb0: 30, limb1: 0, limb2: 0, limb3: 0 }, "add_output");
    assert!(circuit_output == u384 { limb0: 300, limb1: 0, limb2: 0, limb3: 0 }, "circuit_output");
```

回顾一下，我们执行了以下步骤：

- 定义电路输入
- 描述电路
- 指定输出
- 为输入赋值
- 定义模数
- 评估电路
- 获取输出值

完整的代码是：

```cairo, noplayground
use core::circuit::{
    AddInputResultTrait, CircuitElement, CircuitInput, CircuitInputs, CircuitModulus,
    CircuitOutputsTrait, EvalCircuitTrait, circuit_add, circuit_mul, u384,
};

// Circuit: a * (a + b)
// witness: a = 10, b = 20
// expected output: 10 * (10 + 20) = 300
fn eval_circuit() -> (u384, u384) {
    // ANCHOR: inputs
    let a = CircuitElement::<CircuitInput<0>> {};
    let b = CircuitElement::<CircuitInput<1>> {};
    // ANCHOR_END: inputs

    // ANCHOR: description
    let add = circuit_add(a, b);
    let mul = circuit_mul(a, add);
    // ANCHOR_END: description

    // ANCHOR: output
    let output = (mul,);
    // ANCHOR_END: output

    // ANCHOR: instance
    let mut inputs = output.new_inputs();
    inputs = inputs.next([10, 0, 0, 0]);
    inputs = inputs.next([20, 0, 0, 0]);

    let instance = inputs.done();
    // ANCHOR_END: instance

    // ANCHOR: modulus
    let bn254_modulus = TryInto::<
        _, CircuitModulus,
    >::try_into([0x6871ca8d3c208c16d87cfd47, 0xb85045b68181585d97816a91, 0x30644e72e131a029, 0x0])
        .unwrap();
    // ANCHOR_END: modulus

    // ANCHOR: eval
    let res = instance.eval(bn254_modulus).unwrap();
    // ANCHOR_END: eval

    // ANCHOR: output_values
    let add_output = res.get_output(add);
    let circuit_output = res.get_output(mul);

    assert!(add_output == u384 { limb0: 30, limb1: 0, limb2: 0, limb3: 0 }, "add_output");
    assert!(circuit_output == u384 { limb0: 300, limb1: 0, limb2: 0, limb3: 0 }, "circuit_output");
    // ANCHOR_END: output_values

    (add_output, circuit_output)
}
```

## 零知识证明系统中的算术电路

在零知识证明系统中，证明者创建计算陈述的证明，验证者可以在不执行完整计算的情况下检查该证明。然而，这些陈述必须首先转换为适合证明系统的表示形式。

### zk-SNARKs 方法

一些证明系统，如 zk-SNARKs，使用有限域 \\(F_p\\) 上的算术电路。这些电路在特定门处包括约束，表示为方程：

\\[ (a_1 \cdot s_1 + ... + a_n \cdot s_n) \cdot (b_1 \cdot s_1 + ... + b_n \cdot s_n) + (c_1 \cdot s_1 + ... + c_n \cdot s_n) = 0 \mod p \\] 其中 \\(s_1, ..., s_n\\) 是信号，\\(a_i, b_i, c_i\\) 是系数。

见证 (witness) 是满足电路中所有约束的信号分配。zk-SNARK 证明使用这些属性来证明对见证的知识，而不泄露私有输入信号，确保证明者的诚实同时保护隐私。

已经做了一些工作，例如 [Garaga Groth16 verifier](https://felt.gitbook.io/garaga/deploy-your-snark-verifier-on-starknet/groth16/generate-and-deploy-your-verifier-contract)。

### zk-STARKs 方法

STARKs（Cairo 使用的）使用代数中间表示 (AIR) 代替算术电路。AIR 将计算描述为一组多项式约束。

通过允许模拟算术电路，Cairo 可用于在 STARK 证明内实现 zk-SNARKs 证明验证。
