# 运算符重载

运算符重载是一些编程语言中的一个特性，允许重新定义标准运算符，例如加法 (`+`)、减法 (`-`)、乘法 (`*`) 和除法 (`/`)，以便与用户定义的类型一起使用。这可以通过使对用户定义类型的操作能够以与对原始类型的操作相同的方式表达，从而使代码的语法更加直观。

在 Cairo 中，运算符重载是通过实现特定的 traits 来实现的。每个运算符都有一个关联的 trait，重载该运算符涉及为自定义类型提供该 trait 的实现。然而，必须明智地使用运算符重载。滥用会导致混乱，使代码更难维护，例如当被重载的运算符没有语义意义时。

考虑一个例子，其中需要组合两个 `Potion`（药水）。`Potion` 有两个数据字段，mana（法力值）和 health（生命值）。组合两个 `Potion` 应该将它们各自的字段相加。

```cairo
struct Potion {
    health: felt252,
    mana: felt252,
}

impl PotionAdd of Add<Potion> {
    fn add(lhs: Potion, rhs: Potion) -> Potion {
        Potion { health: lhs.health + rhs.health, mana: lhs.mana + rhs.mana }
    }
}

#[executable]
fn main() {
    let health_potion: Potion = Potion { health: 100, mana: 0 };
    let mana_potion: Potion = Potion { health: 0, mana: 100 };
    let super_potion: Potion = health_potion + mana_potion;
    // Both potions were combined with the `+` operator.
    assert!(super_potion.health == 100);
    assert!(super_potion.mana == 100);
}
```

在上面的代码中，我们为 `Potion` 类型实现了 `Add` trait。add 函数接受两个参数：`lhs` 和 `rhs`（左手边和右手边）。函数体返回一个新的 `Potion` 实例，其字段值是 `lhs` 和 `rhs` 的组合。

如示例所示，重载运算符需要指定被重载的具体类型。重载的泛型 trait 是 `Add<T>`，我们使用 `Add<Potion>` 为 `Potion` 类型定义了一个具体实现。

{{#quiz ../quizzes/ch12-03-operator-overloading.toml}}
