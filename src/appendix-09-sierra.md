# Sierra

从 Starknet Alpha v0.11.0 开始，编译 Cairo 产生的合约类包含称为 Safe Intermediate Representation（简称 _Sierra_）的中间表示指令。然后，排序器通过 Sierra &rarr; Casm 编译器编译这个新合约类，以生成与此类关联的 Cairo 汇编。然后，Casm 代码由 Starknet OS 执行。

## 为什么我们需要 Casm？

Starknet 是一个有效性 rollup (validity rollup)，这意味着每个区块内的执行都需要被证明，这就是 STARK 派上用场的地方。然而，STARK 证明只能处理用多项式约束语言表述的陈述，并且不了解智能合约的执行。为了克服这个差距，我们开发了 Cairo。

Cairo 指令（以前称为 Casm）被翻译成多项式约束，根据 [_Cairo：图灵完备的 STARK 友好 CPU 架构_](https://github.com/starknet-io/starknet-stack-resources/blob/main/Cairo/Cairo%20%E2%80%93%20a%20Turing-complete%20STARK-friendly%20CPU%20architecture.pdf) 中定义的 Cairo 语义强制执行程序的正确执行。

多亏了 Cairo，我们可以用一种我们可以证明的方式来表述“这个 Starknet 区块是有效的”这一陈述。请注意，我们只能证明关于 Casm 的事情。也就是说，无论用户向 Starknet 排序器发送什么，被证明的都是正确的 Casm 执行。

这意味着我们需要一种将 Sierra 翻译成 Casm 的方法，这通过 Sierra -> Casm 编译器来实现。

## 为什么我们需要 Sierra？

为了理解为什么我们选择在用户编写的代码和被证明的代码 (Casm) 之间添加一个额外的层，我们需要考虑系统中的更多组件，以及 Cairo 的局限性。

### 撤销的交易、不可满足的 AIR 和 DoS 攻击

每个去中心化 L2 的一个关键属性是，排序器保证会因其所做的工作而获得报酬。撤销的交易 (reverted transactions) 是一个很好的例子：即使用户的交易在执行中途失败，排序器也应该能够将其包含在区块中，并收取直到失败点的执行费用。

如果排序器无法对此类交易收费，那么发送最终会失败（经过大量计算步骤后）的交易是对排序器的明显 DoS 攻击。排序器无法在不实际执行工作的情况下查看交易并断定它会失败（这等同于解决停机问题）。

上述困境的明显解决方案是将此类交易包含在区块中，类似于以太坊。然而，在有效性 rollup 中这可能并不容易做到。使用 Cairo Zero，在用户代码和被证明的内容之间没有分离层。

这意味着用户可以编写在某些情况下无法证明的代码。实际上，此类代码很容易编写，例如 `assert 0=1` 是一个有效的 Cairo 指令，无法被证明，因为它转换为不可满足的多项式约束。包含此指令的任何 Casm 执行都无法被证明。Sierra 是用户代码和可证明陈述之间的层，它使我们能够确保所有交易最终都是可证明的。

### 安全 Casm

Sierra 保证用户代码始终可证明的方法是将 Sierra 指令编译为 Casm 的子集，我们称之为“安全 Casm”。我们对安全 Casm 的重要要求是它对所有输入都是可证明的。安全 Casm 的一个典型示例是使用 `if/else` 指令而不是 `assert`，即确保所有失败都是优雅的。

为了更好地理解设计 Sierra &rarr; Casm 编译器时的考虑因素，请考虑 Cairo Zero 公共库中的 `find_element` 函数：

```
func find_element{range_check_ptr}(array_ptr: felt*, elm_size, n_elms, key) -> (elm_ptr: felt*) {
    alloc_locals;
    local index;
    %{
        ...
    %}
    assert_nn_le(a=index, b=n_elms - 1);
    tempvar elm_ptr = array_ptr + elm_size * index;
    assert [elm_ptr] = key;
    return (elm_ptr=elm_ptr);
}
```

> 下面我们滥用“Casm”符号，不区分 Cairo Zero 和 Casm，并将上述内容称为 Casm（虽然我们实际上指的是上述内容的编译结果）。

为简洁起见，我们在上面的代码片段中省略了 hint，但很明显，只有在数组中存在请求的元素时，此函数才能正确执行（否则它将对每个可能的 hint 失败——我们无法用任何东西替换 `index`，使后续行成功运行）。

Sierra->Casm 编译器无法生成此类 Casm。此外，简单地将断言替换为 if/else 语句也不行，因为这会导致非确定性执行。也就是说，对于相同的输入，不同的 hint 值可能会产生不同的结果。恶意的证明者可以利用这种自由来伤害用户——在这个例子中，他们能够让它看起来好像一个元素不是数组的一部分，即使它实际上是。

用于在数组中查找元素的安全 Casm 在快乐流程（元素存在）中的行与上面的代码片段类似：hint 中给出一个索引，我们验证 hint 索引处的数组包含请求的元素。但是，在不快乐流程（元素不存在）中，我们 _必须_ 遍历整个数组来验证这一点。

在 Cairo 0 中情况并非如此，因为我们可以接受某些路径不可证明（在上面的代码片段中，元素不在数组中的不快乐流程永远不可证明）。

> Sierra 的 gas 计量给上面的例子增加了更多的复杂性。即使遍历数组以验证元素不存在，也可能给证明者留下一些灵活性。

如果我们考虑 gas 限制，用户可能有足够的 gas 用于快乐流程，但不足以用于不快乐流程，这使得执行在搜索中途停止，并允许证明者通过谎称元素不存在而逃脱惩罚。

我们计划处理此问题的方法是要求用户在实际调用 `find_element` 之前拥有足够的 gas 用于不快乐流程。

### Cairo 中的 Hints

使用 Cairo 编写的智能合约不能包含用户定义的 hints。这在 Cairo Zero 合约中已经是这样（只接受列入白名单的 hints），但在 Cairo 中，使用的 hints 由 Sierra &rarr; Casm 编译器确定的。由于此编译旨在确保仅生成“安全”的 Casm，因此没有空间容纳非编译器生成的 hints。

将来，原生 Cairo 可能包含类似于 Cairo 0 的 hint 语法，但它在 Starknet 智能合约中将不可用（Starknet 之上的 [L3](https://medium.com/starkware/fractal-scaling-from-l2-to-l3-7fe238ecfb4f) 可能会使用此类功能）。请注意，目前这才不是 Starknet 路线图的一部分。
