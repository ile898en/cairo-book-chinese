# Segment Arena 内置函数

_Segment Arena_ 内置函数通过跟踪段端点来扩展 Cairo VM 的内存处理。这种方法简化了需要分配和最终确定段的内存操作。

## 单元格组织

每个 Segment Arena 内置函数实例使用 3 个单元格的块工作，这些块维护字典的状态：

- 第一个单元格：包含信息指针的基地址
- 第二个单元格：包含当前分配的段数
- 第三个单元格：包含当前已压缩/最终确定的段数

此结构与信息段密切配合，信息段也按 3 个单元格的块组织：

- 第一个单元格：段的基地址
- 第二个单元格：段的结束地址（压缩时）
- 第三个单元格：当前已压缩段的数量（压缩索引）

<div align="center">
  <img src="segment-arena.png" alt="segment arena builtin segment"/>
</div>
<div align="center">
  <span class="caption">Segment Arena 内置函数段</span>
</div>

让我们看两个 Segment Arena 段的快照，这是在 Cairo VM 执行虚拟程序期间。

在第一个快照中，让我们看看分配字典时的第一种情况：

- `info_ptr` 指向新的信息段
- `n_dicts` 增加到 1
- 创建具有三个单元格的信息段
- 字典获取新段 `<3:0>`

现在，在第二种情况下，又分配了一个字典：

- 每个字典的信息段增加三个单元格
- 已压缩字典设置了结束地址
- 按顺序分配压缩索引
- 未完成的字典具有 `0` 结束地址

<div align="center">
  <img src="segment-arena-valid.png" alt="valid segment arena builtin segment"/>
</div>
<div align="center">
  <span class="caption">快照 1 - 有效的 Segment Arena 内置函数段</span>
</div>

第二个快照显示了两种错误情况。在第一种情况下，当 `info_ptr` 包含 _非可重定位_ 值 `ABC` 时，会出现无效状态。访问信息段时会触发错误。在第二种情况下，当出现快照所示的不一致状态（`n_squashed` 大于 `n_segments`）时，会出现错误。

<div align="center">
  <img src="segment-arena-error.png" alt="invalid segment arena builtin segment"/>
</div>
<div align="center">
  <span class="caption">快照 2 - 无效的 Segment Arena 内置函数段</span>
</div>

### 关键验证规则

内置函数强制执行多项规则：

- 每个段必须恰好分配和最终确定一次
- 所有单元格值必须是有效的 field 元素
- 段大小必须是非负的
- 压缩操作必须保持顺序
- 信息段条目必须对应于段分配

## 实现参考

这些 Segment Arena 内置函数的实现参考可能并不详尽。

- [TypeScript Segment Arena Builtin](https://github.com/kkrt-labs/cairo-vm-ts/blob/main/src/builtins/segmentArena.ts)
- [Rust Segment Arena Builtin](https://github.com/lambdaclass/cairo-vm/blob/41476335884bf600b62995f0c005be7d384eaec5/vm/src/vm/runners/builtin_runner/segment_arena.rs)
- [Zig Segment Arena Builtin](https://github.com/keep-starknet-strange/ziggy-starkdust/blob/55d83e61968336f6be93486d7acf8530ba868d7e/src/vm/builtins/builtin_runner/segment_arena.zig)
