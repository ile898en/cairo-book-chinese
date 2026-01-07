# 字典

Cairo 在其核心库中提供了一种类似字典的类型。`Felt252Dict<T>` 数据类型表示键值对的集合，其中每个键都是唯一的，并与相应的值相关联。这种数据结构在不同的编程语言中有不同的名称，如映射 (maps)、哈希表 (hash tables)、关联数组 (associative arrays) 等。

`Felt252Dict<T>` 类型在你想要以某种方式组织数据，而使用 `Array<T>` 和索引不足以应付时非常有用。Cairo 字典还允许程序员在没有可变内存的情况下轻松模拟可变内存的存在。

## 字典的基本用法

在其他语言中，创建新字典时通常要定义键和值的数据类型。在 Cairo 中，键类型被限制为 `felt252`，只留下了指定值数据类型的可能性，由 `Felt252Dict<T>` 中的 `T` 表示。

`Felt252Dict<T>` 的核心功能是在 `Felt252DictTrait` trait 中实现的，其中包括所有基本操作。其中我们可以找到：

1. `insert(felt252, T) -> ()` 用于向字典实例写入值，以及
2. `get(felt252) -> T` 用于从中读取值。

这些函数允许我们像在任何其他语言中一样操作字典。在下面的示例中，我们创建一个字典来表示个人与其余额之间的映射：

```cairo
use core::dict::Felt252Dict;

#[executable]
fn main() {
    let mut balances: Felt252Dict<u64> = Default::default();

    balances.insert('Alex', 100);
    balances.insert('Maria', 200);

    let alex_balance = balances.get('Alex');
    assert!(alex_balance == 100, "Balance is not 100");

    let maria_balance = balances.get('Maria');
    assert!(maria_balance == 200, "Balance is not 200");
}
```

我们可以通过使用 `Default` trait 的 `default` 方法创建一个 `Felt252Dict<u64>` 的新实例，并使用 `insert` 方法添加两个人，每个人都有自己的余额。最后，我们使用 `get` 方法检查用户的余额。这些方法定义在核心库的 `Felt252DictTrait` trait 中。

在整本书中，我们要一直在谈论 Cairo 的内存是如何不可变的，这意味着你只能向内存单元写入一次，但 `Felt252Dict<T>` 类型代表了一种克服这一障碍的方法。我们稍后将在 [“底层字典”][dict underneath] 中解释这是如何实现的。

基于我们之前的例子，让我们展示一个同一个用户的余额发生变化的代码示例：

```cairo
use core::dict::Felt252Dict;

#[executable]
fn main() {
    let mut balances: Felt252Dict<u64> = Default::default();

    // Insert Alex with 100 balance
    balances.insert('Alex', 100);
    // Check that Alex has indeed 100 associated with him
    let alex_balance = balances.get('Alex');
    assert!(alex_balance == 100, "Alex balance is not 100");

    // Insert Alex again, this time with 200 balance
    balances.insert('Alex', 200);
    // Check the new balance is correct
    let alex_balance_2 = balances.get('Alex');
    assert!(alex_balance_2 == 200, "Alex balance is not 200");
}
```

注意在这个例子中我们是如何两次添加 'Alex' 个人的，每次使用不同的余额，并且每次我们检查其余额时，它都有最后插入的值！`Felt252Dict<T>` 有效地允许我们“重写”任何给定键的存储值。

在继续解释字典是如何实现的之前，值得一提的是，一旦你实例化一个 `Felt252Dict<T>`，在幕后所有的键都有其关联的值初始化为零。这意味着如果例如，你试图获取一个不存在的用户的余额，你会得到 0 而不是错误或未定义的值。这也意味着没有办法从字典中删除数据。这是将此结构纳入你的代码时需要考虑的事情。

到目前为止，我们已经看到了 `Felt252Dict<T>` 的所有基本特性，以及它如何模仿任何其他语言中相应数据结构的相同行为，当然是在表面上。Cairo 本质上是一种非确定性图灵完备编程语言，与现存的任何其他流行语言都非常不同，因此这意味着字典的实现也非常不同！

在接下来的部分中，我们将给出一些关于 `Felt252Dict<T>` 内部机制以及为使其工作而做出的妥协的见解。之后，我们将看看如何将字典与其他数据结构一起使用，以及使用 `entry` 方法作为与它们交互的另一种方式。

[dict underneath]: ./ch03-02-dictionaries.md#dictionaries-underneath

## 底层字典 (Dictionaries Underneath)

Cairo 非确定性设计的约束之一是其内存系统是不可变的，因此为了模拟可变性，该语言将 `Felt252Dict<T>` 实现为条目（entries）列表。每个条目代表字典被访问以进行读取/更新/写入目的的时间。一个条目有三个字段：

1. 一个 `key` 字段，标识此字典键值对的键。
2. 一个 `previous_value` 字段，指示先前在 `key` 处保存的值。
3. 一个 `new_value` 字段，指示在 `key` 处保存的新值。

如果我们尝试使用高级结构实现 `Felt252Dict<T>`，这种内部定义为 `Array<Entry<T>>`，其中每个 `Entry<T>` 都有关于它代表什么键值对以及它保存的先前和新值的信息。`Entry<T>` 的定义将是：

```cairo,noplayground
struct Entry<T> {
    key: felt252,
    previous_value: T,
    new_value: T,
}
```

每当我们与 `Felt252Dict<T>` 交互时，都会注册一个新的 `Entry<T>`：

- `get` 会注册一个状态没有变化的条目，并且先前和新值存储相同的值。
- `insert` 会注册一个新的 `Entry<T>`，其中 `new_value` 将是被插入的元素，而 `previous_value` 是在此之前最后插入的元素。如果它是某个键的第一个条目，那么先前的值将是零。

使用这个条目列表表明没有任何重写，只是每次 `Felt252Dict<T>` 交互都创建新的内存单元。让我们用上一节的 `balances` 字典并插入用户 'Alex' 和 'Maria' 来展示这方面的一个例子：

```cairo
    balances.insert('Alex', 100_u64);
    balances.insert('Maria', 50_u64);
    balances.insert('Alex', 200_u64);
    balances.get('Maria');
```

这些指令将产生以下条目列表：

|  key  | previous | new |
| :---: | -------- | --- |
| Alex  | 0        | 100 |
| Maria | 0        | 50  |
| Alex  | 100      | 200 |
| Maria | 50       | 50  |

注意，因为 'Alex' 被插入了两次，它出现了两次，并且 `previous` 和 `current` 值被正确设置。另外从 'Maria' 读取也注册了一个先前到当前值没有变化的条目。

这种实现 `Felt252Dict<T>` 的方法意味着对于每个读/写操作，都会扫描整个条目列表以查找具有相同 `key` 的最后一个条目。一旦找到条目，它的 `new_value` 将被提取并用作要添加的新条目的 `previous_value`。这意味着与 `Felt252Dict<T>` 交互具有 `O(n)` 的最坏情况时间复杂度，其中 `n` 是列表中的条目数。

如果你仔细思考实现 `Felt252Dict<T>` 的替代方法，你肯定会找到它们，甚至可能完全放弃对 `previous_value` 字段的需求，尽管如此，由于 Cairo 不是你的普通语言，这行不通。Cairo 的目的之一是，使用 STARK 证明系统生成计算完整性的证明。这意味着你需要验证程序执行是正确的并且在 Cairo 限制的边界内。这些边界检查之一包括“字典压缩 (dictionary squashing)”，这需要每个条目的先前值和新值的信息。

## 压缩字典 (Squashing Dictionaries)

为了验证使用 `Felt252Dict<T>` 的 Cairo 程序执行生成的证明是正确的，我们需要检查字典没有被非法篡改。这是通过一种名为 `squash_dict` 的方法完成的，该方法审查条目列表的每个条目，并检查对字典的访问在整个执行过程中保持连贯。

压缩的过程如下：给定所有具有特定键 `k` 的条目，按它们被插入的顺序排列，验证第 i 个条目的 `new_value` 是否等于第 i + 1 个条目的 `previous_value`。

例如，给定以下条目列表：

|   key   | previous | new |
| :-----: | -------- | --- |
|  Alex   | 0        | 150 |
|  Maria  | 0        | 100 |
| Charles | 0        | 70  |
|  Maria  | 100      | 250 |
|  Alex   | 150      | 40  |
|  Alex   | 40       | 300 |
|  Maria  | 250      | 190 |
|  Alex   | 300      | 90  |

压缩后，条目列表将减少为：

|   key   | previous | new |
| :-----: | -------- | --- |
|  Alex   | 0        | 90  |
|  Maria  | 0        | 190 |
| Charles | 0        | 70  |

如果在第一个表中的任何值上有变化，压缩将在运行时失败。

## 字典销毁 (Dictionary Destruction)

如果你运行 [“字典的基本用法”][basic dictionaries] 部分的示例，你会注意到从未调用过字典压缩，但程序还是成功编译了。幕后发生的事情是，通过 `Destruct<T>` trait 的 `Felt252Dict<T>` 实现自动调用了压缩。此调用发生在 `balance` 字典超出作用域之前。

`Destruct<T>` trait 代表了除了 `Drop<T>` 之外将实例移出作用域的另一种方式。这两者之间的主要区别是 `Drop<T>` 被视为无操作 (no-op)，这意味着它不生成新的 CASM，而 `Destruct<T>` 没有此限制。唯一主动使用 `Destruct<T>` trait 的类型是 `Felt252Dict<T>`，对于每种其他类型，`Destruct<T>` 和 `Drop<T>` 是同义词。你可以在附录 C 的 [Drop 和 Destruct][drop destruct] 部分阅读更多关于这些 trait 的信息。

稍后在 [“作为结构体成员的字典”][dictionaries in structs] 部分，我们将有一个实践示例，我们将为自定义类型实现 `Destruct<T>` trait。

[basic dictionaries]: ./ch03-02-dictionaries.md#basic-use-of-dictionaries
[drop destruct]: ./appendix-03-derivable-traits.md#drop-and-destruct
[dictionaries in structs]: ./ch12-01-custom-data-structures.md#dictionaries-as-struct-members

## 更多字典

到目前为止，我们已经对 `Felt252Dict<T>` 的功能以及它为何以某种方式实现进行了全面的概述。如果你没有完全理解它，别担心，因为在本节中我们将有更多使用字典的例子。

我们将从解释 `entry` 方法开始，该方法是包含在 `Felt252DictTrait<T>` 中的字典基本功能的一部分，我们在开始时没有提到。之后不久，我们将看到如何将 `Felt252Dict<T>` 与其他 [复杂类型][nullable dictionaries values] 一起使用的例子，例如 `Array<T>`。

[nullable dictionaries values]: ./ch03-02-dictionaries.md#dictionaries-of-types-not-supported-natively

## Entry 和 Finalize

在 [“底层字典”][dict underneath] 部分，我们解释了 `Felt252Dict<T>` 在内部是如何工作的。它是每次以任何方式访问字典时的条目列表。它首先会根据给定的特定 `key` 找到最后一个条目，然后根据它正在执行的任何操作对其进行相应的更新。Cairo 语言为我们提供了通过 `entry` 和 `finalize` 方法自己复制此操作的工具。

`entry` 方法作为 `Felt252DictTrait<T>` 的一部分出现，目的是根据给定键创建一个新条目。一旦调用，此方法获取字典的所有权并返回要更新的条目。方法签名如下：

```cairo,noplayground
fn entry(self: Felt252Dict<T>, key: felt252) -> (Felt252DictEntry<T>, T) nopanic
```

第一个输入参数获取字典的所有权，而第二个参数用于创建适当的条目。它返回一个元组，其中包含 `Felt252DictEntry<T>`（Cairo 用于表示字典条目的类型）和一个表示先前保存的值的 `T`。`nopanic` 符号简单地表示该函数保证永远不会 panic。

接下来要做的是用新值更新条目。为此，我们使用 `finalize` 方法，该方法插入条目并返回字典的所有权：

```cairo,noplayground
fn finalize(self: Felt252DictEntry<T>, new_value: T) -> Felt252Dict<T>
```

此方法接收条目和新值作为参数，并返回更新后的字典。

让我们看一个使用 `entry` 和 `finalize` 的例子。想象一下我们想实现我们自己的字典 `get` 方法版本。那么我们应该做以下事情：

1. 使用 `entry` 方法创建要添加的新条目。
2. 将 `new_value` 等于 `previous_value` 的条目插回。
3. 返回该值。

实现我们的自定义 get 看起来像这样：

```cairo,noplayground
use core::dict::{Felt252Dict, Felt252DictEntryTrait};

fn custom_get<T, +Felt252DictValue<T>, +Drop<T>, +Copy<T>>(
    ref dict: Felt252Dict<T>, key: felt252,
) -> T {
    // Get the new entry and the previous value held at `key`
    let (entry, prev_value) = dict.entry(key);

    // Store the value to return
    let return_value = prev_value;

    // Update the entry with `prev_value` and get back ownership of the dictionary
    dict = entry.finalize(prev_value);

    // Return the read value
    return_value
}
```

`ref` 关键字意味着变量的所有权将在函数结束时归还。这个概念将在 [“引用和快照”][references] 部分更详细地解释。

实现 `insert` 方法将遵循类似的工作流程，除了在定稿 (finalize) 时插入新值。如果我们实现它，它看起来像下面这样：

```cairo,noplayground
use core::dict::{Felt252Dict, Felt252DictEntryTrait};

fn custom_insert<T, +Felt252DictValue<T>, +Destruct<T>, +Drop<T>>(
    ref dict: Felt252Dict<T>, key: felt252, value: T,
) {
    // Get the last entry associated with `key`
    // Notice that if `key` does not exist, `_prev_value` will
    // be the default value of T.
    let (entry, _prev_value) = dict.entry(key);

    // Insert `entry` back in the dictionary with the updated value,
    // and receive ownership of the dictionary
    dict = entry.finalize(value);
}
```

作为最后的说明，这两个方法的实现方式与 `Felt252Dict<T>` 的 `insert` 和 `get` 的实现方式类似。此代码显示了一些用法示例：

```cairo
#[executable]
fn main() {
    let mut dict: Felt252Dict<u64> = Default::default();

    custom_insert(ref dict, '0', 100);

    let val = custom_get(ref dict, '0');

    assert!(val == 100, "Expecting 100");
}
```

[dict underneath]: ./ch03-02-dictionaries.md#dictionaries-underneath
[references]: ./ch04-02-references-and-snapshots.md

## 原生不支持类型的字典

我们没有谈到的 `Felt252Dict<T>` 的一个限制是 `Felt252DictValue<T>` trait。此 trait 定义了 `zero_default` 方法，当字典中不存在值时会调用该方法。这是一些常见数据类型（如大多数无符号整数、`bool` 和 `felt252`）实现的——但它没有为更复杂的类型（如数组、结构体（包括 `u256`）和核心库中的其他类型）实现。这意味着制作原生不支持类型的字典不是一项简单的任务，因为你需要编写几个 trait 实现才能使数据类型成为有效的字典值类型。为了弥补这一点，你可以将你的类型包装在 `Nullable<T>` 中。

`Nullable<T>` 是一个智能指针类型，在没有值的情况下，它可以指向一个值或是 `null`。它通常在面向对象编程语言中当引用不指向任何地方时使用。与 `Option` 的区别在于包装的值存储在 `Box<T>` 数据类型中。`Box<T>` 类型是一个智能指针，允许我们为数据使用专用的 `boxed_segment` 内存段，并使用一次只能在一个地方操作的指针访问此段。有关更多信息，请参阅 [智能指针章节](./ch12-02-smart-pointers.md)。

让我们用一个例子来展示。我们将尝试在字典中存储一个 `Span<felt252>`。为此，我们将使用 `Nullable<T>` 和 `Box<T>`。此外，我们存储的是 `Span<T>` 而不是 `Array<T>`，因为后者没有实现从字典读取所需的 `Copy<T>` trait。

```cairo,noplayground
use core::dict::Felt252Dict;
use core::nullable::{FromNullableResult, NullableTrait, match_nullable};

#[executable]
fn main() {
    // Create the dictionary
    let mut d: Felt252Dict<Nullable<Span<felt252>>> = Default::default();

    // Create the array to insert
    let a = array![8, 9, 10];

    // Insert it as a `Span`
    d.insert(0, NullableTrait::new(a.span()));

//...
```

在此代码片段中，我们做的第一件事是创建一个新字典 `d`。我们要它保存一个 `Nullable<Span>`。在那之后，我们创建了一个数组并填入了值。

最后一步是将数组作为 span 插入字典中。注意我们使用 `NullableTrait` 的 `new` 函数来执行此操作。

一旦元素在字典中，并且我们要获取它，我们遵循相同的步骤但顺序相反。以下代码显示了如何实现这一点：

```cairo,noplayground
//...

    // Get value back
    let val = d.get(0);

    // Search the value and assert it is not null
    let span = match match_nullable(val) {
        FromNullableResult::Null => panic!("No value found"),
        FromNullableResult::NotNull(val) => val.unbox(),
    };

    // Verify we are having the right values
    assert!(*span.at(0) == 8, "Expecting 8");
    assert!(*span.at(1) == 9, "Expecting 9");
    assert!(*span.at(2) == 10, "Expecting 10");
}
```

在这里我们：

1. 使用 `get` 读取值。
2. 使用 `match_nullable` 函数验证它非 null。
3. 解包 box 内的值并断言它是正确的。

完整的脚本看起来像这样：

```cairo
// ANCHOR: imports
use core::dict::Felt252Dict;
use core::nullable::{FromNullableResult, NullableTrait, match_nullable};
// ANCHOR_END: imports

// ANCHOR: header
#[executable]
fn main() {
    // Create the dictionary
    let mut d: Felt252Dict<Nullable<Span<felt252>>> = Default::default();

    // Create the array to insert
    let a = array![8, 9, 10];

    // Insert it as a `Span`
    d.insert(0, NullableTrait::new(a.span()));
    // ANCHOR_END: header

    // ANCHOR: footer
    // Get value back
    let val = d.get(0);

    // Search the value and assert it is not null
    let span = match match_nullable(val) {
        FromNullableResult::Null => panic!("No value found"),
        FromNullableResult::NotNull(val) => val.unbox(),
    };

    // Verify we are having the right values
    assert!(*span.at(0) == 8, "Expecting 8");
    assert!(*span.at(1) == 9, "Expecting 9");
    assert!(*span.at(2) == 10, "Expecting 10");
}
// ANCHOR_END: footer

```

## 在字典内使用数组

在上一节中，我们探索了如何使用 `Nullable<T>` 和 `Box<T>` 在字典中存储和检索复杂类型。现在，让我们看看如何在字典中存储数组并动态修改其内容。

在 Cairo 的字典中存储数组与存储其他类型略有不同。这是因为数组是更复杂的数据结构，需要特殊处理以避免内存复制和引用问题。

首先，让我们看看如何创建一个字典并将数组插入其中。这个过程非常直接，遵循与插入其他类型数据类似的模式：

```cairo
use core::dict::Felt252Dict;

#[executable]
fn main() {
    let arr = array![20, 19, 26];
    let mut dict: Felt252Dict<Nullable<Array<u8>>> = Default::default();
    dict.insert(0, NullableTrait::new(arr));
    println!("Array inserted successfully.");
}
```

然而，试图使用 `get` 方法从字典中读取数组将导致编译器错误。这是因为 `get` 试图在内存中复制数组，这对数组来说是不可能的（正如我们已经在 [上一节][nullable dictionaries values] 中提到的，`Array<T>` 没有实现 `Copy<T>` trait）：

```cairo
//TAG: does_not_compile
use core::dict::Felt252Dict;
use core::nullable::{FromNullableResult, match_nullable};

#[executable]
fn main() {
    let arr = array![20, 19, 26];
    let mut dict: Felt252Dict<Nullable<Array<u8>>> = Default::default();
    dict.insert(0, NullableTrait::new(arr));
    println!("Array: {:?}", get_array_entry(ref dict, 0));
}

fn get_array_entry(ref dict: Felt252Dict<Nullable<Array<u8>>>, index: felt252) -> Span<u8> {
    let val = dict.get(0); // This will cause a compiler error
    let arr = match match_nullable(val) {
        FromNullableResult::Null => panic!("No value!"),
        FromNullableResult::NotNull(val) => val.unbox(),
    };
    arr.span()
}
```

```shell
$ scarb execute 
   Compiling no_listing_15_dict_of_array_attempt_get v0.1.0 (listings/ch03-common-collections/no_listing_15_dict_of_array_attempt_get/Scarb.toml)
error: Trait has no implementation in context: core::traits::Copy::<core::nullable::Nullable::<core::array::Array::<core::integer::u8>>>.
 --> listings/ch03-common-collections/no_listing_15_dict_of_array_attempt_get/src/lib.cairo:14:20
    let val = dict.get(0); // This will cause a compiler error
                   ^^^

error: could not compile `no_listing_15_dict_of_array_attempt_get` due to previous error
error: `scarb` command exited with error

```

为了正确地从字典中读取数组，我们需要使用字典条目。这允许我们获取数组值的引用而不复制它：

```cairo,noplayground
fn get_array_entry(ref dict: Felt252Dict<Nullable<Array<u8>>>, index: felt252) -> Span<u8> {
    let (entry, _arr) = dict.entry(index);
    let mut arr = _arr.deref_or(array![]);
    let span = arr.span();
    dict = entry.finalize(NullableTrait::new(arr));
    span
}
```

> 注意：我们必须在 finalize 条目之前将数组转换为 `Span`，因为调用 `NullableTrait::new(arr)` 会移动数组，从而使其无法从函数返回。

要修改存储的数组，例如追加一个新值，我们可以使用类似的方法。下面的 `append_value` 函数演示了这一点：

```cairo,noplayground
fn append_value(ref dict: Felt252Dict<Nullable<Array<u8>>>, index: felt252, value: u8) {
    let (entry, arr) = dict.entry(index);
    let mut unboxed_val = arr.deref_or(array![]);
    unboxed_val.append(value);
    dict = entry.finalize(NullableTrait::new(unboxed_val));
}
```

在 `append_value` 函数中，我们访问字典条目，解引用数组，追加新值，并用更新的数组 finalize 条目。

> 注意：从存储的数组中移除项目可以用类似的方式实现。

下面是演示在字典中创建、插入、读取和修改数组的完整示例：

```cairo
use core::dict::{Felt252Dict, Felt252DictEntryTrait};
use core::nullable::NullableTrait;

//ANCHOR: append
fn append_value(ref dict: Felt252Dict<Nullable<Array<u8>>>, index: felt252, value: u8) {
    let (entry, arr) = dict.entry(index);
    let mut unboxed_val = arr.deref_or(array![]);
    unboxed_val.append(value);
    dict = entry.finalize(NullableTrait::new(unboxed_val));
}
//ANCHOR_END: append

//ANCHOR: get
fn get_array_entry(ref dict: Felt252Dict<Nullable<Array<u8>>>, index: felt252) -> Span<u8> {
    let (entry, _arr) = dict.entry(index);
    let mut arr = _arr.deref_or(array![]);
    let span = arr.span();
    dict = entry.finalize(NullableTrait::new(arr));
    span
}
//ANCHOR_END: get

#[executable]
fn main() {
    let arr = array![20, 19, 26];
    let mut dict: Felt252Dict<Nullable<Array<u8>>> = Default::default();
    dict.insert(0, NullableTrait::new(arr));
    println!("Before insertion: {:?}", get_array_entry(ref dict, 0));

    append_value(ref dict, 0, 30);

    println!("After insertion: {:?}", get_array_entry(ref dict, 0));
}
```

{{#quiz ../quizzes/ch03-02-dictionaries.toml}}
