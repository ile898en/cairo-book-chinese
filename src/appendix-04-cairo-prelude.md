# 附录 D - Cairo Prelude

## Prelude

Cairo prelude 是常用模块、函数、数据类型和 traits 的集合，它们会自动引入 Cairo crate 中每个模块的作用域，而无需显式导入语句。Cairo 的 prelude 提供了开发人员开始编写 Cairo 程序和智能合约所需的基本构建块。

核心库 prelude 定义在 corelib crate 的 _[lib.cairo](https://github.com/starkware-libs/cairo/blob/main/corelib/src/lib.cairo)_ 文件中，包含 Cairo 的原始数据类型、traits、运算符和实用函数。这包括：

- 数据类型：整数、布尔值、数组、字典等。
- Traits：算术、比较和序列化操作的行为
- 运算符：算术、逻辑、按位
- 实用函数 - 数组、映射、装箱 (boxing) 等的助手

核心库 prelude 提供了基本 Cairo 程序所需的基础编程结构和操作，而无需显式导入元素。由于核心库 prelude 是自动导入的，因此其内容可在任何 Cairo crate 中使用，无需显式导入。这防止了重复并提供了更好的开发体验 (devX)。这就是允许你使用 `ArrayTrait::append()` 或 `Default` trait 而无需显式将它们引入作用域的原因。

你可以选择使用哪个 prelude。例如，在 _Scarb.toml_ 配置文件中添加 `edition = "2024_07"` 将加载 2024 年 7 月的 prelude。注意，当你使用 `scarb new` 命令创建新项目时，_Scarb.toml_ 文件将自动包含 `edition = "2024_07"`。不同的 prelude 版本将公开不同的函数和 traits，因此在 _Scarb.toml_ 文件中指定正确的版本非常重要。通常，你会希望使用该最新版本开始新项目，并随着新版本的发布迁移到更新的版本。

## Cairo 版本 (editions)

以下是可用 Cairo 版本（即 prelude 版本）及其详细信息的列表：

| 版本                 | 详情                                                                                                                           |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `2024-07`            | [2024-07 的详细信息](https://community.starknet.io/t/cairo-v2-7-0-is-coming/114362#the-2024_07-edition-3)                       |
| `2023-11`            | [2023-11 的详细信息](https://community.starknet.io/t/cairo-v2-5-0-is-out/112807#the-pub-keyword-9)                              |
| `2023-10` / `2023-1` | [2023-10 的详细信息](https://community.starknet.io/t/cairo-v2-4-0-is-out/109275#editions-and-the-introduction-of-preludes-10) |
