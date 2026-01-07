# 过程宏 (Procedural Macros)

Cairo 提供了宏作为一项基本功能，让你能够编写生成其他代码的代码（称为元编程）。当你使用宏时，你可以将 Cairo 的能力扩展到常规函数所能提供的范围之外。在整本书中，我们要使用了像 `println!` 和 `assert!` 这样的宏，但还没有完全探索如何创建我们自己的宏。

在深入研究过程宏之前，让我们了解一下当我们已经有了函数时为什么还需要宏：

> 提示：对于许多表达式级用例，首选直接用 Cairo 编写的声明式内联宏（参见 [宏 → 通用元编程的声明式内联宏](./ch12-05-macros.md#declarative-inline-macros-for-general-metaprogramming)）。
> 当你需要属性/派生或跨项操作或需要 Rust 侧逻辑的高级转换时，请使用过程宏。

## Cairo 过程宏是 Rust 函数

就像 Cairo 编译器是用 Rust 编写的一样，过程宏是转换 Cairo 代码的 Rust 函数。这些函数接受 Cairo 代码作为输入，并返回修改后的 Cairo 代码作为输出。要实现宏，你需要一个同时具有 `Cargo.toml` 和 `Scarb.toml` 文件的包。`Cargo.toml` 定义宏实现依赖项，而 `Scarb.toml` 将包标记为宏并定义其元数据。

定义过程宏的函数对两个关键类型进行操作：

- `TokenStream`: 代表你的源代码的 Cairo 标记序列。标记是编译器识别的最小代码单元（如标识符、关键字和运算符）。
- `ProcMacroResult`: `TokenStream` 的增强版本，包括生成的代码和应该在编译期间向用户显示的任何诊断消息（警告或错误）。

实现宏的函数必须用三个特殊属性之一进行装饰，这些属性告诉编译器应该如何使用宏：

- `#[inline_macro]`: 用于看起来像函数调用的宏（例如 `println!()`）
- `#[attribute_macro]`: 用于充当属性的宏（例如 `#[generate_trait]`）
- `#[derive_macro]`: 用于自动实现 traits 的宏

每个属性类型对应于不同的用例，并影响宏如何在你的代码中被调用。

以下是每种类型的签名：

```rust, ignore
#[inline_macro]
pub fn inline(code: TokenStream) -> ProcMacroResult {}

#[attribute_macro]
pub fn attribute(attr: TokenStream, code: TokenStream) -> ProcMacroResult {}

#[derive_macro]
pub fn derive(code: TokenStream) -> ProcMacroResult {}
```

### 安装依赖

要使用过程宏，你需要在机器上安装 Rust 工具链 (Cargo)。要使用 Rustup 安装 Rust，你可以在终端中运行以下命令：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## 创建你的宏

创建过程宏需要设置特定的项目结构。你的宏项目需要：

1. 一个 Rust 项目（你在其中实现宏）：
   - `Cargo.toml`: 定义 Rust 依赖项和构建设置
   - `src/lib.rs`: 包含宏实现

2. 一个 Cairo 项目：
   - `Scarb.toml`: 为 Cairo 项目声明宏
   - 不需要 Cairo 源文件

让我们逐步了解每个组件并理解其作用：

```bash
├── Cargo.toml
├── Scarb.toml
├── src
│   └── lib.rs
```

项目根目录包含 `Scarb.toml` 和 `Cargo.toml` 文件。

Cargo 清单文件需要在 `[lib]` 目标上包含 `crate-type = ["cdylib"]`，并在 `[dependencies]` 目标上包含 `cairo-lang-macro` crate。这是一个例子：

```toml
[package]
name = "pow"
version = "0.1.0"
edition = "2021"
publish = false

[lib]
crate-type = ["cdylib"]

[dependencies]
bigdecimal = "0.4.5"
cairo-lang-macro = "0.1.1"
cairo-lang-parser = "2.13.1"
cairo-lang-syntax = "2.13.1"

[workspace]
```

Scarb 清单文件必须定义 `[cairo-plugin]` 目标类型。这是一个例子：

```toml
[package]
name = "pow"
version = "0.1.0"

[cairo-plugin]
```

最后，项目需要包含一个 Rust 库 (`lib.rs`)，在 `src/` 目录下实现过程宏 API。

正如你可能注意到的，该项目不需要任何 cairo 代码，它只需要提到的 `Scarb.toml` 清单文件。

## 使用你的宏

从用户的角度来看，你只需要在依赖项中添加定义宏的包。在使用宏的项目中，你的 Scarb 清单文件将包含：

```toml
[package]
name = "no_listing_15_procedural_macro"
version = "0.1.0"
edition = "2024_07"

# See more keys and their definitions at https://docs.swmansion.com/scarb/docs/reference/manifest.html

[dependencies]
cairo_execute = "2.13.1"
pow = { path = "../no_listing_16_procedural_macro_expression" }
hello_macro = { path = "../no_listing_17_procedural_macro_derive" }
rename_macro = { path = "../no_listing_18_procedural_macro_attribute" }


[dev-dependencies]
cairo_test = "2.13.1"

[cairo]
enable-gas = false


[[target.executable]]
name = "main"
function = "no_listing_15_procedural_macro::main"
```

## 表达式宏 (Expression Macros)

注意：如果你的目标是生成或转换表达式和小块，前面介绍的内联声明式宏更易于编写和维护。当你特别需要 Rust 驱动的解析或当你的宏逻辑位于 Cairo 之外时，请使用过程表达式宏。作为一个具体的例子，让我们看一个作为过程宏实现的编译时幂函数。

### 创建表达式宏

为了理解如何创建表达式宏，我们将查看来自 [Alexandria](https://github.com/keep-starknet-strange/alexandria) 库的 `pow` 宏实现，它在编译时计算数字的幂。

宏实现的核心代码是使用三个 Rust crates 的 Rust 代码：特定于宏实现的 `cairo_lang_macro`，具有与编译器解析器相关功能的 `cairo_lang_parser` crate，以及与编译器语法相关的 `cairo_lang_syntax`。后两者最初是为 Cairo 语言编译器创建的，由于宏函数在 Cairo 语法级别操作，我们可以直接重用为编译器创建的语法函数中的逻辑来创建宏。

> **注意：** 要更好地理解 Cairo 编译器以及我们这里仅提到的一些概念（如 Cairo 解析器或 Cairo 语法），你可以阅读 [Cairo 编译器研讨会](https://github.com/software-mansion-labs/cairo-compiler-workshop)。

在下面的 `pow` 函数示例中，处理输入以提取底数参数和指数参数的值，以返回 \\(base^{exponent}\\) 的结果。

```rust, noplayground
use bigdecimal::{num_traits::pow, BigDecimal};
use cairo_lang_macro::{inline_macro, Diagnostic, ProcMacroResult, TokenStream};
use cairo_lang_parser::utils::SimpleParserDatabase;

#[inline_macro]
pub fn pow(token_stream: TokenStream) -> ProcMacroResult {
    let db = SimpleParserDatabase::default();
    let (parsed, _diag) = db.parse_virtual_with_diagnostics(token_stream);

    // extracting the args from the parsed input
    let macro_args: Vec<String> = parsed
        .descendants(&db)
        .next()
        .unwrap()
        .get_text(&db)
        .trim_matches(|c| c == '(' || c == ')')
        .split(',')
        .map(|s| s.trim().to_string())
        .collect();

    if macro_args.len() != 2 {
        return ProcMacroResult::new(TokenStream::empty()).with_diagnostics(
            Diagnostic::error(format!("Expected two arguments, got {:?}", macro_args)).into(),
        );
    }

    // getting the value from the base arg
    let base: BigDecimal = match macro_args[0].parse() {
        Ok(val) => val,
        Err(_) => {
            return ProcMacroResult::new(TokenStream::empty())
                .with_diagnostics(Diagnostic::error("Invalid base value").into());
        }
    };

    // getting the value from the exponent arg
    let exp: usize = match macro_args[1].parse() {
        Ok(val) => val,
        Err(_) => {
            return ProcMacroResult::new(TokenStream::empty())
                .with_diagnostics(Diagnostic::error("Invalid exponent value").into());
        }
    };

    // base^exp
    let result: BigDecimal = pow(base, exp);

    ProcMacroResult::new(TokenStream::new(result.to_string()))
}
```

现在宏已定义，我们可以使用它了。在 Cairo 项目中，我们需要在 `Scarb.toml` 清单文件的 `[dependencies]` 目标中有 `pow = { path = "path/to/pow" }`。然后我们可以像这样无需进一步导入即可使用它：

```cairo
    let res = pow!(10, 2);
    println!("res : {}", res);
```

## 派生宏 (Derive Macros)

派生宏允许你定义可以自动应用于类型的自定义 trait 实现。当你用 `#[derive(TraitName)]` 注释类型时，你的派生宏：

1. 接收类型的结构作为输入
2. 包含你生成 trait 实现的自定义逻辑
3. 输出将包含在 crate 中的实现代码

编写派生宏通过使用关于如何生成 trait 实现的通用逻辑消除了重复的 trait 实现代码。

### 创建派生宏

在这个例子中，我们将实现一个派生宏，它将实现 `Hello` Trait。`Hello` trait 将有一个 `hello()` 函数，它将打印：`Hello, StructName!`，其中 _StructName_ 是结构体的名称。

这是 `Hello` trait 的定义：

```cairo
trait Hello<T> {
    fn hello(self: @T);
}
```

让我们检查宏实现，首先 `hello_derive` 函数解析输入标记流，然后提取 `struct_name` 以为该特定结构体实现 trait。

然后 hello derived 返回一段硬编码的代码，其中包含类型 _StructName_ 的 `Hello` trait 的实现。

```rust, noplayground
use cairo_lang_macro::{derive_macro, ProcMacroResult, TokenStream};
use cairo_lang_parser::utils::SimpleParserDatabase;
use cairo_lang_syntax::node::kind::SyntaxKind::{TerminalStruct, TokenIdentifier};

#[derive_macro]
pub fn hello_macro(token_stream: TokenStream) -> ProcMacroResult {
    let db = SimpleParserDatabase::default();
    let (parsed, _diag) = db.parse_virtual_with_diagnostics(token_stream);
    let mut nodes = parsed.descendants(&db);

    let mut struct_name = String::new();
    for node in nodes.by_ref() {
        if node.kind(&db) == TerminalStruct {
            struct_name = nodes
                .find(|node| node.kind(&db) == TokenIdentifier)
                .unwrap()
                .get_text(&db)
                .to_string();
            break;
        }
    }

    ProcMacroResult::new(TokenStream::new(indoc::formatdoc! {r#"
            impl SomeHelloImpl of Hello<{0}> {{
                fn hello(self: @{0}) {{
                    println!("Hello {0}!");
                }}
            }}
        "#, struct_name}))
}
```

现在宏已定义，我们可以使用它了。在 Cairo 项目中，我们需要在 `Scarb.toml` 清单文件的 `[dependencies]` 目标中有 `hello_macro = { path = "path/to/hello_macro" }`。然后我们可以在任何结构体上无需进一步导入即可使用它：

```cairo, noplayground
#[derive(HelloMacro, Drop, Destruct)]
struct SomeType {}
```

现在我们可以对 `SomeType` 类型的变量调用已实现的函数 `hello`。

```cairo, noplayground
    let a = SomeType {};
    a.hello();
```

注意，宏中实现的 `Hello` trait 必须在代码中的某处定义或导入。

## 属性宏 (Attribute Macros)

属性类宏类似于自定义派生宏，但允许更多的可能性，它们不限于结构体和枚举，也可以应用于其他项，例如函数。它们可用于比实现 trait 更通过的代码生成。它可用于修改结构体的名称、在他机构中添加字段、在函数之前执行某些代码、更改函数的签名以及许多其他可能性。

额外的可能性也来自于它们是用第二个参数 `TokenStream` 定义的这一事实，实际上签名看起来像这样：

```rust, noplayground
#[attribute_macro]
pub fn attribute(attr: TokenStream, code: TokenStream) -> ProcMacroResult {}
```

第一个参数 (`attr`) 用于属性参数 (#[macro(arguments)])，第二个参数用于应用属性的实际代码，第二个参数是其他两个宏所拥有的唯一参数。

### 创建属性宏

现在让我们看一个自定义属性宏的例子，在这个例子中我们将创建一个重命名结构体的宏。

```rust, noplayground
use cairo_lang_macro::attribute_macro;
use cairo_lang_macro::{ProcMacroResult, TokenStream};

#[attribute_macro]
pub fn rename(_attr: TokenStream, token_stream: TokenStream) -> ProcMacroResult {
    ProcMacroResult::new(TokenStream::new(
        token_stream
            .to_string()
            .replace("struct OldType", "#[derive(Drop)]\n struct RenamedType"),
    ))
}
```

同样，要在 Cairo 项目中使用该宏，我们需要在 `Scarb.toml` 清单文件的 `[dependencies]` 目标中有 `rename_macro = { path = "path/to/rename_macro" }`。然后我们可以在任何结构体上无需进一步导入即可使用它。

重命名宏可以如下派生：

```cairo
#[rename]
struct OldType {}
```

现在编译器知道了 _RenamedType_ 结构体，因此我们可以像这样创建一个实例：

```cairo
    let _a = RenamedType {};
```

你可以注意到 _OldType_ 和 _RenamedType_ 的名称在示例中是硬编码的，但可以是利用属性宏第二个参数的变量。另请注意，由于编译顺序的原因，其他宏（如这里的 _Drop_）的派生必须在宏生成的代码中完成。创建自定义宏可能需要对 Cairo 编译有更深入的了解。
