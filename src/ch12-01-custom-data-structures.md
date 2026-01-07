# 自定义数据结构

当你第一次开始用 Cairo 编程时，你可能会想用数组 (`Array<T>`) 来存储数据集合。然而，你很快就会意识到数组有一个很大的限制 —— 存储在其中的数据是不可变的。一旦向数组追加了一个值，就不能再修改它。

当你想要使用可变数据结构时，这可能会令人沮丧。例如，假设你正在制作一个游戏，玩家有等级，并且他们可以升级。你可能会尝试将玩家的等级存储在一个数组中：

```cairo,noplayground
    let mut level_players = array![5, 1, 10];
```

但是随后你意识到，一旦设置好，你就无法增加特定索引处的等级。如果玩家死亡，你也无法将其从数组中移除，除非他碰巧在第一个位置。

幸运的是，Cairo 提供了一个方便的内置 [字典类型](./ch03-02-dictionaries.md) 叫做 `Felt252Dict<T>`，它允许我们模拟可变数据结构的行为。让我们首先探索如何创建一个包含 `Felt252Dict<T>` 等成员的结构体。

> 注意：本章中使用的几个概念在本书前面已经介绍过。如果你需要复习，我们建议查看以下章节：[结构体](ch05-00-using-structs-to-structure-related-data.md)、[方法](./ch05-03-method-syntax.md)、[泛型类型](./ch08-00-generic-types-and-traits.md)、[Traits](./ch08-02-traits-in-cairo.md)。

## 字典作为结构体成员

在 Cairo 中可以将字典定义为结构体成员，但正确地与它们交互可能并不完全无缝。让我们尝试实现一个自定义的 _用户数据库_，它将允许我们添加用户并查询他们。我们需要定义一个结构体来表示新类型，并定义一个 trait 来定义其功能：

```cairo,noplayground
struct UserDatabase<T> {
    users_updates: u64,
    balances: Felt252Dict<T>,
}

trait UserDatabaseTrait<T> {
    fn new() -> UserDatabase<T>;
    fn update_user<+Drop<T>>(ref self: UserDatabase<T>, name: felt252, balance: T);
    fn get_balance<+Copy<T>>(ref self: UserDatabase<T>, name: felt252) -> T;
}
```

我们的新类型 `UserDatabase<T>` 表示一个用户数据库。它在用户的余额上是泛型的，为使用我们数据类型的任何人提供了极大的灵活性。它的两个成员是：

- `users_updates`，字典中用户更新的次数。
- `balances`，每个用户到其余额的映射。

数据库的核心功能由 `UserDatabaseTrait` 定义。定义了以下方法：

- `new` 用于轻松创建新的 `UserDatabase` 类型。
- `update_user` 用于更新数据库中用户的余额。
- `get_balance` 用于查找数据库中用户的余额。

剩下的唯一步骤是实现 `UserDatabaseTrait` 中的每个方法，但由于我们正在处理 [泛型类型](./ch08-00-generic-types-and-traits.md)，我们还需要正确建立 `T` 的要求，以便它可以是一个有效的 `Felt252Dict<T>` 值类型：

1. `T` 应该实现 `Copy<T>`，因为从 `Felt252Dict<T>` 获取值需要它。
2. 字典的所有值类型都实现了 `Felt252DictValue<T>`，我们的泛型类型也应该这样做。
3. 要插入值，`Felt252DictTrait<T>` 要求所有值类型都是可丢弃的（实现 `Drop<T>` trait）。

在所有限制到位的情况下，实现将如下所示：

```cairo,noplayground
impl UserDatabaseImpl<T, +Felt252DictValue<T>> of UserDatabaseTrait<T> {
    // Creates a database
    fn new() -> UserDatabase<T> {
        UserDatabase { users_updates: 0, balances: Default::default() }
    }

    // Get the user's balance
    fn get_balance<+Copy<T>>(ref self: UserDatabase<T>, name: felt252) -> T {
        self.balances.get(name)
    }

    // Add a user
    fn update_user<+Drop<T>>(ref self: UserDatabase<T>, name: felt252, balance: T) {
        self.balances.insert(name, balance);
        self.users_updates += 1;
    }
}
```

我们的数据库实现几乎完成了，只有一件事除外：编译器不知道如何让 `UserDatabase<T>` 离开作用域，因为它没有实现 `Drop<T>` trait，也没有实现 `Destruct<T>` trait。因为它有一个 `Felt252Dict<T>` 作为成员，它不能被丢弃，所以我们被迫手动实现 `Destruct<T>` trait（有关更多信息，请参阅 [所有权](ch04-01-what-is-ownership.md#the-drop-trait) 章节）。在 `UserDatabase<T>` 定义之上使用 `#[derive(Destruct)]` 将不起作用，因为结构体定义中使用了 [泛型类型][generics]。我们需要自己编写 `Destruct<T>` trait 实现：

```cairo,noplayground
impl UserDatabaseDestruct<T, +Drop<T>, +Felt252DictValue<T>> of Destruct<UserDatabase<T>> {
    fn destruct(self: UserDatabase<T>) nopanic {
        self.balances.squash();
    }
}
```

为 `UserDatabase` 实现 `Destruct<T>` 是我们获得功能齐全的数据库的最后一步。我们现在可以试用它：

```cairo
#[executable]
fn main() {
    let mut db = UserDatabaseTrait::<u64>::new();

    db.update_user('Alex', 100);
    db.update_user('Maria', 80);

    db.update_user('Alex', 40);
    db.update_user('Maria', 0);

    let alex_latest_balance = db.get_balance('Alex');
    let maria_latest_balance = db.get_balance('Maria');

    assert!(alex_latest_balance == 40, "Expected 40");
    assert!(maria_latest_balance == 0, "Expected 0");
}
```

[generics]: ./ch08-00-generic-types-and-traits.md

## 使用字典模拟动态数组

首先，让我们考虑一下我们要让我们的可变动态数组表现得如何。它应该支持哪些操作？

它应该：

- 允许我们在末尾追加项目。
- 让我们通过索引访问任何项目。
- 允许设置特定索引处的项目值。
- 返回当前长度。

我们可以在 Cairo 中这样定义这个接口：

```cairo,noplayground
trait MemoryVecTrait<V, T> {
    fn new() -> V;
    fn get(ref self: V, index: usize) -> Option<T>;
    fn at(ref self: V, index: usize) -> T;
    fn push(ref self: V, value: T) -> ();
    fn set(ref self: V, index: usize, value: T);
    fn len(self: @V) -> usize;
}
```

这为我们的动态数组的实现提供了蓝图。我们将其命名为 _MemoryVec_，因为它类似于 Rust 中的 `Vec<T>` 数据结构。

> 注意：Cairo 的核心库已经包含了一个 `Vec<T>` 数据结构，严格用作智能合约中的存储类型。为了将我们的数据结构与核心库的数据结构区分开来，我们将我们的实现命名为 _MemoryVec_。

### 在 Cairo 中实现动态数组

为了存储我们的数据，我们将使用 `Felt252Dict<T>`，它将索引号 (felts) 映射到值。我们还将存储一个单独的 `len` 字段来跟踪长度。

这是我们的结构体的样子。我们将类型 `T` 包装在 `Nullable` 指针内，以允许在我们的数据结构中使用任何类型 `T`，如 [字典][nullable] 部分所述：

```cairo,noplayground
struct MemoryVec<T> {
    data: Felt252Dict<Nullable<T>>,
    len: usize,
}
```

因为我们再次拥有 `Felt252Dict<T>` 作为结构体成员，我们需要实现 `Destruct<T>` trait 来告诉编译器如何让 `MemoryVec<T>` 离开作用域。

```cairo,noplayground
impl DestructMemoryVec<T, +Drop<T>> of Destruct<MemoryVec<T>> {
    fn destruct(self: MemoryVec<T>) nopanic {
        self.data.squash();
    }
}
```

使这个向量可变的关键是我们可以在字典中插入值来设置或更新我们数据结构中的值。例如，要更新特定索引处的值，我们这样做：

```cairo,noplayground
    fn set(ref self: MemoryVec<T>, index: usize, value: T) {
        assert!(index < self.len(), "Index out of bounds");
        self.data.insert(index.into(), NullableTrait::new(value));
    }
```

这将覆盖字典中该索引处先前存在的值。

虽然数组是不可变的，但字典提供了我们需要的灵活性，用于像向量这样的可修改数据结构。

其余接口的实现很简单。我们接口中定义的所有方法的实现可以如下完成：

```cairo,noplayground
impl MemoryVecImpl<T, +Drop<T>, +Copy<T>> of MemoryVecTrait<MemoryVec<T>, T> {
    fn new() -> MemoryVec<T> {
        MemoryVec { data: Default::default(), len: 0 }
    }

    fn get(ref self: MemoryVec<T>, index: usize) -> Option<T> {
        if index < self.len() {
            Some(self.data.get(index.into()).deref())
        } else {
            None
        }
    }

    fn at(ref self: MemoryVec<T>, index: usize) -> T {
        assert!(index < self.len(), "Index out of bounds");
        self.data.get(index.into()).deref()
    }

    fn push(ref self: MemoryVec<T>, value: T) -> () {
        self.data.insert(self.len.into(), NullableTrait::new(value));
        self.len.wrapping_add(1_usize);
    }
    // ANCHOR: set
    fn set(ref self: MemoryVec<T>, index: usize, value: T) {
        assert!(index < self.len(), "Index out of bounds");
        self.data.insert(index.into(), NullableTrait::new(value));
    }
    // ANCHOR_END: set
    fn len(self: @MemoryVec<T>) -> usize {
        *self.len
    }
}
```

`MemoryVec` 结构的完整实现可以在社区维护的库 [Alexandria](https://github.com/keep-starknet-strange/alexandria/blob/main/packages/data_structures/src/vec.cairo) 中找到。

[nullable]: ./ch03-02-dictionaries.md#dictionaries-of-types-not-supported-natively

## 使用字典模拟栈

我们现在将看第二个例子及其实现细节：一个栈 (Stack)。

栈是一个后进先出 (LIFO, Last-In, First-Out) 集合。新元素的插入和现有元素的移除发生在同一端，表示为栈顶。

让我们定义我们需要什么操作来创建一个栈：

- 将一个项目推送到栈顶。
- 从栈顶弹出一个项目。
- 检查栈中是否还有任何元素。

从这些规范中，我们可以定义以下接口：

```cairo,noplayground
trait StackTrait<S, T> {
    fn push(ref self: S, value: T);
    fn pop(ref self: S) -> Option<T>;
    fn is_empty(self: @S) -> bool;
}
```

### 在 Cairo 中实现可变栈

要在 Cairo 中创建栈数据结构，我们可以再次使用 `Felt252Dict<T>` 来存储栈的值，以及一个 `usize` 字段来跟踪栈的长度以便对其进行迭代。

栈结构体定义为：

```cairo,noplayground
struct NullableStack<T> {
    data: Felt252Dict<Nullable<T>>,
    len: usize,
}
```

接下来，让我们看看我们的主要函数 `push` 和 `pop` 是如何实现的。

```cairo,noplayground
impl NullableStackImpl<T, +Drop<T>, +Copy<T>> of StackTrait<NullableStack<T>, T> {
    fn push(ref self: NullableStack<T>, value: T) {
        self.data.insert(self.len.into(), NullableTrait::new(value));
        self.len += 1;
    }

    fn pop(ref self: NullableStack<T>) -> Option<T> {
        if self.is_empty() {
            return None;
        }
        self.len -= 1;
        Some(self.data.get(self.len.into()).deref())
    }

    fn is_empty(self: @NullableStack<T>) -> bool {
        *self.len == 0
    }
}
```

代码使用 `insert` 和 `get` 方法来访问 `Felt252Dict<T>` 中的值。要将元素推送到栈顶，`push` 函数将元素插入到字典的索引 `len` 处，并增加栈的 `len` 字段以跟踪栈顶的位置。要移除值，`pop` 函数减少 `len` 的值以更新栈顶的位置，然后检索位置 `len` 处的最后一个值。

栈的完整实现，以及你可以在代码中使用的更多数据结构，可以在社区维护的 [Alexandria][alexandria data structures] 库的 "data_structures" crate 中找到。

[alexandria data structures]: https://github.com/keep-starknet-strange/alexandria/tree/main/packages/data_structures/src

{{#quiz ../quizzes/ch12-01-custom-structs.toml}}

## 总结

干得好！现在你已经了解了数组、字典甚至自定义数据结构。虽然 Cairo 的内存模型是不可变的，这可能会使实现可变数据结构变得困难，但幸运的是我们可以使用 `Felt252Dict<T>` 类型来模拟可变数据结构。这允许我们实现广泛的对许多应用程序有用的数据结构，有效地隐藏了底层内存模型的复杂性。
