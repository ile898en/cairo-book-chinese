# 合约事件

事件是智能合约通知外部世界再其执行过程中发生了任何变化的一种方式。它们在智能合约集成到现实世界应用程序中起着关键作用。

从技术上讲，事件是智能合约在执行期间发出的自定义数据结构，并存储在相应的交易收据中，允许任何外部工具解析和索引它（最常见的是 [Starknet SDK](https://docs.starknet.io/tools/overview/)，如 [Starknet.js](https://starknetjs.com/docs/guides/contracts/events/)）。

## 定义事件

智能合约的事件定义在一个用属性 `#[event]` 注释的枚举中。此枚举必须命名为 `Event`。

```cairo,noplayground
    //ANCHOR: full_events
    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        BookAdded: BookAdded,
        #[flat]
        FieldUpdated: FieldUpdated,
        BookRemoved: BookRemoved,
    }
```

每个变体，如 `BookAdded` 或 `FieldUpdated` 代表一个可以由合约发出的事件。变体数据表示与事件关联的数据。它可以是任何实现了 `starknet::Event` trait 的 `struct` 或 `enum`。这可以通过在你的类型定义之上简单地添加 `#[derive(starknet::Event)]` 属性来实现。

每个事件数据字段都可以用属性 `#[key]` 注释。键字段与数据字段分开存储，以供外部工具使用，以便轻松地根据这些键过滤事件。

让我们看看这个示例的完整事件定义，用于添加、更新和删除书籍：

```cairo,noplayground
    #[event]
    #[derive(Drop, starknet::Event)]
    pub enum Event {
        BookAdded: BookAdded,
        #[flat]
        FieldUpdated: FieldUpdated,
        BookRemoved: BookRemoved,
    }
    //ANCHOR_END: event

    #[derive(Drop, starknet::Event)]
    pub struct BookAdded {
        pub id: u32,
        pub title: felt252,
        #[key]
        pub author: felt252,
    }

    #[derive(Drop, starknet::Event)]
    pub enum FieldUpdated {
        Title: UpdatedTitleData,
        Author: UpdatedAuthorData,
    }

    #[derive(Drop, starknet::Event)]
    pub struct UpdatedTitleData {
        #[key]
        pub id: u32,
        pub new_title: felt252,
    }

    #[derive(Drop, starknet::Event)]
    pub struct UpdatedAuthorData {
        #[key]
        pub id: u32,
        pub new_author: felt252,
    }

    #[derive(Drop, starknet::Event)]
    pub struct BookRemoved {
        pub id: u32,
    }
```

在这个例子中：

- 有 3 个事件：`BookAdded`、`FieldUpdated` 和 `BookRemoved`，
- `BookAdded` 和 `BookRemoved` 事件使用简单的 `struct` 来存储它们的数据，而 `FieldUpdated` 事件使用结构体的 `enum`，
- 在 `BookAdded` 事件中，`author` 字段是一个键字段，将在智能合约外部用于按 `author` 过滤 `BookAdded` 事件，而 `id` 和 `title` 是数据字段。

> **变体** 及其关联的数据结构可以命名不同，尽管通常的做法是使用相同的名称。**变体名称** 在内部用作 **第一个事件键** 来表示事件的名称并帮助过滤事件，而 **变体数据名称** 在智能合约中用于在发出事件之前 **构建事件**。

### #[flat] 属性

有时你可能有一个复杂的事件结构，其中嵌套了一些枚举，例如前一个示例中的 `FieldUpdated` 事件。在这种情况下，你可以使用 `#[flat]` 属性展平此结构，这意味着使用内部变体名称作为事件名称，而不是注释枚举的变体名称。在前面的示例中，因为 `FieldUpdated` 变体用 `#[flat]` 注释，当你发出 `FieldUpdated::Title` 事件时，它的名称将是 `Title` 而不是 `FieldUpdated`。如果你的嵌套枚举超过 2 层，你可以在多个级别上使用 `#[flat]` 属性。

## 发出事件

定义了事件列表后，你想在智能合约中发出它们。这可以通过调用 `self.emit()` 并传入事件数据结构作为参数来简单实现。

```cairo,noplayground
        fn add_book(ref self: ContractState, id: u32, title: felt252, author: felt252) {
            // ... logic to add a book in the contract storage ...
            self.emit(BookAdded { id, title, author });
        }

        fn change_book_title(ref self: ContractState, id: u32, new_title: felt252) {
            self.emit(FieldUpdated::Title(UpdatedTitleData { id, new_title }));
        }

        fn change_book_author(ref self: ContractState, id: u32, new_author: felt252) {
            self.emit(FieldUpdated::Author(UpdatedAuthorData { id, new_author }));
        }

        fn remove_book(ref self: ContractState, id: u32) {
            self.emit(BookRemoved { id });
        }
```

为了更好地理解幕后发生的事情，让我们看两个发出的事件以及它们如何存储在交易收据中的示例：

### 示例 1: 添加一本书

在这个例子中，我们发送一个交易，调用 `add_book` 函数，参数为 `id` = 42, `title` = 'Misery' 和 `author` = 'S. King'。

如果你阅读交易收据的 "events" 部分，你会得到类似以下内容：

```json
"events": [
    {
      "from_address": "0x27d07155a12554d4fd785d0b6d80c03e433313df03bb57939ec8fb0652dbe79",
      "keys": [
        "0x2d00090ebd741d3a4883f2218bd731a3aaa913083e84fcf363af3db06f235bc",
        "0x532e204b696e67"
      ],
      "data": [
        "0x2a",
        "0x4d6973657279"
      ]
    }
  ]
```

在此收据中：

- `from_address` 是你的智能合约的地址，
- `keys` 包含发出的 `BookAdded` 事件的键字段，序列化为 `felt252` 数组。
  - 第一个键 `0x2d00090ebd741d3a4883f2218bd731a3aaa913083e84fcf363af3db06f235bc` 是事件名称的选择器，即 `Event` 枚举中的变体名称，所以是 `selector!("BookAdded")`，
  - 第二个键 `0x532e204b696e67 = 'S. King'` 是事件的 `author` 字段，因为它已使用 `#[key]` 属性定义，
- `data` 包含发出的 `BookAdded` 事件的数据字段，序列化为 `felt252` 数组。第一项 `0x2a = 42` 是 `id` 数据字段，`0x4d6973657279 = 'Misery'` 是 `title` 数据字段。

### 示例 2: 更新书籍作者

现在我们想更改书籍的作者姓名，所以我们发送一个交易，调用 `change_book_author`，参数为 `id` = `42` 和 `new_author` = 'Stephen King'。

此 `change_book_author` 调用发出一个 `FieldUpdated` 事件，事件数据为 `FieldUpdated::Author(UpdatedAuthorData { id: 42, title: author: 'Stephen King' })`。如果你阅读交易收据的 "events" 部分，你会得到类似以下内容：

```json
"events": [
    {
      "from_address": "0x27d07155a12554d4fd785d0b6d80c03e433313df03bb57939ec8fb0652dbe79",
      "keys": [
        "0x1b90a4a3fc9e1658a4afcd28ad839182217a69668000c6104560d6db882b0e1",
        "0x2a"
      ],
      "data": [
        "0x5374657068656e204b696e67"
      ]
    }
  ]
```

由于 `Event` 枚举中的 `FieldUpdated` 变体已用 `#[flat]` 属性注释，因此使用内部变体 `Author` 作为事件名称，而不是 `FieldUpdated`。所以：

- 第一个键是 `selector!("Author")`，
- 第二个键是 `id` 字段，用 `#[key]` 注释，
- 数据字段是 `0x5374657068656e204b696e67 = 'Stephen King'`。
