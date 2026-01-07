# 解引用强制转换 (Deref Coercion)

解引用强制转换 (Deref coercion) 简化了我们要与嵌套或包装数据结构交互的方式，允许一种类型的实例像另一种类型的实例一样行动。这种机制通过实现 `Deref` trait 来启用，它允许隐式转换（或强制转换）为不同的类型，提供对底层数据的直接访问。

解引用强制转换通过 `Deref` 和 `DerefMut` traits 实现。当类型 `T` 实现了对类型 `K` 的 `Deref` 或 `DerefMut` 时，`T` 的实例可以直接访问 `K` 的成员。

Cairo 中的 `Deref` trait 定义如下：

```cairo, noplayground
pub trait Deref<T> {
    type Target;
    fn deref(self: T) -> Self::Target;
}
```

`Target` 类型指定了解引用的结果，`deref` 方法定义了如何将 `T` 转换为 `K`。

## 使用解引用强制转换

为了更好地理解解引用强制转换是如何工作的，让我们看一个实际的例子。我们将创建一个围绕类型 `T` 的简单泛型包装器类型 `Wrapper<T>`，并使用它来包装一个 `UserProfile` 结构体。

```cairo, noplayground
#[derive(Drop, Copy)]
struct UserProfile {
    username: felt252,
    email: felt252,
    age: u16,
}

#[derive(Drop, Copy)]
struct Wrapper<T> {
    value: T,
}
```

`Wrapper` 结构体包装了一个 `T` 类型的单个值泛型。为了简化对包装值的访问，我们为 `Wrapper<T>` 实现 `Deref` trait。

```cairo, noplayground
impl DerefWrapper<T> of Deref<Wrapper<T>> {
    type Target = T;
    fn deref(self: Wrapper<T>) -> T {
        self.value
    }
}
```

这个实现非常简单。`deref` 方法返回包装的值，允许 `Wrapper<T>` 的实例直接访问 `T` 的成员。

在实践中，这种机制是完全透明的。以下示例演示了如何持有一个 `Wrapper<UserProfile>` 实例，我们可以打印底层 `UserProfile` 实例的 `username` 和 `age` 字段。

```cairo
#[executable]
fn main() {
    let wrapped_profile = Wrapper {
        value: UserProfile { username: 'john_doe', email: 'john@example.com', age: 30 },
    };
    // Access fields directly via deref coercion
    println!("Username: {}", wrapped_profile.username);
    println!("Current age: {}", wrapped_profile.age);
}
```

### 将解引用强制转换限制为可变变量

虽然 `Deref` 适用于可变和不可变变量，但 `DerefMut` 仅适用于可变变量。与名称可能暗示的相反，`DerefMut` 不提供对底层数据的可变访问。

```cairo, noplayground
impl DerefMutWrapper<T, +Copy<T>> of DerefMut<Wrapper<T>> {
    type Target = T;
    fn deref_mut(ref self: Wrapper<T>) -> T {
        self.value
    }
}
```

如果你尝试在不可变变量上使用 `DerefMut`，编译器会抛出错误。这是一个例子：

```cairo, noplayground
fn error() {
    let wrapped_profile = Wrapper {
        value: UserProfile { username: 'john_doe', email: 'john@example.com', age: 30 },
    };
    // Uncommenting the next line will cause a compilation error
    println!("Username: {}", wrapped_profile.username);
}
```

编译此代码将导致以下错误：

```plaintext
$ scarb build 
   Compiling no_listing_09_deref_coercion_example v0.1.0 (listings/ch12-advanced-features/no_listing_09_deref_mut_example/Scarb.toml)
error: Type "no_listing_09_deref_coercion_example::Wrapper::<no_listing_09_deref_coercion_example::UserProfile>" has no member "username"
 --> listings/ch12-advanced-features/no_listing_09_deref_mut_example/src/lib.cairo:32:46
    println!("Username: {}", wrapped_profile.username);
                                             ^^^^^^^^

error: could not compile `no_listing_09_deref_coercion_example` due to previous error

```

为了使上述代码工作，我们需要将 `wrapped_profile` 定义为可变变量。

```cairo, noplayground
#[executable]
fn main() {
    let mut wrapped_profile = Wrapper {
        value: UserProfile { username: 'john_doe', email: 'john@example.com', age: 30 },
    };

    println!("Username: {}", wrapped_profile.username);
    println!("Current age: {}", wrapped_profile.age);
}
```

## 通过解引用强制转换调用方法

除了访问成员外，解引用强制转换还允许直接在源类型实例上调用目标类型上定义的方法。让我们用一个例子来说明这一点：

```cairo
struct MySource {
    pub data: u8,
}

struct MyTarget {
    pub data: u8,
}

#[generate_trait]
impl TargetImpl of TargetTrait {
    fn foo(self: MyTarget) -> u8 {
        self.data
    }
}

impl SourceDeref of Deref<MySource> {
    type Target = MyTarget;
    fn deref(self: MySource) -> MyTarget {
        MyTarget { data: self.data }
    }
}

#[executable]
fn main() {
    let source = MySource { data: 5 };
    // Thanks to the Deref impl, we can call foo directly on MySource
    let res = source.foo();
    assert!(res == 5);
}
```

在这个例子中，`MySource` 实现了解引用到 `MyTarget`。`MyTarget` 结构体有 trait `TargetTrait` 的实现 `TargetImpl`，该 trait 定义了一个方法 `foo`。因为 `MySource` 解引用到 `MyTarget`，我们可以直接在 `MySource` 的实例上调用 `foo` 方法，如 `main` 函数所示。

## 总结

通过使用 `Deref` 和 `DerefMut` traits，我们可以透明地将一种类型转换为另一种类型，简化对嵌套或包装数据结构的访问，并启用对目标类型上定义的方法的调用。当使用泛型类型或构建需要轻松访问底层数据的抽象时，此功能特别有用，并且可以帮助减少样板代码。
