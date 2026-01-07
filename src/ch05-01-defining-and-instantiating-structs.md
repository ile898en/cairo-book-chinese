# 定义并实例化结构体

结构体和我们在 [数据类型](ch02-02-data-types.md) 部分讨论过的元组类似，它们都包含多个相关的值。和元组一样，结构体的每一部分可以是不同类型的。但不同于元组，结构体需要给每一部分数据命名，以便清楚它们表示什么。因为有了这些名字，结构体比元组更灵活：不需要依赖数据的顺序来指定或访问实例中的值。

定义结构体，需要使用 `struct` 关键字并为整个结构体命名。结构体的名字应该描述被组合在一起的数据的意义。然后，在花括号中，定义每一部分数据的名字和类型，我们称为字段 (field)。例如，清单 {{#ref user-struct}} 展示了一个存储用户账号信息的结构体。

<span class="filename">文件名: src/lib.cairo</span>

```cairo, noplayground
#[derive(Drop)]
struct User {
    active: bool,
    username: ByteArray,
    email: ByteArray,
    sign_in_count: u64,
}
```

{{#label user-struct}}

<span class="caption">清单 {{#ref user-struct}}: `User` 结构体定义</span>

> **注意 :**  
> 你可以在结构体上派生多个 trait，例如用于比较的 `Drop`、`PartialEq` 和用于调试打印的 `Debug`。  
> 请参阅 [附录：可派生 Traits](./appendix-03-derivable-traits.md) 以获取完整列表和示例。

定义了结构体后，为了使用它，我们需要通过为每个字段指定具体值来创建该结构体的 _实例_。我们通过声明结构体的名字，然后添加包含 _key: value_ 对的花括号来创建实例，其中键是字段的名字，值是我们想要存储在这些字段中的数据。我们不需要按照在结构体中声明的顺序指定字段。换句话说，结构体定义就像是类型的通用模板，而实例则用特定数据填充该模板以创建该类型的值。

例如，我们可以声明两个特定的用户，如清单 {{#ref user-instances}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
// ANCHOR: user
#[derive(Drop)]
struct User {
    active: bool,
    username: ByteArray,
    email: ByteArray,
    sign_in_count: u64,
}
// ANCHOR_END: user

// ANCHOR: main
#[executable]
fn main() {
    let user1 = User {
        active: true, username: "someusername123", email: "someone@example.com", sign_in_count: 1,
    };
    let user2 = User {
        sign_in_count: 1, username: "someusername123", active: true, email: "someone@example.com",
    };
}
// ANCHOR_END: main

```

{{#label user-instances}} <span class="caption">清单 {{#ref user-instances}}:
创建 `User` 结构体的两个实例</span>

要从结构体中获取特定值，我们使用点号。例如，要访问 `user1` 的电子邮件地址，我们使用 `user1.email`。如果实例是可变的，我们可以通过使用点号并赋值给特定字段来更改值。清单 {{#ref user-mut}} 展示了如何更改可变 `User` 实例的 `email` 字段中的值。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[executable]
fn main() {
    let mut user1 = User {
        active: true, username: "someusername123", email: "someone@example.com", sign_in_count: 1,
    };
    user1.email = "anotheremail@example.com";
}
```

{{#label user-mut}} <span class="caption">清单 {{#ref user-mut}}: 更改 `User` 实例 `email` 字段的值</span>

注意整个实例必须是可变的；Cairo 不允许我们仅将某些字段标记为可变。

和任何表达式一样，我们可以构造一个新的结构体实例作为函数体的最后一个表达式，以隐式返回该新实例。

清单 {{#ref build-user}} 展示了一个 `build_user` 函数，它返回一个带有给定电子邮件和用户名的 `User` 实例。`active` 字段的值为 `true`，`sign_in_count` 的值为 `1`。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
fn build_user(email: ByteArray, username: ByteArray) -> User {
    User { active: true, username: username, email: email, sign_in_count: 1 }
}
```

{{#label build-user}} <span class="caption">清单 {{#ref build-user}}: 一个 `build_user` 函数，它接受电子邮件和用户名并返回一个 `User` 实例。</span>

使用与结构体字段相同的名称来命名函数参数是有意义的，但是必须重复 `email` 和 `username` 字段名称和变量有点繁琐。如果结构体有更多字段，重复每个名称会变得更加烦人。幸运的是，有一个方便的简写！

## 使用字段初始化简写

因为清单 {{#ref build-user}} 中的参数名称和结构体字段名称完全相同，我们可以使用字段初始化简写语法重写 `build_user`，使其行为完全相同，但没有 `username` 和 `email` 的重复，如清单 {{#ref init-shorthand}} 所示。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
fn build_user_short(email: ByteArray, username: ByteArray) -> User {
    User { active: true, username, email, sign_in_count: 1 }
}
```

{{#label init-shorthand}} <span class="caption">清单 {{#ref init-shorthand}}:
一个使用字段初始化简写的 `build_user` 函数，因为 `username` 和 `email` 参数与结构体字段名称相同。</span>

在这里，我们正在创建一个 `User` 结构体的新实例，它有一个名为 `email` 的字段。我们想要将 `email` 字段的值设置为 `build_user` 函数的 `email` 参数中的值。因为 `email` 字段和 `email` 参数具有相同的名称，我们只需要写 `email` 而不是 `email: email`。

## 使用结构体更新语法从其他实例创建实例

创建一个新结构体实例，其中包含来自另一个实例的大部分值，但更改了一些值，这通常很有用。你可以使用 _结构体更新语法_ 来做到这一点。

首先，在清单 {{#ref without-update-syntax}} 中，我们展示了如何在 `user2` 中常规创建一个新的 `User` 实例，而不使用更新语法。我们要为 `email` 设置一个新值，但在其他方面使用我们在清单 {{#ref user-instances}} 中创建的 `user1` 中的相同值。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[executable]
fn main() {
    // --snip--

    let user2 = User {
        active: user1.active,
        username: user1.username,
        email: "another@example.com",
        sign_in_count: user1.sign_in_count,
    };
}
```

{{#label without-update-syntax}}

<span class="caption">清单 {{#ref without-update-syntax}}: 使用 `user1` 中的除一个值之外的所有值创建一个新的 `User` 实例</span>

使用结构体更新语法，我们可以用更少的代码实现相同的效果，如清单 {{#ref update-syntax}} 所示。语法 `..` 指定未显式设置的剩余字段应具有与给定实例中的字段相同的值。

<span class="filename">文件名: src/lib.cairo</span>

```cairo
#[executable]
fn main() {
    // --snip--

    let user2 = User { email: "another@example.com", ..user1 };
}
```

{{#label update-syntax}}

<span class="caption">清单 {{#ref update-syntax}}: 使用结构体更新语法为一个 `User` 实例设置新的 `email` 值，但使用 `user1` 中的其余值</span>

清单 {{#ref update-syntax}} 中的代码也创建了一个 `user2` 实例，它的 `email` 值不同，但在 `username`、`active` 和 `sign_in_count` 字段上的值与 `user1` 相同。`..user1` 部分必须放在最后，以指定任何剩余字段应从 `user1` 中的相应字段获取其值，但我们可以选择以任何顺序为任意数量的字段指定值，而不管结构体定义中字段的顺序如何。

注意结构体更新语法像赋值一样使用 `=`；这是因为它移动了数据，就像我们在 ["移动值"][move]<!-- ignore --> 部分看到的那样。在这个例子中，我们在创建 `user2` 后不能再作为一个整体使用 `user1`，因为 `user1` 的 `username` 字段中的 `ByteArray` 被移入到了 `user2` 中。如果我们给 `user2` 的 `email` 和 `username` 都提供了新的 `ByteArray` 值，从而只使用了 `user1` 中的 `active` 和 `sign_in_count` 值，那么 `user1` 在创建 `user2` 后仍然有效。`active` 和 `sign_in_count` 都是实现了 `Copy` trait 的类型，所以我们在 ["`Copy` Trait"][copy]<!-- ignore --> 部分讨论的行为将适用。

{{#quiz ../quizzes/ch05-01-defining-and-instantiating-structs.toml}}

[move]: ch04-01-what-is-ownership.md#moving-values
[copy]: ch04-01-what-is-ownership.md#the-copy-trait
