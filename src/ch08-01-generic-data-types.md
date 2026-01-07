# 泛型数据类型

我们使用泛型来为项声明（如结构体和函数）创建定义，然后我们可以将它们与许多不同的具体数据类型一起使用。在 Cairo 中，我们可以在定义函数、结构体、枚举、traits、实现和方法时使用泛型。在本章中，我们将看看如何有效地对所有这些使用泛型类型。

泛型允许我们编写适用于多种类型的可重用代码，从而避免代码重复，同时增强代码的可维护性。

## 泛型函数

使函数泛型化意味着它可以对不同类型进行操作，从而避免了需要多个特定于类型的实现。这就导致了显着的代码减少并增加了代码的灵活性。

在定义使用泛型的函数时，我们将泛型放在函数签名中，我们通常会在那里指定参数和返回值的数据类型。例如，想象我们想要创建一个函数，给定两个项的 `Array`，它将返回最大的那个。如果我们需对不同类型的列表执行此操作，那么我们将不得不每次重新定义该函数。幸运的是，我们可以使用泛型一次性实现该函数，然后继续处理其他任务。

```cairo
// Specify generic type T between the angulars
fn largest_list<T>(l1: Array<T>, l2: Array<T>) -> Array<T> {
    if l1.len() > l2.len() {
        l1
    } else {
        l2
    }
}

#[executable]
fn main() {
    let mut l1 = array![1, 2];
    let mut l2 = array![3, 4, 5];

    // There is no need to specify the concrete type of T because
    // it is inferred by the compiler
    let l3 = largest_list(l1, l2);
}
```

`largest_list` 函数比较两个相同类型的列表，并返回元素较多的那个，并删除另一个。如果你编译前面的代码，你会注意到它会因错误而失败，说没有定义用于删除泛型类型数组的 traits。发生这种情况是因为编译器在执行 `main` 函数时无法保证 `Array<T>` 是可删除的 (droppable)。为了删除 `T` 的数组，编译器必须首先知道如何删除 `T`。这可以通过在 `largest_list` 的函数签名中指定 `T` 必须实现 `Drop` trait 来修复。`largest_list` 的正确函数定义如下：

```cairo
fn largest_list<T, impl TDrop: Drop<T>>(l1: Array<T>, l2: Array<T>) -> Array<T> {
    if l1.len() > l2.len() {
        l1
    } else {
        l2
    }
}
```

新的 `largest_list` 函数在其定义中包含了要求：无论放在那里的泛型类型是什么，它必须是可删除的。这就是我们要说的 _trait 约束 (trait bounds)_。`main` 函数保持不变，编译器足够聪明，可以推断出正在使用哪种具体类型以及它是否实现了 `Drop` trait。

### 泛型类型的约束

在定义泛型类型时，拥有关于它们的信息很有用。知道泛型类型实现了哪些 trait 允许我们在函数的逻辑中更有效地使用它，代价是约束可以与函数一起使用的泛型类型。我们之前通过将 `TDrop` 实现添加为 `largest_list` 的泛型参数的一部分来看到了这一点的示例。虽然添加 `TDrop` 是为了满足编译器的要求，但我们也可以添加约束以使我们的函数逻辑受益。

想象我们有一个某种泛型类型 `T` 的元素列表，我们想在其中找到最小的元素。最初，我们知道对于类型 `T` 的元素是可比较的，它必须实现 `PartialOrd` trait。结果函数将是：

```cairo
// Given a list of T get the smallest one
// The PartialOrd trait implements comparison operations for T
fn smallest_element<T, impl TPartialOrd: PartialOrd<T>>(list: @Array<T>) -> T {
    // This represents the smallest element through the iteration
    // Notice that we use the desnap (*) operator
    let mut smallest = *list[0];

    // The index we will use to move through the list
    let mut index = 1;

    // Iterate through the whole list storing the smallest
    while index < list.len() {
        if *list[index] < smallest {
            smallest = *list[index];
        }
        index = index + 1;
    }

    smallest
}

#[executable]
fn main() {
    let list: Array<u8> = array![5, 3, 10];

    // We need to specify that we are passing a snapshot of `list` as an argument
    let s = smallest_element(@list);
    assert!(s == 3);
}
```

`smallest_element` 函数使用实现 `PartialOrd` trait 的泛型类型 `T`，接受 `Array<T>` 的快照作为参数并返回最小元素的副本。因为参数是 `@Array<T>` 类型，我们不再需要在执行结束时删除它，所以我们不需要为 `T` 也实现 `Drop` trait。那为什么它不能编译呢？

在 `list` 上索引时，结果值是索引元素的快照，除非为 `@T` 实现了 `PartialOrd`，否则我们需要使用 `*` 除快照 (desnap) 元素。`*` 操作需要从 `@T` 复制到 `T`，这意味着 `T` 需要实现 `Copy` trait。将类型 `@T` 的元素复制到 `T` 后，现在有了需要删除的类型 `T` 的变量，这就要求 `T` 也实现 `Drop` trait。我们必须添加 `Drop` 和 `Copy` traits 实现以使函数正确。更新 `smallest_element` 函数后，结果代码将是：

```cairo
fn smallest_element<T, impl TPartialOrd: PartialOrd<T>, impl TCopy: Copy<T>, impl TDrop: Drop<T>>(
    list: @Array<T>,
) -> T {
    let mut smallest = *list[0];
    let mut index = 1;

    while index < list.len() {
        if *list[index] < smallest {
            smallest = *list[index];
        }
        index = index + 1;
    }

    smallest
}
```

### 匿名泛型实现参数 (`+` 运算符)

到目前为止，我们总是为每个需要的泛型 trait 实现指定一个名称：`TPartialOrd` 对应 `PartialOrd<T>`，`TDrop` 对应 `Drop<T>`，`TCopy` 对应 `Copy<T>`。

然而，大多数时候，我们在函数​​体内不使用实现；我们只将其用作约束。在这些情况下，我们可以使用 `+` 运算符来指定泛型类型必须实现一个 trait，而无需命名该实现。这被称为 _匿名泛型实现参数 (anonymous generic implementation parameter)_。

例如，`+PartialOrd<T>` 等价于 `impl TPartialOrd: PartialOrd<T>`。

我们可以如下重写 `smallest_element` 函数签名：

```cairo
<!-- Warning: Anchor '1' not found in lib.cairo -->
```

## 结构体

我们还可以定义结构体以对一个或多个字段使用泛型类型参数，使用 `<>` 语法，类似于函数定义。首先，我们在结构体名称后面的尖括号内声明类型参数的名称。然后我们在结构体定义中使用泛型类型，否则我们会指定具体数据类型。下一个代码示例展示了定义 `Wallet<T>`，它有一个类型为 `T` 的 `balance` 字段。

```cairo
#[derive(Drop)]
struct Wallet<T> {
    balance: T,
}

#[executable]
fn main() {
    let w = Wallet { balance: 3 };
}
```

上述代码自动为 `Wallet` 类型派生 `Drop` trait。它等效于编写以下代码：

```cairo
struct Wallet<T> {
    balance: T,
}

impl WalletDrop<T, +Drop<T>> of Drop<Wallet<T>>;

#[executable]
fn main() {
    let w = Wallet { balance: 3 };
}
```

我们避免对 `Wallet` 的 `Drop` 实现使用 `derive` 宏，而是定义我们自己的 `WalletDrop` 实现。注意，就像函数一样，我们必须为 `WalletDrop` 定义一个额外的泛型类型，说明 `T` 也实现了 `Drop` trait。我们基本上是在说，只要 `T` 也是可删除的，结构体 `Wallet<T>` 就是可删除的。

最后，如果我们想向 `Wallet` 添加一个表示其地址的字段，并且我们希望该字段与 `T` 不同但也通用的，我们可以简单地在 `<>` 之间添加另一个泛型类型：

```cairo
#[derive(Drop)]
struct Wallet<T, U> {
    balance: T,
    address: U,
}

#[executable]
fn main() {
    let w = Wallet { balance: 3, address: 14 };
}
```

我们向 `Wallet` 结构体定义添加了一个新的泛型类型 `U`，然后将此类型分配给新的字段成员 `address`。注意 `Drop` trait 的 `derive` 属性对 `U`也有效。

## 枚举

正如我们对结构体所做的那样，我们可以定义枚举以在其变体中持有泛型数据类型。例如 Cairo 核心库提供的 `Option<T>` 枚举：

```cairo,noplayground
enum Option<T> {
    Some: T,
    None,
}
```

`Option<T>` 枚举在类型 `T` 上是泛型的，并且有两个变体：`Some`，它持有一个类型为 `T` 的值，以及 `None`，它不持有任何值。通过使用 `Option<T>` 枚举，我们可以表达可选值的抽象概念，并且因为该值具有泛型类型 `T`，我们可以将此抽象与任何类型一起使用。

枚举也可以使用多个泛型类型，就像核心库提供的 `Result<T, E>` 枚举的定义一样：

```cairo,noplayground
enum Result<T, E> {
    Ok: T,
    Err: E,
}
```

`Result<T, E>` 枚举有两个泛型类型 `T` 和 `E`，以及两个变体：`Ok` 持有类型 `T` 的值，`Err` 持有类型 `E` 的值。这个定义使得在任何我们有一个可能成功（通过返回类型 `T` 的值）或失败（通过返回类型 `E` 的值）的操作的地方使用 `Result` 枚举变得方便。

## 泛型方法

我们可以在结构体和枚举上实现方法，并在其定义中也使用泛型类型。使用我们之前的 `Wallet<T>` 结构体定义，我们为它定义一个 `balance` 方法：

```cairo
#[derive(Copy, Drop)]
struct Wallet<T> {
    balance: T,
}

trait WalletTrait<T> {
    fn balance(self: @Wallet<T>) -> T;
}

impl WalletImpl<T, +Copy<T>> of WalletTrait<T> {
    fn balance(self: @Wallet<T>) -> T {
        return *self.balance;
    }
}

#[executable]
fn main() {
    let w = Wallet { balance: 50 };
    assert!(w.balance() == 50);
}
```

我们首先定义 `WalletTrait<T>` trait，使用泛型类型 `T` 定义一个返回 `Wallet` 中 `balance` 字段值的方法。然后在 `WalletImpl<T>` 中给出该 trait 的实现。注意，你需要在 trait 和实现的定义中都包含泛型类型。

在类型上定义方法时，我们还可以指定对泛型类型的约束。例如，我们可以只为 `Wallet<u128>` 实例实现方法，而不是为 `Wallet<T>` 实现。在代码示例中，我们为 `balance` 字段具有具体类型 `u128` 的钱包定义了一个实现。

```cairo
#[derive(Copy, Drop)]
struct Wallet<T> {
    balance: T,
}

/// Generic trait for wallets
trait WalletTrait<T> {
    fn balance(self: @Wallet<T>) -> T;
}

impl WalletImpl<T, +Copy<T>> of WalletTrait<T> {
    fn balance(self: @Wallet<T>) -> T {
        return *self.balance;
    }
}

/// Trait for wallets of type u128
trait WalletReceiveTrait {
    fn receive(ref self: Wallet<u128>, value: u128);
}

impl WalletReceiveImpl of WalletReceiveTrait {
    fn receive(ref self: Wallet<u128>, value: u128) {
        self.balance += value;
    }
}

#[executable]
fn main() {
    let mut w = Wallet { balance: 50 };
    assert!(w.balance() == 50);

    w.receive(100);
    assert!(w.balance() == 150);
}
```

新方法 `receive` 增加了任何 `Wallet<u128>` 实例的 `balance` 大小。注意我们更改了 `main` 函数，使 `w` 成为可变变量，以便它能够更新其余额。如果我们通过更改 `balance` 的类型来更改 `w` 的初始化，之前的代码将无法编译。

Cairo 也允许我们在泛型 trait 中定义泛型方法。使用过去 `Wallet<U, V>` 的实现，我们将定义一个 trait，它选择两个不同泛型类型的钱包并创建一个具有每种泛型类型的新钱包。首先，让我们重写结构体定义：

```cairo,noplayground
struct Wallet<T, U> {
    balance: T,
    address: U,
}
```

接下来，我们将天真地定义 mixup trait 和实现：

```cairo,noplayground
// This does not compile!
trait WalletMixTrait<T1, U1> {
    fn mixup<T2, U2>(self: Wallet<T1, U1>, other: Wallet<T2, U2>) -> Wallet<T1, U2>;
}

impl WalletMixImpl<T1, U1> of WalletMixTrait<T1, U1> {
    fn mixup<T2, U2>(self: Wallet<T1, U1>, other: Wallet<T2, U2>) -> Wallet<T1, U2> {
        Wallet { balance: self.balance, address: other.address }
    }
}

```

我们创建了一个带有 `mixup<T2, U2>` 方法的 trait `WalletMixTrait<T1, U1>`，给定 `Wallet<T1, U1>` 和 `Wallet<T2, U2>` 的实例，它创建一个新的 `Wallet<T1, U2>`。正如 `mixup` 签名所指定的，`self` 和 `other` 都在函数结束时被删除，这就是为什么这段代码无法编译的原因。如果你从一开始就跟随到现在，你会知道我们必须为所有泛型类型添加一个要求，指定它们将实现 `Drop` trait，以便编译器知道如何删除 `Wallet<T, U>` 的实例。更新后的实现如下：

```cairo
trait WalletMixTrait<T1, U1> {
    fn mixup<T2, +Drop<T2>, U2, +Drop<U2>>(
        self: Wallet<T1, U1>, other: Wallet<T2, U2>,
    ) -> Wallet<T1, U2>;
}

impl WalletMixImpl<T1, +Drop<T1>, U1, +Drop<U1>> of WalletMixTrait<T1, U1> {
    fn mixup<T2, +Drop<T2>, U2, +Drop<U2>>(
        self: Wallet<T1, U1>, other: Wallet<T2, U2>,
    ) -> Wallet<T1, U2> {
        Wallet { balance: self.balance, address: other.address }
    }
}
```

我们在 `WalletMixImpl` 声明中为 `T1` 和 `U1` 添加了可删除的要求。然后我们为 `T2` 和 `U2` 做同样的事情，这次是作为 `mixup` 签名的一部分。我们现在可以尝试 `mixup` 函数：

```cairo,noplayground
#[executable]
fn main() {
    let w1: Wallet<bool, u128> = Wallet { balance: true, address: 10 };
    let w2: Wallet<felt252, u8> = Wallet { balance: 32, address: 100 };

    let w3 = w1.mixup(w2);

    assert!(w3.balance);
    assert!(w3.address == 100);
}
```

我们首先创建两个实例：一个 `Wallet<bool, u128>` 和另一个 `Wallet<felt252, u8>`。然后，我们调用 `mixup` 并创建一个新的 `Wallet<bool, u8>` 实例。
