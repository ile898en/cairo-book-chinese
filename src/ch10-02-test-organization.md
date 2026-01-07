# 测试组织

我们将从两个主要类别来考虑测试：单元测试和集成测试。单元测试很小且更专注，一次独立测试一个模块，并且可以测试私有函数。集成测试以与其他外部代码相同的方式使用你的代码，仅使用公共接口，并且每个测试可能练习多个模块。

编写这两种测试对于确保你的库的各个部分分别和一起按预期工作都很重要。

## 单元测试

单元测试的目的是将每个代码单元与其余代码隔离进行测试，以快速查明代码在何处按预期工作，在何处不工作。你会将单元测试放在每个文件的 `src` 目录中，与它们正在测试的代码在一起。

惯例是在每个文件中创建一个名为 `tests` 的模块以包含测试函数，并使用 `#[cfg(test)]` 属性注释该模块。

### 测试模块和 `#[cfg(test)]`

测试模块上的 `#[cfg(test)]` 注释告诉 Cairo 仅在运行 `scarb test` 时编译和运行测试代码，而不是在运行 `scarb build` 时。当你只想构建项目时，这可以节省编译时间，并节省生成的已编译工件中的空间，因为测试不包含在内。你会看到，因为集成测试在不同的目录中，它们不需要 `#[cfg(test)]` 注释。但是，因为单元测试与代码位于相同的文件中，你将使用 `#[cfg(test)]` 来指定它们不应包含在已编译的结果中。

回想一下，当我们于本章第一节创建新的 `adder` 项目时，我们编写了这第一个测试：

```cairo
pub fn add(left: usize, right: usize) -> usize {
    left + right
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        let result = add(2, 2);
        assert_eq!(result, 4);
    }
}
```

属性 `cfg` 代表 _配置 (configuration)_ ，并告诉 Cairo 只有在给定的配置选项下才应包含以下项目。在这种情况下，配置选项是 `test`，这是 Cairo 用于编译和运行测试的。通过使用 `cfg` 属性，只有当我们使用 `scarb test` 积极运行测试时，Cairo 才会编译我们的测试代码。这包括可能在此模块内的任何辅助函数，以及用 `#[test]` 注释的函数。

### 测试私有函数

测试社区内关于是否应该直接测试私有函数存在争议，其他语言使得测试私有函数变得困难或不可能。无论你坚持哪种测试意识形态，Cairo 的隐私规则确实允许你测试私有函数。
考虑下面带有私有函数 `internal_adder` 的代码。

<span class="caption">文件名: src/lib.cairo</span>

```cairo, noplayground
pub fn add(a: u32, b: u32) -> u32 {
    internal_adder(a, 2)
}

fn internal_adder(a: u32, b: u32) -> u32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add() {
        assert_eq!(4, internal_adder(2, 2));
    }
}
```

{{#label test_internal}} <span class="caption">清单 {{#ref test_internal}}:
测试私有函数</span>

注意 `internal_adder` 函数没有被标记为 `pub`。测试只是 Cairo 代码，测试模块只是另一个模块。正如我们在 [“引用模块树中项目的路径”](ch07-03-paths-for-referring-to-an-item-in-the-module-tree.md) 部分中讨论的那样，子模块中的项目可以使用其祖先模块中的项目。在这个测试中，我们将 `tests` 模块的父级 `internal_adder` 引入范围 `use super::internal_adder;`，然后测试可以调用 `internal_adder`。如果你不认为应该测试私有函数，Cairo 中没有任何东西会强迫你这样做。

## 集成测试

集成测试以与其他代码相同的方式使用你的库。它们的目的是测试你的库的许多部分是否可以正确地协同工作。单独工作正常的代码单元在集成时可能会出现问题，因此集成代码的测试覆盖率也很重要。要创建集成测试，你首先需要一个 _tests_ 目录。

### _tests_ 目录

我们在项目目录的顶层创建一个 _tests_ 目录，在 _src_ 旁边。Scarb 知道要在此目录中查找集成测试文件。然后我们可以制作任意数量的测试文件，Scarb 将把每个文件编译为一个单独的 crate。

让我们创建一个集成测试。在 _src/lib.cairo_ 文件中仍然保留清单 {{#ref test_internal}} 中的代码的情况下，创建一个 _tests_ 目录，并创建一个名为 _tests/integration_test.cairo_ 的新文件。你的目录结构应如下所示：

```shell
adder
├── Scarb.lock
├── Scarb.toml
├── src
│   └── lib.cairo
└── tests
    └── integration_tests.cairo

```

将清单 {{#ref test_integration}} 中的代码输入到 _tests/integration_test.cairo_ 文件中：

<span class="caption">文件名: tests/integration_tests.cairo</span>

```cairo, noplayground
use adder::add_two;

#[test]
fn it_adds_two() {
    assert_eq!(4, add_two(2));
}
```

{{#label test_integration}} <span class="caption">清单
{{#ref test_integration}}: `adder` crate 中函数的一个集成测试</span>

`tests` 目录中的每个文件都是一个单独的 crate，因此我们需要将我们的库引入每个测试 crate 的范围。因为这个原因，我们在代码顶部添加 `use adder::add_two`，我们在单元测试中不需要这样做。

我们不需要用 `#[cfg(test)]` 注释 _tests/integration_test.cairo_ 中的任何代码。Scarb 会特殊对待 tests 目录，并且仅在我们运行 `scarb test` 时才编译此目录中的文件。现在运行 `scarb test`：

```shell
$ scarb test 
     Running test adder (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(adder_unittest) adder v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_09_integration_test/Scarb.toml)
   Compiling test(adder_integrationtest) adder_integrationtest v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_09_integration_test/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from adder package
Running 1 test(s) from tests/
[PASS] adder_integrationtest::integration_tests::it_adds_two (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Running 1 test(s) from src/
[PASS] adder::tests::internal (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

输出的两个部分包括单元测试和集成测试。注意，如果某个部分中的任何测试失败，则不会运行后续部分。例如，如果单元测试失败，则通过不会有集成测试的任何输出，因为只有在所有单元测试都通过的情况下才会运行这些测试。

显示的第一个部分是针对集成测试的。每个集成测试文件都有自己的部分，因此如果我们在 _tests_ 目录中添加更多文件，就会有更多集成测试部分。

显示的第二部分与我们可以看到的一样：每个单元测试一行（我们刚刚在上面添加的名叫 add 的那个），然后是单元测试的摘要行。

我们仍然可以通过将测试函数的名称作为 `scarb test` 的 `-f` 选项的参数来运行特定的集成测试函数，例如 `scarb test -f integration_tests::internal`。要运行特定集成测试文件中的所有测试，我们使用相同的 `scarb test` 选项，但仅使用文件名。

然后，要运行我们所有的集成测试，我们可以只添加一个过滤器来仅运行路径包含 _integration_tests_ 的测试。

```shell
$ scarb test -f integration_tests
     Running test adder (snforge test)

```

我们看到在单元测试的第二部分中，有 1 个被过滤掉了，因为它不在 _integration_tests_ 文件中。

### 集成测试中的子模块

当你添加更多集成测试时，你可能希望在 _tests_ 目录中制作更多文件以帮助组织它们；例如，你可以按它们测试的功能对测试函数进行分组。如前所述，tests 目录中的每个文件都被编译为自己的单独 crate，这对于创建单独的范围以更紧密地模仿最终用户使用你的 crate 的方式很有用。然而，这意味着 tests 目录中的文件不像 _src_ 中的文件那样共享相同的行为，正如你在第 {{#chap packages-and-crates}} 章中关于如何将代码分离到模块和文件中所学到的那样。

当你有一组要在多个集成测试文件中使用的辅助函数，并尝试遵循第 {{#chap packages-and-crates}} 章的 [将模块分离到不同文件](ch07-05-separating-modules-into-different-files.md) 部分中的步骤将它们提取到一个通用模块时，测试目录文件的不同行为最为明显。例如，如果我们创建 _tests/common.cairo_ 并在其中放置一个名为 `setup` 的函数，我们可以向 `setup` 添加一些代码，我们希望从多个测试文件中的多个测试函数调用这些代码：

<span class="caption">文件名: tests/common.cairo</span>

```cairo, noplayground
pub fn setup() {
    println!("Setting up tests...");
}
```

<span class="caption">文件名: tests/integration_tests.cairo</span>

```cairo, noplayground
use adder::it_adds_two;

#[test]
fn internal() {
    assert!(it_adds_two(2, 2) == 4, "internal_adder failed");
}
```

<span class="caption">文件名: src/lib.cairo</span>

```cairo, noplayground
pub fn it_adds_two(a: u8, b: u8) -> u8 {
    a + b
}

#[cfg(test)]
mod tests {
    #[test]
    fn add() {
        assert_eq!(4, super::it_adds_two(2, 2));
    }
}
```

当我们使用 `scarb test` 运行测试时，我们将在测试输出中看到 _common.cairo_ 文件的一个新部分，即使此文件不包含任何测试函数，我们也未从任何地方调用 setup 函数：

```shell
$ scarb test 
     Running test adder (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(adder_unittest) adder v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_12_submodules/Scarb.toml)
   Compiling test(adder_integrationtest) adder_integrationtest v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_12_submodules/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from adder package
Running 1 test(s) from tests/
[PASS] adder_integrationtest::integration_tests::internal (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Running 1 test(s) from src/
[PASS] adder::tests::add (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

为了避免系统地为 _tests_ 文件夹的每个文件获取一个部分，我们还可以选择通过添加 `tests/lib.cairo` 文件使 `tests/` 目录像常规 crate 一样运行。在这种情况下，`tests` 目录将不再作为每个文件一个 crate 编译，而是作为整个目录的一个 crate 编译。

让我们创建这个 _tests/lib.cairo_ 文件：

<span class="caption">文件名: tests/lib.cairo</span>

```cairo, noplayground
mod common;
mod integration_tests;
```

项目目录现在看起来像这样：

```shell
adder
├── Scarb.lock
├── Scarb.toml
├── src
│   └── lib.cairo
└── tests
    ├── common.cairo
    ├── integration_tests.cairo
    └── lib.cairo
```

当我们再次运行 `scarb test` 命令时，输出如下：

```shell
$ scarb test 
     Running test adder (snforge test)
    Blocking waiting for file lock on registry db cache
    Blocking waiting for file lock on registry db cache
   Compiling test(adder_unittest) adder v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_13_single_integration_crate/Scarb.toml)
   Compiling test(adder_tests) adder_tests v0.1.0 (listings/ch10-testing-cairo-programs/no_listing_13_single_integration_crate/Scarb.toml)
    Finished `dev` profile target(s) in 1 second


Collected 2 test(s) from adder package
Running 1 test(s) from tests/
[PASS] adder_tests::integration_tests::internal (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Running 1 test(s) from src/
[PASS] adder::tests::add (l1_gas: ~0, l1_data_gas: ~0, l2_gas: ~40000)
Tests: 2 passed, 0 failed, 0 ignored, 0 filtered out


```

这样，只有测试函数会被测试，`setup` 函数可以被导入而无需被测试。

## 总结

Cairo 的测试功能提供了一种指定代码应如何运行的方法，以确保即使在你进行更改时它也能按预期继续工作。单元测试分别练习库的不同部分，并可以测试私有实现细节。集成测试检查库的许多部分是否正确地协同工作，并且它们使用库的公共 API 以与外部代码使用它的方式相同的方式测试代码。即使 Cairo 的类型系统和所有权规则有助于防止某些类型的错误，测试对于减少与代码预期行为方式有关的逻辑错误仍然很重要。

{{#quiz ../quizzes/ch10-02-testing-organization.toml}}
