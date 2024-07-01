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