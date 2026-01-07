# Cairo 中的 Traits

Trait 定义了某种类型可以实现的一组方法。当此 trait 被实现时，可以在该类型的实例上调用这些方法。Trait 与泛型类型结合，定义了某种特定类型具有并可以与其他类型共享的功能。我们可以使用 traits 以抽象的方式定义共享行为。我们可以使用 _trait 约束 (trait bounds)_ 来指定泛型类型可以是任何具有特定行为的类型。

> 注意：Traits 类似于其他语言中常称为接口 (interfaces) 的功能，尽管有一些差异。

虽然编写 traits 可以不接受泛型类型，但当与泛型类型一起使用时，它们最有用。我们在 [上一章][generics] 已经涵盖了泛型，我们将在本章中使用它们来演示如何使用 traits 来定义泛型类型的共享行为。

[generics]: ./ch08-01-generic-data-types.md

## 定义 Trait

一个类型的行为由我们可以对该类型调用的方法组成。如果我们可以在所有那些类型上调用相同的方法，则不同的类型共享相同的行为。Trait 定义是一种将方法签名组合在一起的方法，以定义实现某些目的所需的一组行为。

例如，通过在特定位置持有新闻故事，比方说我们有一个结构体 `NewsArticle`。我们可以定义一个 trait `Summary`，它描述了可以总结 `NewsArticle` 类型的东西的行为。

```cairo,noplayground
pub trait Summary {
    fn summarize(self: @NewsArticle) -> ByteArray;
}
```

{{#label first_trait_signature}} <span class="caption"> 清单
{{#ref first_trait_signature}}: 一个 `Summary` trait，由 `summarize` 方法提供的行为组成</span>

在清单 {{#ref first_trait_signature}} 中，我们使用 `trait` 关键字声明一个 trait，然后是 trait 的名称，在本例中为 `Summary`。我们还将 trait 声明为 `pub`，以便依赖此 crate 的 crate 也可以使用此 trait，我们将在几个示例中看到。

在花括号内，我们声明描述实现此 trait 的类型的行为的方法签名，在这个例子中是 `fn summarize(self: @NewsArticle) -> ByteArray;`。在方法签名之后，我们不提供花括号内的实现，而是使用分号。

> 注意：`ByteArray` 类型是 Cairo 中用于表示字符串的类型。

由于 trait 不是泛型的，`self` 参数也不是泛型的，并且是 `@NewsArticle` 类型。这意味着 `summarize` 方法只能在 `NewsArticle` 的实例上调用。

现在，考虑我们想制作一个名为 _aggregator_ 的媒体聚合器库 crate，它可以显示可能存储在 `NewsArticle` 或 `Tweet` 实例中的数据的摘要。为此，我们需要每种类型的摘要，我们将通过在该类型的实例上调用 summarize 方法来请求该摘要。通过在泛型类型 `T` 上定义 `Summary` trait，我们可以在任何我们想要能够总结的类型上实现 `summarize` 方法。

```cairo,noplayground
    pub trait Summary<T> {
        fn summarize(self: @T) -> ByteArray;
    }
```

{{#label trait_signature}} <span class="caption"> 清单
{{#ref trait_signature}}: 一个 `Summary` trait，由泛型类型的 `summarize` 方法提供的行为组成</span>

每个实现此 trait 的类型都必须为方法体提供自己的自定义行为。编译器将强制要求任何实现 `Summary` trait 的类型都由完全具有此签名的方法 `summarize` 定义。

一个 trait 可以在其主体中有多个方法：方法签名每行一个，每行以分号结尾。

## 在类型上实现 Trait

现在我们已经定义了 `Summary` trait 方法所需的签名，我们可以在我们的媒体聚合器中的类型上实现它。以下代码展示了 `Summary` trait 在 `NewsArticle` 结构体上的实现，它使用标题、作者和位置来创建 `summarize` 的返回值。对于 `Tweet` 结构体，我们将 `summarize` 定义为用户名后跟推文的全部文本，假设推文内容已被限制为 280 个字符。

```cairo,noplayground
    #[derive(Drop)]
    pub struct NewsArticle {
        pub headline: ByteArray,
        pub location: ByteArray,
        pub author: ByteArray,
        pub content: ByteArray,
    }

    impl NewsArticleSummary of Summary<NewsArticle> {
        fn summarize(self: @NewsArticle) -> ByteArray {
            format!("{} by {} ({})", self.headline, self.author, self.location)
        }
    }

    #[derive(Drop)]
    pub struct Tweet {
        pub username: ByteArray,
        pub content: ByteArray,
        pub reply: bool,
        pub retweet: bool,
    }

    impl TweetSummary of Summary<Tweet> {
        fn summarize(self: @Tweet) -> ByteArray {
            format!("{}: {}", self.username, self.content)
        }
    }
```

{{#label trait_impl}} <span class="caption"> 清单 {{#ref trait_impl}}:
`Summary` trait 在 `NewsArticle` 和 `Tweet` 上的实现</span>

在类型上实现 trait 类似于实现常规方法。区别在于，在 `impl` 之后，我们为实现放一个名称，然后使用 `of` 关键字，然后指定我们要为其编写实现的 trait 的名称。如果实现是针对泛型类型的，我们将泛型类型名称放在 trait 名称后的尖括号中。

注意，为了使 trait 方法可访问，必须有从调用方法的范围可见的该 trait 的实现。如果 trait 是 `pub` 而实现不是，并且实现从调用 trait 方法的范围不可见，这将导致编译错误。

在 `impl` 块中，我们放置 trait 定义已定义的方法签名。我们不使用分号结束每个签名，而是使用花括号，并在方法体中填写我们希望 trait 方法对特定类型具有的特定行为。

现在库已经在 `NewsArticle` 和 `Tweet` 上实现了 `Summary` trait，crate 的用户可以像调用常规方法一样在 `NewsArticle` 和 `Tweet` 的实例上调用 trait 方法。唯一的区别是用户必须将 trait 以及类型引入作用域。这是一个 crate 如何使用我们的 `aggregator` crate 的示例：

```cairo
use aggregator::{NewsArticle, Summary, Tweet};

#[executable]
fn main() {
    let news = NewsArticle {
        headline: "Cairo has become the most popular language for developers",
        location: "Worldwide",
        author: "Cairo Digger",
        content: "Cairo is a new programming language for zero-knowledge proofs",
    };

    let tweet = Tweet {
        username: "EliBenSasson",
        content: "Crypto is full of short-term maximizing projects. \n @Starknet and @StarkWareLtd are about long-term vision maximization.",
        reply: false,
        retweet: false,
    }; // Tweet instantiation

    println!("New article available! {}", news.summarize());
    println!("New tweet! {}", tweet.summarize());
}
```

{{#label trait_main}}

这段代码打印以下内容：

```shell
$ scarb execute 
   Compiling no_listing_15_traits v0.1.0 (listings/ch08-generic-types-and-traits/no_listing_15_traits/Scarb.toml)
    Finished `dev` profile target(s) in 1 second
   Executing no_listing_15_traits
New article available! Cairo has become the most popular language for developers by Cairo Digger (Worldwide)
New tweet! EliBenSasson: Crypto is full of short-term maximizing projects. 
 @Starknet and @StarkWareLtd are about long-term vision maximization.


```

依赖 _aggregator_ crate 的其他 crate 也可以将 `Summary` trait 引入作用域，以在它们自己的类型上实现 `Summary`。

## 默认实现

有时，为 trait 中的某些或所有方法提供默认行为，而不是要求在每种类型上实现所有方法，这很有用。然后，当我们在特定类型上实现 trait 时，我们可以保留或覆盖每个方法的默认行为。

在清单 {{#ref default_impl}} 中，我们要为 `Summary` trait 的 `summarize` 方法指定一个默认字符串，而不是仅定义方法签名，就像我们在清单 {{#ref trait_signature}} 中所做的那样。

<span class="caption">文件名: src/lib.cairo</span>

```cairo
    pub trait Summary<T> {
        fn summarize(self: @T) -> ByteArray {
            "(Read more...)"
        }
    }
```

{{#label default_impl}} <span class="caption"> 清单 {{#ref default_impl}}:
定义一个带有 `summarize` 方法默认实现的 `Summary` trait</span>

要使用默认实现总结 `NewsArticle` 的实例，我们指定一个空的 `impl` 块 `impl NewsArticleSummary of Summary<NewsArticle> {}`。

即使我们不再直接在 `NewsArticle` 上定义 `summarize` 方法，我们已经提供了一个默认实现，并指定 `NewsArticle` 实现了 `Summary` trait。因此，我们仍然可以在 `NewsArticle` 的实例上调用 `summarize` 方法，如下所示：

```cairo
use aggregator::{NewsArticle, Summary};

#[executable]
fn main() {
    let news = NewsArticle {
        headline: "Cairo has become the most popular language for developers",
        location: "Worldwide",
        author: "Cairo Digger",
        content: "Cairo is a new programming language for zero-knowledge proofs",
    };

    println!("New article available! {}", news.summarize());
}
```

这段代码打印 `New article available! (Read more...)`。

创建默认实现不需要我们更改关于 `Tweet` 上 `Summary` 的先前实现的任何内容。原因是覆盖默认实现的语法与实现没有默认实现的 trait 方法的语法相同。

默认实现可以调用同一 trait 中的其他方法，即使那些其他方法没有默认实现。通过这种方式，trait 可以提供许多有用的功能，并且只需要实现者指定其中的一小部分。例如，我们可以定义 `Summary` trait 有一个 `summarize_author` 方法，其实现是必需的，然后定义一个 `summarize` 方法，该方法具有调用 `summarize_author` 方法的默认实现：

```cairo
    pub trait Summary<T> {
        fn summarize(
            self: @T,
        ) -> ByteArray {
            format!("(Read more from {}...)", Self::summarize_author(self))
        }
        fn summarize_author(self: @T) -> ByteArray;
    }
```

要使用这个版本的 `Summary`，我们在类型上实现该 trait 时只需要定义 `summarize_author`：

```cairo
    impl TweetSummary of Summary<Tweet> {
        fn summarize_author(self: @Tweet) -> ByteArray {
            format!("@{}", self.username)
        }
    }
```

在定义了 `summarize_author` 后，我们可以在 `Tweet` 结构体的实例上调用 `summarize`，`summarize` 的默认实现将调用我们提供的 `summarize_author` 定义。因为我们已经实现了 `summarize_author`，`Summary` trait 给了我们 `summarize` 方法的行为，而不需要我们编写更多代码。

```cairo
use aggregator::{Summary, Tweet};

#[executable]
fn main() {
    let tweet = Tweet {
        username: "EliBenSasson",
        content: "Crypto is full of short-term maximizing projects. \n @Starknet and @StarkWareLtd are about long-term vision maximization.",
        reply: false,
        retweet: false,
    };

    println!("1 new tweet: {}", tweet.summarize());
}
```

这段代码打印 `1 new tweet: (Read more from @EliBenSasson...)`。

注意，无法从同一方法的覆盖实现中调用默认实现。

<!-- TODO: NOT AVAILABLE IN CAIRO FOR NOW move traits as parameters here -->
<!-- ## Traits as parameters
 
 Now that you know how to define and implement traits, we can explore how to use
 traits to define functions that accept many different types. We'll use the
 `Summary` trait we implemented on the `NewsArticle` and `Tweet` types to define a `notify` function that calls the `summarize` method
 on its `item` parameter, which is of some type that implements the `Summary` trait. To do this, we use the `impl Trait` syntax.
 
 Instead of a concrete type for the `item` parameter, we specify the `impl`
 keyword and the trait name. This parameter accepts any type that implements the
 specified trait. In the body of `notify`, we can call any methods on `item`
 that come from the `Summary` trait, such as `summarize`. We can call `notify`
 and pass in any instance of `NewsArticle` or `Tweet`. Code that calls the
 function with any other type, such as a `String` or an `i32`, won’t compile
 because those types don’t implement `Summary`. -->

<!-- TODO NOT AVAILABLE IN CAIRO FOR NOW Using trait bounds to conditionally implement methods -->

## 管理和使用外部 Trait

要使用 trait 方法，你需要确保障导入了正确的 traits/实现。在某些情况下，如果 trait 和实现是在单独的模块中声明的，你可能不仅需要导入 trait，还需要导入实现。如果 `CircleGeometry` 实现位于名为 _circle_ 的单独模块/文件中，那么为了在 `Circle` 结构体上定义 `boundary` 方法，我们需要在 _circle_ 模块中导入 `ShapeGeometry` trait。

如果代码像清单 {{#ref external_trait}} 那样组织成模块，其中 trait 的实现定义在与 trait 本身不同的模块中，则需要显式导入相关的 trait 或实现。

```cairo,noplayground
// Here T is an alias type which will be provided during implementation
pub trait ShapeGeometry<T> {
    fn boundary(self: T) -> u64;
    fn area(self: T) -> u64;
}

mod rectangle {
    // Importing ShapeGeometry is required to implement this trait for Rectangle
    use super::ShapeGeometry;

    #[derive(Copy, Drop)]
    pub struct Rectangle {
        pub height: u64,
        pub width: u64,
    }

    // Implementation RectangleGeometry passes in <Rectangle>
    // to implement the trait for that type
    impl RectangleGeometry of ShapeGeometry<Rectangle> {
        fn boundary(self: Rectangle) -> u64 {
            2 * (self.height + self.width)
        }
        fn area(self: Rectangle) -> u64 {
            self.height * self.width
        }
    }
}

mod circle {
    // Importing ShapeGeometry is required to implement this trait for Circle
    use super::ShapeGeometry;

    #[derive(Copy, Drop)]
    pub struct Circle {
        pub radius: u64,
    }

    // Implementation CircleGeometry passes in <Circle>
    // to implement the imported trait for that type
    impl CircleGeometry of ShapeGeometry<Circle> {
        fn boundary(self: Circle) -> u64 {
            (2 * 314 * self.radius) / 100
        }
        fn area(self: Circle) -> u64 {
            (314 * self.radius * self.radius) / 100
        }
    }
}
use circle::Circle;
use rectangle::Rectangle;

#[executable]
fn main() {
    let rect = Rectangle { height: 5, width: 7 };
    println!("Rectangle area: {}", ShapeGeometry::area(rect)); //35
    println!("Rectangle boundary: {}", ShapeGeometry::boundary(rect)); //24

    let circ = Circle { radius: 5 };
    println!("Circle area: {}", ShapeGeometry::area(circ)); //78
    println!("Circle boundary: {}", ShapeGeometry::boundary(circ)); //31
}
```

{{#label external_trait}} <span class="caption"> 清单
{{#ref external_trait}}: 实现外部 trait</span>

注意，在清单 {{#ref external_trait}} 中，`CircleGeometry` 和 `RectangleGeometry` 实现不需要声明为 `pub`。事实上，公有的 `ShapeGeometry` trait 用于在 `main` 函数中打印结果。无论实现的可见性如何，编译器都会为 `ShapeGeometry` 公有 trait 找到合适的实现。

## Impl 别名

实现在导入时可以使用别名。当你想用具体类型实例化泛型实现时，这最有用。例如，假设我们定义了一个用于为类型 `T` 返回值 `2` 的 trait `Two`。我们可以通过简单地加上两倍的 `one` 值并返回它，为所有实现 `One` trait 的类型编写 `Two` 的微不足道的泛型实现。然而，在我们的公共 API 中，我们可能只想暴露 `u8` 和 `u128` 类型的 `Two` 实现。

```cairo,noplayground
trait Two<T> {
    fn two() -> T;
}

mod one_based {
    pub impl TwoImpl<
        T, +Copy<T>, +Drop<T>, +Add<T>, impl One: core::num::traits::One<T>,
    > of super::Two<T> {
        fn two() -> T {
            One::one() + One::one()
        }
    }
}

pub impl U8Two = one_based::TwoImpl<u8>;
pub impl U128Two = one_based::TwoImpl<u128>;
```

{{#label impl-aliases}} <span class="caption"> 清单 {{#ref impl-aliases}}:
使用 impl 别名用具体类型实例化泛型 impl</span>

我们可以在私有模块中定义泛型实现，使用 impl 别名为这两个具体类型实例化泛型实现，并使这两个实现公有，同时保持泛型实现私有且不暴露。这样，我们可以避免使用泛型实现的代码重复，同时保持公共 API 的干净和简单。

## 负实现 (Negative Impls)

> 注意：这仍然是一个实验性功能，只有在你的 _Scarb.toml_ 文件的 `[package]` 部分下启用 `experimental-features = ["negative_impls"]` 才能使用。

负实现，也称为负 traits 或负约束，是一种允许你在定义泛型类型的 trait 实现时表达类型 _不_ 实现某个 trait 的机制。负 impl 使你能够编写仅在当前范围内不存在另一个实现时才适用的实现。

例如，假设我们有一个 trait `Producer` 和一个 trait `Consumer`，我们想定义一个泛型行为，其中所有类型默认实现 `Consumer` trait。然而，我们要确保没有类型既可以作为 `Consumer` 又可以作为 `Producer`。我们可以使用负 impl 来表达此限制。

在清单 {{#ref negative-impls}} 中，我们定义了一个实现 `Producer` trait 的 `ProducerType`，以及另外两个不实现 `Producer` trait 的类型 `AnotherType` 和 `AThirdType`。然后我们使用负 impl 为所有不实现 `Producer` trait 的类型创建 `Consumer` trait 的默认实现。

```cairo
#[derive(Drop)]
struct ProducerType {}

#[derive(Drop, Debug)]
struct AnotherType {}

#[derive(Drop, Debug)]
struct AThirdType {}

trait Producer<T> {
    fn produce(self: T) -> u32;
}

trait Consumer<T> {
    fn consume(self: T, input: u32);
}

impl ProducerImpl of Producer<ProducerType> {
    fn produce(self: ProducerType) -> u32 {
        42
    }
}

impl TConsumerImpl<T, +core::fmt::Debug<T>, +Drop<T>, -Producer<T>> of Consumer<T> {
    fn consume(self: T, input: u32) {
        println!("{:?} consumed value: {}", self, input);
    }
}

#[executable]
fn main() {
    let producer = ProducerType {};
    let another_type = AnotherType {};
    let third_type = AThirdType {};
    let production = producer.produce();

    // producer.consume(production); Invalid: ProducerType does not implement Consumer
    another_type.consume(production);
    third_type.consume(production);
}
```

{{#label negative-impls}} <span class="caption"> 清单
{{#ref negative-impls}}: 使用负 impl 强制类型不能同时实现 `Producer` 和 `Consumer` traits</span>

在 `main` 函数中，我们创建 `ProducerType`、`AnotherType` 和 `AThirdType` 的实例。然后我们在 `producer` 实例上调用 `produce` 方法，并将结果传递给 `another_type` 和 `third_type` 实例上的 `consume` 方法。最后，我们尝试在 `producer` 实例上调用 `consume` 方法，这会导致编译时错误，因为 `ProducerType` 没有实现 `Consumer` trait。

## 关联项上的约束 traits

> 目前，关联项被认为是一项实验性功能。为了使用它们，你需要在你的 `Scarb.toml` 的 `[package]` 部分下添加以下内容：
> `experimental-features = ["associated_item_constraints"]`。

在某些情况下，你可能希望根据泛型参数的类约束 trait 的 [关联项][associated items]。你可以在 trait 约束后使用 `[AssociatedItem: ConstrainedValue]` 语法来做到这一点。

[associated items]: ./ch12-10-associated-items.md

假设你想为集合实现一个 `extend` 方法。此方法接受一个迭代器并将其元素添加到集合中。为了确保类型安全，我们要让迭代器的元素与集合的元素类型匹配。我们可以通过约束 `Iterator::Item` 关联类型与集合的类型匹配来实现这一点。

在清单 {{#ref associated-items-constraints}} 中，我们通过定义一个 trait `Extend<T, A>` 并在 `extend` 函数的 trait 约束上使用 `[Item: A]` 作为约束来实现这一点。此外，我们使用 `Destruct` trait 确保迭代器被消耗，并展示了 `Extend<Array<T>, T>` 的示例实现。

```cairo
trait Extend<T, A> {
    fn extend<I, +core::iter::Iterator<I>[Item: A], +Destruct<I>>(ref self: T, iterator: I);
}

impl ArrayExtend<T, +Drop<T>> of Extend<Array<T>, T> {
    fn extend<I, +core::iter::Iterator<I>[Item: T], +Destruct<I>>(ref self: Array<T>, iterator: I) {
        for item in iterator {
            self.append(item);
        }
    }
}
```

{{#label associated-items-constraints}} <span class="caption"> 清单
{{#ref associated-items-constraints}}: 使用关联项约束确保类型与另一种类型的关联类型匹配</span>

## 用于类型相等约束的 `TypeEqual` Trait

来自 `core::metaprogramming` 模块的 `TypeEqual` trait 让你能够基于类型相等创建约束。在大多数情况下，你不需要 `+TypeEqual`，你可以仅使用泛型参数和关联类型约束来实现相同的目的，但 `TypeEqual` 在某些高级场景中可能很有用。

第一个用例是为匹配特定条件的所有类型实现一个 trait，但排除特定类型。我们通过在 `TypeEqual` trait 上使用负实现来做到这一点。

在清单 {{#ref type-equal-negative-constraints}} 中，我们创建一个 `SafeDefault` trait 并为实现 `Default` 的任何类型 `T` 实现它。但是，我们使用 `-TypeEqual<T, SensitiveData>` 排除 `SensitiveData` 类型。

```cairo
trait SafeDefault<T> {
    fn safe_default() -> T;
}

#[derive(Drop, Default)]
struct SensitiveData {
    secret: felt252,
}

// Implement SafeDefault for all types EXCEPT SensitiveData
impl SafeDefaultImpl<
    T, +Default<T>, -core::metaprogramming::TypeEqual<T, SensitiveData>,
> of SafeDefault<T> {
    fn safe_default() -> T {
        Default::default()
    }
}

#[executable]
fn main() {
    let _safe: u8 = SafeDefault::safe_default();
    let _unsafe: SensitiveData = Default::default(); // Allowed
    // This would cause a compile error:
// let _dangerous: SensitiveData = SafeDefault::safe_default();
}
```

{{#label type-equal-negative-constraints}} <span class="caption"> 清单
{{#ref type-equal-negative-constraints}}: 使用 `TypeEqual` trait 从实现中排除特定类型</span>

第二个用例是确保两种类型相等，在处理 [关联类型][associated types] 时特别有用。

[associated types]: ./ch12-10-associated-items.md#associated-types

在清单 {{#ref type-equal-constraints}} 中，我们用一个具有关联类型 `State` 的 `StateMachine` trait 来展示这一点。我们创建两个类型，`TA` 和 `TB`，都使用 `StateCounter` 作为它们的 `State`。然后我们实现一个 `combine` 函数，该函数仅在两个状态机具有相同的状态类型时才工作，使用约束 `TypeEqual<A::State, B::State>`。

```cairo
trait StateMachine {
    type State;
    fn transition(ref state: Self::State);
}

#[derive(Copy, Drop)]
struct StateCounter {
    counter: u8,
}

impl TA of StateMachine {
    type State = StateCounter;
    fn transition(ref state: StateCounter) {
        state.counter += 1;
    }
}

impl TB of StateMachine {
    type State = StateCounter;
    fn transition(ref state: StateCounter) {
        state.counter *= 2;
    }
}

fn combine<
    impl A: StateMachine,
    impl B: StateMachine,
    +core::metaprogramming::TypeEqual<A::State, B::State>,
>(
    ref self: A::State,
) {
    A::transition(ref self);
    B::transition(ref self);
}

#[executable]
fn main() {
    let mut initial = StateCounter { counter: 0 };
    combine::<TA, TB>(ref initial);
}
```

{{#label type-equal-constraints}} <span class="caption"> 清单
{{#ref type-equal-constraints}}: 使用 `TypeEqual` trait 确保两种类型具有匹配的关联类型</span>

{{#quiz ../quizzes/ch08-02-traits.toml}}
