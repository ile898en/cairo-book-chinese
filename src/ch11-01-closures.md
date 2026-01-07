# 闭包

闭包是可以保存在变量中或作为参数传递给其他函数的匿名函数。你可以在一个地方创建闭包，然后在并在其他地方调用闭包以在不同的上下文中对其进行评估。与函数不同，闭包可以从定义它们的作用域中捕获值。我们将演示这些闭包功能如何允许代码重用和行为自定义。

> 注意：闭包是在 Cairo 2.9 中引入的，目前仍在开发中。
> 一些新功能将在未来的 Cairo 版本中引入，因此本页面将随之发展。

## 理解闭包

在编写 Cairo 程序时，你经常需要将行为作为参数传递给另一个函数。闭包提供了一种内联定义此行为的方法，而无需创建单独的命名函数。在处理集合、错误处理以及任何你想要使用函数作为参数自定义函数行为的场景时，它们特别有价值。

考虑一个简单的例子，我们要根据某些条件以不同方式处理数字。我们无需编写多个函数，而是可以使用闭包在需要的地方定义行为：

```cairo
    let double = |value| value * 2;
    println!("Double of 2 is {}", double(2_u8));
    println!("Double of 4 is {}", double(4_u8));

    // This won't work because `value` type has been inferred as `u8`.
    //println!("Double of 6 is {}", double(6_u16));

    let sum = |x: u32, y: u32, z: u16| {
        x + y + z.into()
    };
    println!("Result: {}", sum(1, 2, 3));
```

闭包的参数在管道符号（`|`）之间。注意我们不需要指定参数和返回值的类型（参见 `double` 闭包），它们将从闭包的使用中推断出来，就像对任何变量所做的那样。
当然，如果你使用具有不同类型的闭包，你将得到一个 `Type annotations needed` 错误，告诉你必须选择并指定闭包参数类型。

主体是一个表达式，可以像 `double` 一样在一行上没有 `{}`，或者像 `sum` 一样在几行上有 `{}`。

## 使用闭包捕获环境

闭包的有趣之处之一是它们可以包含来自其封闭作用域的绑定。

在以下示例中，`my_closure` 使用对 `x` 的绑定来计算 `x + value * 3`。

```cairo
    let x = 8;
    let my_closure = |value| {
        x * (value + 3)
    };

    println!("my_closure(1) = {}", my_closure(1));
```

> 注意，目前闭包仍然不允许捕获可变变量，但这将在未来的 Cairo 版本中得到支持。

## 闭包类型推断和注释

函数和闭包之间还有更多区别。闭包通常不需要像 `fn` 函数那样注释参数或返回值的类型。函数需要类型注释，因为类型是暴露给用户的显式接口的一部分。严格定义此接口对于确保每个人都就函数使用和返回的值类型达成一致至关重要。另一方面，闭包不用于这样的暴露接口：它们存储在变量中，并在不命名它们且不将它们暴露给我们的库用户的情况下使用。

闭包通常很短，并且仅在狭窄的上下文中相关，而不是在任何任意场景中。在这些有限的上下文中，编译器可以推断参数的类型和返回类型，类似于它能够推断大多数量的类型的方式（在极少数情况下，编译器也需要闭包类型注释）。

与变量一样，如果我们想要增加明确性和清晰度，我们可以添加类型注释，代价是比严格必要的更冗长。闭包的类型注释看起来像清单 {{#ref closure-type}} 中显示的定义。在这个例子中，我们定义了一个闭包并将其存储在变量中，而不是像我们在清单 13-1 中所做的那样在作为参数传递它的地方定义闭包。

```cairo
    let expensive_closure = |num: u32| -> u32 {
        num
    };
```

{{#label closure-type}} 清单 {{#ref closure-type}}: 在闭包中添加参数和返回值类型的可选类型注释

添加了类型注释后，闭包的语法看起来更类似于函数的语法。这里我们为了比较，定义了一个将其参数加 1 的函数和一个具有相同行为的闭包。我们要添加了一些空格来对齐相关部分。这说明了闭包语法除了使用管道和可选语法的数量外，与函数语法是多么相似：

```cairo, ignore
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

第一行显示了一个函数定义，第二行显示了一个完全注释的闭包定义。在第三行中，我们从闭包定义中删除了类型注释。在第四行中，我们删除了方括号，这是可选的，因为闭包主体只有一个表达式。这些都是有效的定义，在调用时会产生相同的行为。`add_one_v3` 和 `add_one_v4` 行需要评估闭包才能编译，因为类型将从其使用中推断出来。这类似于 `let array = array![];` 需要类型注释或某种类型的值插入到 `array` 中，以便 Cairo 能够推断类型。

对于闭包定义，编译器将为它的每个参数及其返回值推断一个具体类型。例如，清单 {{#ref closure-different-types}} 显示了一个简短闭包的定义，它只是返回它作为参数接收的值。除了用于此示例的目的外，此闭包不是很有用。注意我们在定义中没有添加任何类型注释。因为没有类型注释，我们可以用任何类型调用闭包，我们在这里第一次用 `u64` 做了这件事。如果我们随后尝试用 `32` 调用 `example_closure`，我们将得到一个错误。

```cairo, noplayground
    let example_closure = |x| x;

    let s = example_closure(5_u64);
    let n = example_closure(5_u32);
```

{{#label closure-different-types}} 清单 {{#ref closure-different-types}}:
尝试调用一个类型被推断为两种不同类型的闭包

编译器给出了这个错误：

```
$ scarb build 
   Compiling listing_closure_different_types v0.1.0 (listings/ch11-functional-features/listing_closure_different_types/Scarb.toml)
warn[E0001]: Unused variable. Consider ignoring by prefixing with `_`.
 --> listings/ch11-functional-features/listing_closure_different_types/src/lib.cairo:7:9
    let s = example_closure(5_u64);
        ^

warn[E0001]: Unused variable. Consider ignoring by prefixing with `_`.
 --> listings/ch11-functional-features/listing_closure_different_types/src/lib.cairo:8:9
    let n = example_closure(5_u32);
        ^

error: Trait has no implementation in context: core::ops::function::Fn::<{closure@listings/ch11-functional-features/listing_closure_different_types/src/lib.cairo:5:27: 5:30}, (core::integer::u32,)>.
 --> listings/ch11-functional-features/listing_closure_different_types/src/lib.cairo:8:13
    let n = example_closure(5_u32);
            ^^^^^^^^^^^^^^^^^^^^^^

error: could not compile `listing_closure_different_types` due to previous error

```

我们第一次使用 `u64` 值调用 `example_closure` 时，编译器推断 `x` 的类型和闭包的返回类型为 `u64`。然后这些类型被锁定在 `example_closure` 的闭包中，当我们下次尝试使用具有相同闭包的不同类型时，我们会得到类型错误。

## 将捕获的值移出闭包和 `Fn` Traits

一旦闭包从定义闭包的环境中捕获了引或捕获了值的所有权（从而影响了什么被 _移动到_ 闭包中，如果有的话），闭包主体中的代码定义了稍后评估闭包时对引用或值发生的情况（从而影响了什么被 _移出_ 闭包，如果有的话）。闭包主体可以做以下任何事情：将捕获的值移出闭包，既不移动也不改变值，或者从一开始就不从环境中捕获任何东西。

闭包捕获和处理环境中值的方式会影响闭包实现哪些 traits，而 traits 是函数和结构体指定它们可以使用哪种闭包的方式。闭包将根据闭包的主体如何处理值，以累加的方式自动实现这些 `Fn` traits 中的一个、两个或全部三个：

1. `FnOnce` 适用于可以调用一次的闭包。所有闭包都至少实现了这个 trait，因为所有闭包都可以被调用。将捕获的值移出其主体的闭包将仅实现 `FnOnce` 而不实现其他 `Fn` traits，因为它只能被调用一次。

2. `Fn` 适用于不将捕获的值移出其主体且不改变捕获的值的闭包，以及不从其环境中捕获任何内容的闭包。这些闭包可以被多次调用而不改变其环境，这在诸如并发多次调用闭包的情况下很重要。

让我们看看我们在清单 13-1 中使用的 `OptionTrait<T>` 上的 `unwrap_or_else` 方法的定义：

```cairo, ignore
pub impl OptionTraitImpl<T> of OptionTrait<T> {
    #[inline]
    fn unwrap_or_else<F, +Drop<F>, impl func: core::ops::FnOnce<F, ()>[Output: T], +Drop<func::Output>>(
        self: Option<T>, f: F,
    ) -> T {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```

回想一下，`T` 是表示 `Option` 的 `Some` 变体中值类型的泛型类型。该类型 `T` 也是 `unwrap_or_else` 函数的返回类型：例如，在 `Option<ByteArray>` 上调用 `unwrap_or_else` 的代码将获得 `ByteArray`。

接下来，注意 `unwrap_or_else` 函数具有额外的泛型类型参数 `F`。`F` 类型是名为 `f` 的参数的类型，这是我们在调用 `unwrap_or_else` 时提供的闭包。

在泛型类型 `F` 上指定的 trait 约束是 `impl func: core::ops::FnOnce<F, ()>[Output: T]`，这意味着 `F` 必须能够被调用一次，不接受参数（使用单元类型 `()`），并返回 `T` 作为输出。在 trait 约束中使用 `FnOnce` 表达了 `unwrap_or_else` 最多只调用 `f` 一次的约束。在 `unwrap_or_else` 的主体中，我们可以看到如果 `Option` 是 `Some`，则不会调用 `f`。如果 `Option` 是 `None`，则 `f` 将被调用一次。因为所有闭包都实现了 `FnOnce`，`unwrap_or_else` 接受所有三种类型的闭包，并且尽可能灵活。

`Fn` traits 在定义或使用利用闭包的函数或类型时很重要。在下一节中，我们将讨论迭代器。许多迭代器方法接受闭包参数，所以在我们继续时请记住这些闭包细节！

[unwrap-or-else]:
  https://docs.swmansion.com/scarb/corelib/core-option-OptionTrait.html#unwrap_or_else

在底层，闭包是通过 `FnOnce` 和 `Fn` traits 实现的。
`FnOnce` 为可能消耗捕获变量的闭包实现，其中 `Fn` 为仅捕获可复制变量的闭包实现。

## 使用闭包实现你的函数式编程模式

闭包的另一个巨大兴趣是，像任何类型的变量一样，你可以将它们作为函数参数传递。这种机制在函数式编程中被大量使用，通过像 `map`、`filter` 或 `reduce` 这样的经典函数。

这是 `map` 的一个潜在实现，用于将相同的函数应用于数组的所有项目：

```cairo, noplayground
#[generate_trait]
impl ArrayExt of ArrayExtTrait {
    // Needed in Cairo 2.11.4 because of a bug in inlining analysis.
    #[inline(never)]
    fn map<T, +Drop<T>, F, +Drop<F>, impl func: core::ops::Fn<F, (T,)>, +Drop<func::Output>>(
        self: Array<T>, f: F,
    ) -> Array<func::Output> {
        let mut output: Array<func::Output> = array![];
        for elem in self {
            output.append(f(elem));
        }
        output
    }
}
```

> 注意，由于内联分析中的错误，应使用 `#[inline(never)]` 禁用此分析过程。

在此实现中，你会注意到，虽然 `T` 是输入数组 `self` 的元素类型，但输出数组的元素类型由 `f` 闭包的输出类型（来自 `Fn` trait 的关联类型 `func::Output`）定义。

这意味着你的 `f` 闭包可以返回与以下代码中的 `_double` 相同类型的元素，或与 `_another` 相同的任何其他类型的元素：

```cairo
    let double = array![1, 2, 3].map(|item: u32| item * 2);
    let another = array![1, 2, 3].map(|item: u32| {
        let x: u64 = item.into();
        x * x
    });

    println!("double: {:?}", double);
    println!("another: {:?}", another);
```

> 目前，Cairo 2.9 提供了一项实验性功能，允许你使用 `Scarb.toml` 中的 `experimental-features = ["associated_item_constraints"]` 指定 trait 的关联类型。

假设我们要为数组实现 `filter` 函数，以过滤掉不符合标准的元素。此标准将通过一个闭包提供，该闭包接受一个元素作为输入，如果必须保留该元素则返回 `true`，否则返回 `false`。这意味着，我们需要指定闭包必须返回一个 `boolean`。

```cairo, noplayground
#[generate_trait]
impl ArrayFilterExt of ArrayFilterExtTrait {
    // Needed in Cairo 2.11.4 because of a bug in inlining analysis.
    #[inline(never)]
    fn filter<
        T,
        +Copy<T>,
        +Drop<T>,
        F,
        +Drop<F>,
        impl func: core::ops::Fn<F, (T,)>[Output: bool],
        +Drop<func::Output>,
    >(
        self: Array<T>, f: F,
    ) -> Array<T> {
        let mut output: Array<T> = array![];
        for elem in self {
            if f(elem) {
                output.append(elem);
            }
        }
        output
    }
}
```

```cairo
    let even = array![3, 4, 5, 6].filter(|item: u32| item % 2 == 0);
    println!("even: {:?}", even);
```
