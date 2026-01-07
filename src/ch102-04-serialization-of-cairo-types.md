# Cairo 类型的序列化

字段元素 (`felt252`) 包含 252 位，是 Cairo 虚拟机中唯一的实际类型，因此 [所有适合 252 位的数据类型](#data-types-using-at-most-252-bits) 都由单个 felt 表示，而 [所有大于 252 位的数据类型](#data-types-using-more-than-252-bits) 都由 felt 列表表示。因此，为了与合约交互，你必须知道如何 [将任何大于 252 位的参数序列化为 felt 列表](#serialization-of-data-types-using-more-than-252-bits) ，以便你可以正确地制定交易中的调用数据 (calldata)。

> 注意：对于任何不开发 Starknet 库或 SDK 的开发人员，强烈建议使用现有的 [Starknet SDKs](https://docs.starknet.io/tools/overview/) 之一或使用 [Starknet Foundry 的 `sncast` 的 `--argument` 标志](https://foundry-rs.github.io/starknet-foundry/starknet/calldata-transformation.html#using---arguments) 来极大地简化序列化过程。从 `sncast` v0.43.0 开始，`sncast call` 的结果也被解析为可读信息，如下例所示：
>
> ```
> sncast call \
>   --contract-address=0x00e270c8396d333f88556edf143ac751240f050d907e5190525accbe275f2348 \
>   --function=get_order \
>   --network=sepolia
> response: Order {
>   position_id: PositionId { value: 1_u32 },
>   base_asset_id: AssetId { value: 0x3 },
>   base_amount: -10_i64,
>   quote_asset_id: AssetId { value: 0x2d },
>   fee_asset_id: AssetId { value: 0x4d2 },
>   fee_amount: 0_u64,
>   expiration: Timestamp { seconds: 12341432_u64 },
>   salt: 0x0,
>   name: "This is my order"
> }
> ```

## 使用最多 252 位的数据类型

以下 Cairo 数据类型使用最多 252 位：

- `ContractAddress`
- `EthAddress`
- `StorageAddress`
- `ClassHash`
- 使用最多 252 位的无符号整数：`u8`、`u16`、`u32`、`u64`、`u128` 和 `usize`
- `bytes31`
- `felt252`
- 有符号整数：`i8`、`i16`、`i32`、`i64` 和 `i128`
  > 注意：负值 \\( -x \\) 序列化为 \\( P-x \\)，其中 \\( P = 2^{251} + 17\*2^{192} + 1 \\)

对于这些类型，每个值都序列化为包含一个 `felt252` 值的单成员列表。

## 使用超过 252 位的数据类型

以下 Cairo 数据类型使用超过 252 位，因此具有非平凡的序列化：

- 大于 252 位的无符号整数：`u256` 和 `u512`
- 数组和 Span
- 枚举
- 结构体和元组
- 字节数组（表示字符串）

## 使用超过 252 位的数据类型的序列化

### `u256` 的序列化

Cairo 中的 `u256` 值由两个 `felt252` 值表示，如下所示：

- 第一个 `felt252` 值包含 128 个最低有效位，通常称为原始 `u256` 值的低部分。
- 第二个 `felt252` 值包含 128 个最高有效位，通常称为原始 `u256` 值的高部分。

例如：

- 十进制值为 2 的 `u256` 变量序列化为 `[2,0]`。要理解原因，请检查 2 的二进制表示并将其分成两个 128 位部分，如下所示：
  - 128 个高位：\\(0 \cdots 0 \\)
  - 128 个低位：\\( 0 \cdots 10 \\)
- 十进制值为 \\( 2^{128} \\) 的 `u256` 变量序列化为 `[0,1]`。要理解原因，请检查 \\( 2^{128} \\) 的二进制表示并将其分成两个 128 位部分，如下所示：
  - 128 个高位：\\( 0 \cdots 01 \\)
  - 128 个低位：\\(0 \cdots 0 \\)

- 十进制值为 \\( 2^{129}+2^{128}+20 \\) 的 `u256` 变量序列化为 `[20,3]`。要理解原因，请检查 \\( 2^{129}+2^{128}+20 \\) 的二进制表示并将其分成两个 128 位部分，如下所示：
  - 128 个高位：\\( 0 \cdots 011 \\)
  - 128 个低位：\\( 0 \cdots 10100 \\)

### `u512` 的序列化

Cairo 中的 `u512` 类型是一个包含四个 `felt252` 成员的结构体，每个成员代表原始整数的一个 128 位肢体 (limb)，类似于 `u256` 类型。

### 数组和 Span 的序列化

数组和 Span 序列化如下：

`<array/span_length>, <first_serialized_member>,..., <last_serialized_member>`

例如，考虑以下 `u256` 值数组：

```cairo, noplayground
let POW_2_128: u256 = 0x100000000000000000000000000000000
let array: Array<u256> = array![10, 20, POW_2_128]
```

数组中的每个 `u256` 值由两个 `felt252` 值表示。所以上面的数组序列化如下：

1. `3`: 数组乘员数
2. `10,0`: 序列化的第一个成员
3. `20,0`: 序列化的第二个成员
4. `0,1`: 序列化的第三个成员

结合以上内容，数组被序列化为 `[3,10,0,20,0,0,1]`。

### 枚举的序列化

枚举序列化如下：

`<index_of_enum_variant>,<serialized_variant>`

注意，枚举变体索引是基于 0 的，不要与其基于 1 的存储布局混淆，后者是为了区分第一个变体和未初始化的存储槽。

**枚举序列化示例 #1**

考虑以下名为 `Week` 的枚举定义：

```cairo,noplayground
enum Week {
    Sunday: (), // Index=0. 变体类型是单元类型 (0-tuple)。
    Monday: u256, // Index=1. 变体类型是 u256。
}
```

现在考虑如下表所示的 `Week` 枚举变体的实例化：

| 实例              | 索引  | 类型   | 序列化        |
| ----------------- | ----- | ------ | ------------- |
| `Week::Sunday`    | `0`   | unit   | `[0]`         |
| `Week::Monday(5)` | `1`   | `u256` | `[1,5,0]`     |

**枚举序列化示例 #2**

考虑以下名为 `MessageType` 的枚举定义：

```cairo,noplayground
enum MessageType {
    A,
    #[default]
    B: u128,
    C
}
```

现在考虑如下表所示的 `MessageType` 枚举变体的实例化：

| 实例                | 索引  | 类型   | 序列化    |
| ------------------- | ----- | ------ | --------- |
| `MessageType::A`    | `1`   | unit   | `[0]`     |
| `MessageType::B(6)` | `0`   | `u128` | `[1,6]`   |
| `MessageType::C`    | `2`   | unit   | `[2]`     |

如你所见，`#[default]` 属性不影响序列化。它只影响 `MessageType` 的存储布局，其中默认变体 `B` 将存储为 `0`。

### 结构体和元组的序列化

结构体和元组通过一次序列化其一个成员来序列化。

结构体的成员按其定义中出现的顺序序列化。

例如，考虑以下结构体 `MyStruct` 的定义：

```cairo,noplayground
struct MyStruct {
    a: u256,
    b: felt252,
    c: Array<felt252>
}
```

对于以下两个结构体成员的实例化，序列化是相同的：

```cairo,noplayground
let my_struct = MyStruct {
    a: 2, b: 5, c: [1,2,3]
};

let my_struct = MyStruct {
    b: 5, c: [1,2,3], a: 2
};
```

`MyStruct` 的序列化如下表所示：

| 成员         | 类型                 | 序列化      |                                           |
| ------------ | -------------------- | ----------- | ----------------------------------------- |
| `a: 2`       | `u256`               | `[2,0]`     | 有关序列化 `u256` 值的更多信息请参见上文  |
| `b: 5`       | `felt252`            | `5`         |
| `c: [1,2,3]` | 大小为 3 的 `felt252` 数组 | `[3,1,2,3]` |

结合以上内容，结构体被序列化为 `[2,0,5,3,1,2,3]`。

### 字节数组的序列化

字符串在 Cairo 中表示为 `ByteArray` 类型。字节数组实际上是一个具有以下成员的结构体：

- `data: Array<felt252>`: 包含字节数组的 31 字节块。每个 `felt252` 值正好有 31 个字节。如果字节数组中的字节数少于 31，则此数组为空。
- `pending_word: felt252`: 用完整的 31 字节块填充 `data` 数组后剩余的字节。挂起字最多包含 30 个字节。
- `pending_word_len: usize`: `pending_word` 中的字节数。

**示例 #1: 短于 31 个字符的字符串**

考虑字符串 `hello`，其 ASCII 编码为 5 字节十六进制值 `0x68656c6c6f`。生成的字节数组序列化如下：

```cairo,noplayground
0, // data 数组中 31 字节字的梳理。
0x68656c6c6f, // 挂起字 (Pending word)
5 // 挂起字的长度，以字节为单位
```

**示例 2: 长于 31 个字节的字符串**

考虑字符串 `Long string, more than 31 characters.`，它由以下十六进制值表示：

- `0x4c6f6e6720737472696e672c206d6f7265207468616e203331206368617261` (31 字节字)
- `0x63746572732e` (6 字节挂起字)

生成的字节数组序列化如下：

```cairo,noplayground
1, // 数组构造中 31 字节字的数量。
0x4c6f6e6720737472696e672c206d6f7265207468616e203331206368617261, // 31 字节字。
0x63746572732e, // 挂起字
6 // 挂起字的长度，以字节为单位
```
