# 泛型类型和 Traits

每种编程语言都有有效处理概念重复的工具。在 Cairo 中，泛型 (generics) 就是这样一个工具：具体类型或其他属性的抽象替代品。我们可以在不知道编译和运行代码时具体是什么类型的情况下，表达泛型的行为或它们与其他泛型的关系。

函数可以接受某种泛型类型的参数，而不是像 `u32` 或 `bool` 这样的具体类型，就像函数接受具有未知值的参数以在多个具体值上运行相同的代码一样。事实上，我们已经在 [第 {{#chap enums-and-pattern-matching}} 章][option enum] 中使用了 `Option<T>` 的泛型。

在本章中，你将探索如何定义你自己的带有泛型的类型、函数和 traits。

泛型允许我们用代表多种类型的占位符替换特定类型，以消除代码重复。编译时，编译器会为替换泛型类型的每个具体类型创建一个新定义，从而减少程序员的开发时间，但编译级别的代码重复仍然存在。如果你正在编写 Starknet 合约并为多种类型使用泛型，这可能很重要，因为这将导致合约大小增加。

然后你将学习如何使用 traits 以通用的方式定义行为。你可以将 traits 与泛型类型结合使用，以约束泛型类型只接受那些具有特定行为的类型，而不仅仅是任何类型。

[option enum]: ./ch06-01-enums.md#the-option-enum-and-its-advantages

## 通过提取函数消除重复

泛型允许我们用代表多种类型的占位符替换特定类型，以消除代码重复。在深入研究泛型语法之前，让我们首先看看如何通过提取一个用代表多个值的占位符替换特定值的函数，以一种不涉及泛型类型的方式消除重复。然后我们将应用相同的技术来提取一个泛型函数！通过学习如何识别可以提取到函数中的重复代码，你将开始认识到可以使用泛型来减少重复的情况。

我们从一个寻找 `u8` 数组中最大数字的简短程序开始：

```cairo
#[executable]
fn main() {
    let mut number_list: Array<u8> = array![34, 50, 25, 100, 65];

    let mut largest = number_list.pop_front().unwrap();

    for number in number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {}", largest);
}
```

我们将一个 `u8` 数组存储在变量 `number_list` 中，并将数组中的第一个数字提取到名为 `largest` 的变量中。然后我们遍历数组中的所有数字，如果当前数字大于存储在 `largest` 中的数字，我们更新 `largest` 的值。但是，如果当前数字小于或等于目前看到的最大数字，变量不会改变，代码将继续处理列表中的下一个数字。在考虑了数组中的所有数字后，`largest` 应该包含最大的数字，在这个例子中是 100。

现在的任务是在两个不同的数字数组中找到最大的数字。为此，我们可以选择重复之前的代码，并在程序的两个不同位置使用相同的逻辑，如下所示：

```cairo
#[executable]
fn main() {
    let mut number_list: Array<u8> = array![34, 50, 25, 100, 65];

    let mut largest = number_list.pop_front().unwrap();

    for number in number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {}", largest);

    let mut number_list: Array<u8> = array![102, 34, 255, 89, 54, 2, 43, 8];

    let mut largest = number_list.pop_front().unwrap();

    for number in number_list {
        if number > largest {
            largest = number;
        }
    }

    println!("The largest number is {}", largest);
}
```

虽然这段代码有效，但复制代码既乏味又容易出错。当我们想更改代码时，咱们还必须记住在多个地方更新代码。

为了消除这种重复，我们将通过定义一个对参数中传入的任何 `u8` 数组进行操作的函数来创建一个抽象。这个解决方案使我们的代码更清晰，并让我们能够抽象地表达在数组中寻找最大数字的概念。

为此，我们将寻找最大数字的代码提取到一个名为 `largest` 的函数中。然后我们调用该函数来查找两个数组中的最大数字。我们也可以在将来可能拥有的任何其他 `u8` 值数组上使用该函数。

```cairo
fn largest(ref number_list: Array<u8>) -> u8 {
    let mut largest = number_list.pop_front().unwrap();

    for number in number_list.span() {
        if *number > largest {
            largest = *number;
        }
    }

    largest
}

#[executable]
fn main() {
    let mut number_list = array![34, 50, 25, 100, 65];

    let result = largest(ref number_list);
    println!("The largest number is {}", result);

    let mut number_list = array![102, 34, 255, 89, 54, 2, 43, 8];

    let result = largest(ref number_list);
    println!("The largest number is {}", result);
}
```

largest 函数有一个名为 `number_list` 的参数，通过引用传递，它代表我们可以传递给函数的任何具体的 `u8` 值数组。结果，当我们调用该函数时，代码在我们传入的特定值上运行。

总之，这是我们要更改代码所采取的步骤：

- 识别重复代码。
- 将重复代码提取到函数体中，并在函数签名中指定该代码的输入和返回值。
- 更新重复代码的两个实例以改为调用该函数。

接下来，我们将对泛型使用相同的步骤来减少代码重复。就像函数体可以对抽象的 `Array<T>` 而不是特定的 `u8` 值进行操作一样，泛型允许代码对抽象类型进行操作。
