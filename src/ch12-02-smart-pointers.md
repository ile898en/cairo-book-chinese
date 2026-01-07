# 智能指针

指针是一个通用概念，指包含内存地址的变量。此地址引用或“指向”其他一些数据。虽然指针是一个强大的功能，但它们也可能是错误和安全漏洞的来源。例如，指针可以引用未分配的内存单元，这意味着尝试访问该地址处的数据会导致程序崩溃，使其变得不可证明。为了防止此类问题，Cairo 使用 _智能指针 (Smart Pointers)_。

智能指针是像指针一样行动的数据结构，但也具有额外的元数据和功能。智能指针的概念并非 Cairo 独有：智能指针起源于 C++，并且也存在于像 Rust 这样的其他语言中。在 Cairo 的特定情况下，智能指针通过严格的类型检查和所有权规则提供一种访问内存的安全方式，确保不会以可能导致程序不可证明的不安全方式寻址内存。

虽然我们当时没有这样称呼它们，但我们在本书中已经遇到过一些智能指针，包括 [第 {{#chap common-collections}} 章][common collections] 中的 `Felt252Dict<T>` 和 `Array<T>`。这两种类型都算作智能指针，因为它们拥有一个内存段并允许你操作它。它们还具有元数据和额外的功能或保证。数组跟踪其当前长度，以确保现有元素不被覆盖，并且新元素仅追加到末尾。

Cairo VM 内存由多个可以存储数据的段组成，每个段由唯一索引标识。当你创建一个数组时，你在内存中分配一个新的段来存储未来的元素。数组本身只是一个指向存储元素的那个段的指针。

[common collections]: ./ch03-00-common-collections.md

## 用于操作指针的 `Box<T>` 类型

Cairo 中的主要智能指针类型是一个 _box_，表示为 `Box<T>`。手动定义 boxes 允许你将数据存储在 Cairo VM 的特定内存段中，称为 _boxed segment_。此段专门用于存储所有 boxed 值，留在执行段中的只是一个指向 boxed 段的指针。每当你实例化一个新的 `Box<T>` 类型的指针变量时，你就将 `T` 类型的数据追加到 boxed 段。

除了将它们的内部值写入 boxed 段外，Boxes 几乎没有性能开销。但它们也没有很多额外的功能。你将在这些情况下最常使用它们：

- 当你有一个在编译时无法知道大小的类型，并且你想要在需要确切大小的上下文中使用该类型的值时
- 当你有大量数据并且你想要转移所有权但确保在这样做时数据不会被复制时

我们将在 [“使用 Boxes 启用递归类型”][nullable recursive types] 部分演示第一种情况。在第二种情况下，转移大量数据的所有权可能需要很长时间，因为数据会在内存中被复制。为了改善这种情况下的性能，我们可以使用 box 类型将大量数据存储在 boxed 段中。然后，只有少量的指针数据在内存中被复制，而它引用的数据则留在 boxed 段的一个地方。

[nullable recursive types]: ./ch12-02-smart-pointers.md#enabling-recursive-types-with-nullable-boxes

### 使用 `Box<T>` 在 Boxed Segment 中存储数据

在讨论 `Box<T>` 的 boxed segment 存储用例之前，我们将介绍语法以及如何与存储在 `Box<T>` 中的值进行交互。

清单 {{#ref basic_box}} 展示了如何使用 box 在 boxed segment 中存储一个值：

```cairo
#[executable]
fn main() {
    let b = BoxTrait::new(5_u128);
    println!("b = {}", b.unbox())
}
```

{{#label basic_box}} <span class="caption">清单 {{#ref basic_box}}: 使用 box 在 boxed segment 中存储 `u128` 值</span>

我们定义变量 `b` 具有指向值 `5` 的 `Box` 的值，该值存储在 boxed segment 中。该程序将打印 `b = 5`；在这种情况下，我们可以像访问执行内存中的数据一样访问 box 中的数据。将单个值放在 box 中并不是很有用，所以你不会经常以这种方式单独使用 boxes。像单个 `u128` 这样的值默认存储在执行内存中，在大多数情况下更合适。让我们看一个例子，其中 boxes 允许我们定义如果没有 boxes 我们将被禁止定义的类型。

### 使用 Boxes 启用递归类型

递归类型的值可以拥有相同类型的另一个值作为其自身的一部分。递归类型会带来问题，因为在编译时 Cairo 需要知道一个类型占用多少空间。然而，递归类型的值的嵌套理论上可以无限继续，所以 Cairo 无法知道该值需要多少空间。因为 boxes 有一个已知的大小，我们可以通过在递归类型定义中插入一个 box 来启用递归类型。

作为递归类型的一个例子，让我们探索二叉树的实现。我们将定义的二叉树类型除了递归之外都很简单；因此，我们将使用的示例中的概念在你遇到涉及递归类型的更复杂情况时都会很有用。

二叉树是一种树数据结构，其中每个节点最多有两个子节点，分别称为左子节点和右子节点。分支的最后一个元素是叶子，它是一个没有子节点的节点。

清单 {{#ref recursive_types_wrong}} 显示了实现 `u32` 值二叉树的尝试。注意这段代码还不能编译，因为 `BinaryTree` 类型没有已知的大小，我们将演示这一点。

```cairo, noplayground
//TAG: does_not_compile

#[derive(Copy, Drop)]
enum BinaryTree {
    Leaf: u32,
    Node: (u32, BinaryTree, BinaryTree),
}

#[executable]
fn main() {
    let leaf1 = BinaryTree::Leaf(1);
    let leaf2 = BinaryTree::Leaf(2);
    let leaf3 = BinaryTree::Leaf(3);
    let node = BinaryTree::Node((4, leaf2, leaf3));
    let _root = BinaryTree::Node((5, leaf1, node));
}
```

{{#label recursive_types_wrong}} <span class="caption">清单 {{#ref recursive_types_wrong}}: 实现 `u32` 值二叉树的第一次尝试</span>

> 注意：为了本示例的目的，我们实现了一个仅保存 u32 值的二叉树。我们可以使用泛型来实现它，正如我们在第 {{#chap generic-types-and-traits}} 章中讨论的那样，来定义一个可以存储任何类型值的二叉树。

根节点包含 5 和两个子节点。左子节点是一个包含 1 的叶子。右子节点是另一个包含 4 的节点，它反过来又有两个叶子子节点：一个包含 2，另一个包含 3。此结构形成了一个深度为 2 的简单二叉树。

如果我们尝试编译清单 {{#ref recursive_types_wrong}} 中的代码，我们会得到以下错误：

```plaintext
$ scarb build 
   Compiling listing_recursive_types_wrong v0.1.0 (listings/ch12-advanced-features/listing_recursive_types_wrong/Scarb.toml)
error: Recursive type "(core::integer::u32, listing_recursive_types_wrong::BinaryTree, listing_recursive_types_wrong::BinaryTree)" has infinite size.
 --> listings/ch12-advanced-features/listing_recursive_types_wrong/src/lib.cairo:6:5
    Node: (u32, BinaryTree, BinaryTree),
    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: Recursive type "listing_recursive_types_wrong::BinaryTree" has infinite size.
 --> listings/ch12-advanced-features/listing_recursive_types_wrong/src/lib.cairo:11:17
    let leaf1 = BinaryTree::Leaf(1);
                ^^^^^^^^^^^^^^^^^^^

error: Recursive type "(core::integer::u32, listing_recursive_types_wrong::BinaryTree, listing_recursive_types_wrong::BinaryTree)" has infinite size.
 --> listings/ch12-advanced-features/listing_recursive_types_wrong/src/lib.cairo:14:33
    let node = BinaryTree::Node((4, leaf2, leaf3));
                                ^^^^^^^^^^^^^^^^^

error: could not compile `listing_recursive_types_wrong` due to previous error

```

错误显示此类型“具有无限大小”。原因是我们将 `BinaryTree` 定义为具有一个递归的变体：它直接持有自身的另一个值。结果，Cairo 无法弄清楚存储 `BinaryTree` 值需要多少空间。

希望我们可以通过使用 `Box<T>` 来存储 `BinaryTree` 的递归变体来修复此错误。因为 `Box<T>` 是一个指针，Cairo 总是知道 `Box<T>` 需要多少空间：指针的大小不会根据它指向的数据量而改变。这意味着我们可以将 `Box<T>` 放在 `Node` 变体中，而不是直接放另一个 `BinaryTree` 值。`Box<T>` 将指向将存储在它们自己段中的子 `BinaryTree` 值，而不是在 `Node` 变体内。从概念上讲，我们仍然有一个二叉树，由持有其他二叉树的二叉树创建，但此实现现在更像是将项目彼此并排放置，而不是彼此嵌套。

我们可以将清单 {{#ref recursive_types_wrong}} 中 `BinaryTree` 枚举的定义和清单 {{#ref recursive_types_wrong}} 中 `BinaryTree` 的使用更改为清单 {{#ref recursive_types}} 中的代码，这将可以编译：

```cairo
mod display;
use display::DebugBinaryTree;

#[derive(Copy, Drop)]
enum BinaryTree {
    Leaf: u32,
    Node: (u32, Box<BinaryTree>, Box<BinaryTree>),
}


#[executable]
fn main() {
    let leaf1 = BinaryTree::Leaf(1);
    let leaf2 = BinaryTree::Leaf(2);
    let leaf3 = BinaryTree::Leaf(3);
    let node = BinaryTree::Node((4, BoxTrait::new(leaf2), BoxTrait::new(leaf3)));
    let root = BinaryTree::Node((5, BoxTrait::new(leaf1), BoxTrait::new(node)));

    println!("{:?}", root);
}
```

{{#label recursive_types}} <span class="caption">清单 {{#ref recursive_types}}: 使用 Boxes 定义递归二叉树</span>

`Node` 变体现在持有 `(u32, Box<BinaryTree>, Box<BinaryTree>)`，表示 `Node` 变体将存储一个 `u32` 值和两个 `Box<BinaryTree>` 值。现在，我们知道 `Node` 变体将需要 `u32` 的大小加上两个 `Box<BinaryTree>` 值的大小。通过使用 box，我们打破了无限的递归链，因此编译器可以计算出存储 `BinaryTree` 值所需的大小。

### 使用 Boxes 提高性能

在函数之间传递指针允许你引用数据而无需复制数据本身。使用 boxes 可以提高性能，因为它允许你将指向某些数据的指针从一个函数传递到另一个函数，而无需在执行函数调用之前复制内存中的整个数据。不必在调用函数之前将 `n` 个值写入内存，只需写入对应于数据指针的单个值。如果存储在 box 中的数据非常大，性能提升可能会很显著，因为你会在每次函数调用前节省 `n-1` 次内存操作。

> 注意：这仅适用于存储在 box 中的数据未被变异（mutated）的情况。如果数据被变异，将创建一个新的 `Box<T>`，这将需要将数据复制到新 box 中。

让我们看看清单 {{#ref box}} 中的代码，它展示了向函数传递数据的两种方式：按值和按指针。

```cairo
#[derive(Drop)]
struct Cart {
    paid: bool,
    items: u256,
    buyer: ByteArray,
}

fn pass_data(cart: Cart) {
    println!("{} is shopping today and bought {} items", cart.buyer, cart.items);
}

fn pass_pointer(cart: Box<Cart>) {
    let cart = cart.unbox();
    println!("{} is shopping today and bought {} items", cart.buyer, cart.items);
}

#[executable]
fn main() {
    let new_struct = Cart { paid: true, items: 1, buyer: "Eli" };
    pass_data(new_struct);

    let new_box = BoxTrait::new(Cart { paid: false, items: 2, buyer: "Uri" });
    pass_pointer(new_box);
}
```

{{#label box}} <span class="caption">清单 {{#ref box}}: 将大量数据存储在 box 中以提高性能。</span>

`main` 函数包含 2 个函数调用：

- `pass_data` 接受一个 `Cart` 类型的变量。
- `pass_pointer` 接受一个 `Box<Cart>` 类型的指针。

当向函数传递数据时，整个数据会在紧接函数调用之前被复制到最后可用的内存单元中。调用 `pass_data` 将把 `Cart` 的所有 3 个字段复制到内存中，而 `pass_pointer` 仅需要复制大小为 1 的 `new_box` 指针。

<div align="center">
    <img src="box_memory.png" alt="box memory" width="500px"/>
<div align="center">
    </div>
    <span class="caption">使用 boxes 时的 CairoVM 内存布局</span>
</div>

上面的插图演示了内存在两种情况下的行为。`Cart` 的第一个实例存储在执行段中，我们需要在调用 `pass_data` 函数之前将其所有字段复制到内存中。`Cart` 的第二个实例存储在 boxed segment 中，指向它的指针存储在执行段中。当调用 `pass_pointer` 函数时，只有指向结构体的指针在紧接函数调用之前被复制到内存中。然而，在这两种情况下，实例化结构体都会将其所有值存储在执行段中：boxed segment 只能用取自执行段的数据填充。

## 字典的 `Nullable<T>` 类型

`Nullable<T>` 是另一种类型的智能指针，它可以指向一个值，或者在没有值的情况下为 `null`。它是在 Sierra 级别定义的。此类型主要用于字典，这些字典包含不实现 `Felt252DictValue<T>` trait 的 `zero_default` 方法的类型（即数组和结构体）。

如果我们尝试访问字典中不存在的元素，如果无法调用 `zero_default` 方法，代码将失败。

关于字典的 [第 {{#chap common-collections}} 章][dictionary nullable span] 彻底解释了如何使用 `Nullable<T>` 类型在字典内存储 `Span<felt252>` 变量。更多信息请参考该章节。

[dictionary nullable span]: ./ch03-02-dictionaries.md#dictionaries-of-types-not-supported-natively

{{#quiz ../quizzes/ch12-02-smart_pointers.toml}}
