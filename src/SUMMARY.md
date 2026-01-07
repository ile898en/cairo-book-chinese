# Cairo 书籍 (The Cairo Book)

[Cairo 书籍](title-page.md) [前言](ch00-01-foreword.md)
[简介](ch00-00-introduction.md)

# Cairo 编程语言 (The Cairo Programming Language)

## 入门 (Getting Started)

- [入门](ch01-00-getting-started.md)
  - [安装](ch01-01-installation.md)
  - [你好，世界！](ch01-02-hello-world.md)
  - [证明一个数是质数](ch01-03-proving-a-prime-number.md)

## 通用编程概念 (Common Programming Concepts)

- [通用编程概念](ch02-00-common-programming-concepts.md)
  - [变量和可变性](ch02-01-variables-and-mutability.md)
  - [数据类型](ch02-02-data-types.md)
  - [函数](ch02-03-functions.md)
  - [注释](ch02-04-comments.md)
  - [控制流](ch02-05-control-flow.md)

## 通用集合 (Common Collections)

- [通用集合](ch03-00-common-collections.md)
  - [数组](ch03-01-arrays.md)
  - [字典](ch03-02-dictionaries.md)

## 理解所有权 (Understanding Ownership)

- [理解所有权](ch04-00-understanding-ownership.md)
  - [什么是所有权？](ch04-01-what-is-ownership.md)
  - [引用和快照](ch04-02-references-and-snapshots.md)

## 使用结构体组织相关数据 (Using Structs to Structure Related Data)

- [使用结构体组织相关数据](ch05-00-using-structs-to-structure-related-data.md)
  - [定义和实例化结构体](ch05-01-defining-and-instantiating-structs.md)
  - [使用结构体的示例程序](ch05-02-an-example-program-using-structs.md)
  - [方法语法](ch05-03-method-syntax.md)

## 枚举和模式匹配 (Enums and Pattern Matching)

- [枚举和模式匹配](ch06-00-enums-and-pattern-matching.md)
  - [枚举](ch06-01-enums.md)
  - [Match 控制流结构](ch06-02-the-match-control-flow-construct.md)
  - [If Let 和 While Let 的简洁控制流](ch06-03-concise-control-flow-with-if-let-and-while-let.md)

## 使用包、Crates 和模块管理 Cairo 项目 (Managing Cairo Projects)

- [利用包、Crate 和模块管理 Cairo 项目](ch07-00-managing-cairo-projects-with-packages-crates-and-modules.md)
  - [包和 Crate](ch07-01-packages-and-crates.md)
  - [定义模块以控制作用域](ch07-02-defining-modules-to-control-scope.md)
  - [引用模块树中项的路径](ch07-03-paths-for-referring-to-an-item-in-the-module-tree.md)
  - [使用 use 关键字将路径引入作用域](ch07-04-bringing-paths-into-scope-with-the-use-keyword.md)
  - [将模块分割成不同的文件](ch07-05-separating-modules-into-different-files.md)

## 泛型数据类型 (Generic Data Types)

- [泛型类型和 Trait](ch08-00-generic-types-and-traits.md)
  - [泛型数据类型](ch08-01-generic-data-types.md)
  - [Cairo 中的 Trait](ch08-02-traits-in-cairo.md)

## 错误处理 (Error Handling)

- [错误处理](ch09-00-error-handling.md)
  - [使用 panic 的不可恢复错误](ch09-01-unrecoverable-errors-with-panic.md)
  - [使用 Result 的可恢复错误](ch09-02-recoverable-errors.md)

## 测试 Cairo 程序 (Testing Cairo Programs)

- [测试 Cairo 程序](ch10-00-testing-cairo-programs.md)
  - [如何编写测试](ch10-01-how-to-write-tests.md)
  - [测试组织](ch10-02-test-organization.md)

## 用 Rust 思考 (Thinking in Rust)

- [函数式语言特性：迭代器和闭包](ch11-00-functional-features.md)
  - [闭包：捕获其环境的匿名函数](ch11-01-closures.md)
  <!-- - [使用迭代器处理一系列项](ch11-02-iterators.md) -->

## 高级 Cairo 特性 (Advanced Cairo Features)

- [高级 Cairo 特性](ch12-00-advanced-features.md)
  - [自定义数据结构](ch12-01-custom-data-structures.md)
  - [智能指针](ch12-02-smart-pointers.md)
  - [Deref 强制转换](ch12-09-deref-coercion.md)
  - [关联项](ch12-10-associated-items.md)
  - [运算符重载](ch12-03-operator-overloading.md)
  - [使用哈希](ch12-04-hash.md)
  - [宏](ch12-05-macros.md)
  - [过程宏](ch12-10-procedural-macros.md)
  - [Cairo 中的内联](ch12-06-inlining-in-cairo.md)
  - [宏打印](ch12-08-printing.md)
  - [算术电路](ch12-10-arithmetic-circuits.md)
  - [使用 Oracle 卸载计算](ch12-11-offloading-computations-with-oracles.md)

- [附录 (Cairo)](appendix-00.md)
  - [A - 关键字](appendix-01-keywords.md)
  - [B - 运算符和符号](appendix-02-operators-and-symbols.md)
  - [C - 可派生 Trait](appendix-03-derivable-traits.md)
  - [D - Cairo 序幕 (Prelude)](appendix-04-cairo-prelude.md)
  - [E - 常见错误消息](appendix-05-common-error-messages.md)
  - [F - 有用的开发工具](appendix-06-useful-development-tools.md)

---

# Cairo 智能合约 (Smart Contracts in Cairo)

## Starknet 智能合约简介

- [智能合约简介](./ch100-00-introduction-to-smart-contracts.md)
  - [Starknet 合约类和实例](ch100-01-contracts-classes-and-instances.md)

## 构建 Starknet 智能合约

- [构建 Starknet 智能合约](./ch101-00-building-starknet-smart-contracts.md)
  - [Starknet 类型](./ch101-01-starknet-types.md)
  - [合约存储](./ch101-01-00-contract-storage.md)
    - [存储映射 (Mappings)](./ch101-01-01-storage-mappings.md)
    - [存储向量 (Vecs)](./ch101-01-02-storage-vecs.md)
  - [合约函数](./ch101-02-contract-functions.md)
  - [合约事件](./ch101-03-contract-events.md)

## Starknet 跨合约交互

- [Starknet 合约交互](./ch102-00-starknet-contract-interactions.md)
  - [合约类 ABI](./ch102-01-contract-class-abi.md)
  - [与另一个合约交互](./ch102-02-interacting-with-another-contract.md)
  - [执行来自另一个类的代码](./ch102-03-executing-code-from-another-class.md)
  - [Cairo 类型的序列化](./ch102-04-serialization-of-cairo-types.md)

## 构建高级 Starknet 智能合约

- [构建高级 Starknet 智能合约](./ch103-00-building-advanced-starknet-smart-contracts.md)
  - [优化存储成本](./ch103-01-optimizing-storage-costs.md)
  - [可组合性和组件](./ch103-02-00-composability-and-components.md)
    - [幕后花絮](./ch103-02-01-under-the-hood.md)
    - [组件依赖项](./ch103-02-02-component-dependencies.md)
    - [测试组件](./ch103-02-03-testing-components.md)
  - [可升级性](./ch103-03-upgradeability.md)
  - [L1 <> L2 消息传递](./ch103-04-L1-L2-messaging.md)
  - [Oracle 交互](./ch103-05-oracle-interactions.md)
    - [喂价](./ch103-05-01-price-feeds.md)
    - [随机性](./ch103-05-02-randomness.md)
  - [其他示例](./ch103-06-00-other-examples.md)
    - [部署投票合约并与之交互](./ch103-06-01-deploying-and-interacting-with-a-voting-contract.md)
    - [使用 ERC20 代币](./ch103-06-02-working-with-erc20-token.md)

## Starknet 智能合约安全

- [Starknet 智能合约安全](./ch104-00-starknet-smart-contracts-security.md)
  - [一般建议](./ch104-01-general-recommendations.md)
  - [测试智能合约](./ch104-02-testing-smart-contracts.md)
  - [静态分析工具](./ch104-03-static-analysis-tools.md)

## 附录 (Starknet)

- [附录 (Starknet)](appendix-000.md)
  - [A - 系统调用](appendix-08-system-calls.md)
  - [B - Sierra](appendix-09-sierra.md)

---

# Cairo 虚拟机 (Cairo VM)

- [简介](ch200-introduction.md)

- [架构](ch201-architecture.md)

## 内存 (Memory)

- [内存](ch202-00-memory.md)
  - [非确定性只读内存](ch202-01-non-deterministic-read-only-memory.md)
  - [段和重定位](ch202-02-segments.md)

## 执行模型 (Execution Model)

- [执行模型](ch203-00-execution-model.md)

## 内置函数 (Builtins)

- [内置函数](ch204-00-builtins.md)
  - [内置函数如何工作](ch204-01-how-builtins-work.md)
  - [内置函数列表](ch204-02-builtins-list.md)
    - [Output](ch204-02-00-output.md)
    - [Pedersen](ch204-02-01-pedersen.md)
    - [Range Check](ch204-02-02-range-check.md)
    - [ECDSA](ch204-02-03-ecdsa.md)
    - [Bitwise](ch204-02-04-bitwise.md)
    - [EC OP](ch204-02-05-ec-op.md)
    - [Keccak](ch204-02-06-keccak.md)
    - [Poseidon](ch204-02-07-poseidon.md)
    - [Mod Builtin](ch204-02-08-mod-builtin.md)
    - [Segment Arena](ch204-02-11-segment-arena.md)

## 提示 (Hints)

- [提示](./ch205-00-hints.md)

## 运行器 (Runner)

- [运行器](./ch206-00-runner.md)
  - [程序]()
    - [程序工件]()
    - [程序解析]()
  - [运行器模式]()
    - [执行模式]()
    - [证明模式]()
  - [输出]()
    - [Cairo PIE]()
    - [内存文件]()
    - [跟踪文件]()
    - [AIR 公共输入]()
    - [AIR 私有输入]()

- [跟踪器 (Tracer)]()

- [实现]()

- [资源]()
