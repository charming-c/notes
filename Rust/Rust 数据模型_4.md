# Rust 数据模型

## 一、枚举类型 enum

枚举是一种可以具有固定值的类型，称为 variant。在Rust中，使用 enum 关键字来定义枚举：

```rust
enum Status {
    ToDo,
    InProgress,
    Done,
}
```

### 1. match 关键字

下面是一个使用 match 的例子：

```rust
enum Status {
    ToDo,
    InProgress,
    Done
}

impl Status {
    fn is_done(&self) -> bool {
        match self {
            Status::Done => true,
            // The `|` operator lets you match multiple patterns.
            // It reads as "either `Status::ToDo` or `Status::InProgress`".
            Status::InProgress | Status::ToDo => false
        }
    }
}
```

一个 match 语句使得我们可以用一个 Rust 值匹配一系列 pattern。和 if 表达式有点像。

#### match 是全面的 exhaustiveness 

必须使用 match 处理 enum 中的所有变体，如果忘记处理了一些 variant，则会有编译错误，例如：

```rust
match self {
    Status::Done => true,
    Status::InProgress => false,
}

// 报错
error[E0004]: non-exhaustive patterns: `ToDo` not covered
 --> src/main.rs:5:9
  |
5 |     match status {
  |     ^^^^^^^^^^^^ pattern `ToDo` not covered
```

#### catch-all 语法

如果对于一些变体不关心，统一处理，可以采用 _ 模式：

```rust
match status {
    Status::Done => true,
    _ => false
}
```

_ 模式会匹配所有前面没有匹配的所有变体

### 2. 变体 variant 

在 Rust 中，variant (enum 中的元素) 本身可以包含数据。例如：

```rust
enum Status {
    ToDo,
    InProgress {
        assigned_to: String,
    },
    Done,
}
```

 不可以通过 enum 直接访问元素中包含的数据：

```rust
let status: Status = /* */;

// This won't compile
println!("Assigned to: {}", status.assigned_to);


error[E0609]: no field `assigned_to` on type `Status`
 --> src/main.rs:5:40
  |
5 |     println!("Assigned to: {}", status.assigned_to);
  |                                        ^^^^^^^^^^^ unknown field
```

因为 assigned_to 是变体特有的，不是所有的 Status 实例都可用，为了访问 assigned_to，需要使用模式匹配 match：

```rust
match status {
    Status::InProgress { assigned_to } => {
        println!("Assigned to: {}", assigned_to);
    },
    Status::ToDo | Status::Done => {
        println!("Done");
    }
}
// 这里的 assigned_to 是一个绑定，这里会解构 Status::InProgress 实例，并且将
// 其中的属性 绑定到 assgned_to 中
// 这是另一种绑定方法

match status {
    Status::InProgress { assigned_to: person } => {
        println!("Assigned to: {}", person);
    },
    Status::ToDo | Status::Done => {
        println!("Done");
    }
}
```

### 3. if let 语句

在 Rust 中可以通过 if let 匹配 enum 中的一个 variant，而不需要去处理所有的 variant:

```rust
impl Ticket {
    pub fn assigned_to(&self) -> &str {
        if let Status::InProgress { assigned_to } = &self.status {
            assigned_to
        } else {
            panic!("Only `In-Progress` tickets can be assigned to someone");
        }
    }
}
```

**let else 类似**：

```rust
impl Ticket {
    pub fn assigned_to(&self) -> &str {
        let Status::InProgress { assigned_to } = &self.status else {
            panic!("Only `In-Progress` tickets can be assigned to someone");
        };
        assigned_to
    }
}
```

## 二、元组 

元祖是 Rust 的基本数据类型之一，是一组（通常是不同类型的）值的集合。

```rust
// Two values, same type
let first: (i32, i32) = (3, 4);
// Three values, different types
let second: (i32, u32, u8) = (-42, 3, 8);

assert_eq!(second.0, -42);
assert_eq!(second.1, 3);
assert_eq!(second.2, 8);
```

## 三、Option 类型

Option 是一个 Rust 类型，可以代表空值。它是一个 枚举类，定义在 Rust 标准库之中：

```rust
enum Option<T> {
    Some(T),
    None,
}
// 其中 Some 是一个像 元组 一样的 variant
// Some 持有了一个没有命名的属性
```

Option 实现了一个 value 可能存在（Some(T)）或者不存在（None）想法。它迫使我们必须要显式地处理这两种情况。如果忘记处理 None，就会抛出一个编译错误。

## 四、Result 类型

和 Option 类似：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

它有两个变体：

- OK(T)：代表一个成功的操作，它持有一个 T。是操作的输出
- Err(E)：代表一个失败的操作，它持有一个 E，是出现的错误

Rust 通过 Result，迫使你必须编程失败的可能，在方法的签名中。如果一个函数会导致错误，并且你希望调用者可以得到该错误，那么函数就应该返回一个 Result：

```rust
// Just by looking at the signature, you know that this function can fail.
// You can also inspect `ParseIntError` to see what kind of failures to expect.
fn parse_int(s: &str) -> Result<i32, ParseIntError> {
    // ...
}
```

**Rust 中 Result 类型的处理**：

- 当出错时 panic：

    ```rust
    // Panics if `parse_int` returns an `Err`.
    let number = parse_int("42").unwrap();
    // `expect` lets you specify a custom panic message.
    let number = parse_int("42").expect("Failed to parse integer");
    ```

- 其他自定义处理方式：

    ```rust
    match parse_int("42") {
        Ok(number) => println!("Parsed number: {}", number),
        Err(err) => eprintln!("Error: {}", err),
    }
    ```