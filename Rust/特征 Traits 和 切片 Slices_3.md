# 	Traits 和 Slices

Trait 特征就是 Rust 中的接口， Slice 切片在 go 中也见过了。

## 一、Traits 

对于之前的 Ticket 结构体：

```rust
pub struct Ticket {
    title: String,
    description: String,
    status: String,
}
```

我们可以通过这样的语句比较 title的值：

```rust
assert_eq!(ticket.title(), "A new title");
```

那如何比较两个 Ticket 实例呢？执行下面的语句。

```rust
let ticket1 = Ticket::new(/* ... */);
let ticket2 = Ticket::new(/* ... */);
ticket1 == ticket2
```

编译器报错：

```rust
error[E0369]: binary operation `==` cannot be applied to type `Ticket`
  --> src/main.rs:18:13
   |
18 |     ticket1 == ticket2
   |     ------- ^^ ------- Ticket
   |     |
   |     Ticket
   |
note: an implementation of `PartialEq` might be missing for `Ticket`
```

Ticket 是一个新类型，还没有对应的 == 操作应用。但是 Rust 编译器提示：an implementation of `PartialEq` might be missing for `Ticket`。这里的 `PartialEq`就是一个 trait。

### 1. 什么是 trait？

Traits 是 Rust 定义接口的方式。一个 trait 定义了一组方法，如果一个类型 type 要实现这个 trait，那么 type 就要实现这个 trait 中定义的所有方法。

#### 定义一个 trait

语法规则如下：

```rust
trait <TraitName> {
    fn <method_name>(<parameters>) -> <return_type>;
}

// 具体实现
trait MaybeZero {
    fn is_zero(self) -> bool;
}
```

#### 实现一个 trait

语法规则如下：

```rust
impl <TraitName> for <TypeName> {
    fn <method_name>(<parameters>) -> <return_type> {
        // Method body
    }
}

//具体实现
pub struct WrappingU32 {
    inner: u32,
}

impl MaybeZero for WrappingU32 {
    fn is_zero(self) -> bool {
        self.inner == 0
    }
}
```

#### 调用一个 trait 方法

和常规方法一样，通过 . 调用：

```rust
let x = WrappingU32 { inner: 5 };
assert!(!x.is_zero());
```

为了调用一个 trait 方法，必须满足以下两点要求：

- 该类型 type 必须实现了 trait
- trait 必须在作用域内

第二个条件通过 use 关键字实现。当然也可以不显式调用 use，在这种情况下：

- trait 已经在同一个 模块 module 下定义了
- trait 包含在标准库 prelude 中了。prelude 是一组 trait 和 types，在每一个 rust 项目中自动导入了。就好像 use std::prelude::*; 已经添加在每一个 Rust module 的最开始。

## 二、实现 traits

如果一个类型已经在其他的包 crate 中定义了，我们不能直接为这个类型定义一个新方法：

```rust
impl u32 {
    fn is_even(&self) -> bool {
        self % 2 == 0
    }
}

// 报错
error[E0390]: cannot define inherent `impl` for primitive types
  |
1 | impl u32 {
  | ^^^^^^^^
  |
  = help: consider using an extension trait instead

```

### 1. 扩展 trait

一个扩展 trait 就是一个主要目的为扩展来自别的 crate 包的类型的 trait，例如 u32。

```rust
// Bring the trait in scope
use my_library::IsEven;

fn main() {
    // Invoke its method on a type that implements it
    if 4.is_even() {
        // [...]
    }
}
```

### 2. 只能实现实现一次 trait

在一个 crate 包中，一个类型只能实现一次 trait。

### 3. 孤儿原则

对于 类型 type 和 特征 trait，必须满足以下两个条件中的一个：

- trait 定义在当前 crate 包中
- type 定义在当前 crate 包中

如果都不满足的的话，就会有下面的情况发生

- Crate `A` defines the `IsEven` trait
- Crate `B` implements `IsEven` for `u32`
- Crate `C` provides a (different) implementation of the `IsEven` trait for `u32`
- Crate `D` depends on both `B` and `C` and calls `1.is_even()`

那么调用  `1.is_even()`会造成混乱。

##  三、运算符重载

### 1. 运算符都是特征

在 Rust 中，所有的 operator 都是 trait。对于每一个运算符，都有对应的 trait 定义了该运算符的行为。比如，PartialEq trait 定义了 == 和 != 操作符的行为。

```rust
// The `PartialEq` trait definition, from Rust's standard library
// (It is *slightly* simplified, for now)
pub trait PartialEq {
    // Required method
    //
    // `Self` is a Rust keyword that stands for 
    // "the type that is implementing the trait"
    fn eq(&self, other: &Self) -> bool;

    // Provided method
    fn ne(&self, other: &Self) -> bool { ... }
}
```

下面是常见运算符的表：

| Operator                 | Trait                                                        |
| :----------------------- | :----------------------------------------------------------- |
| `+`                      | [`Add`](https://doc.rust-lang.org/std/ops/trait.Add.html)    |
| `-`                      | [`Sub`](https://doc.rust-lang.org/std/ops/trait.Sub.html)    |
| `*`                      | [`Mul`](https://doc.rust-lang.org/std/ops/trait.Mul.html)    |
| `/`                      | [`Div`](https://doc.rust-lang.org/std/ops/trait.Div.html)    |
| `%`                      | [`Rem`](https://doc.rust-lang.org/std/ops/trait.Rem.html)    |
| `==` and `!=`            | [`PartialEq`](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) |
| `<`, `>`, `<=`, and `>=` | [`PartialOrd`](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) |

### 2. 默认实现

如果 trait 中的方法有默认实现，我们可以在 类型实现时忽略它（这是选择性的，你也可以实现），例如：

```rust
pub trait PartialEq {
    fn eq(&self, other: &Self) -> bool;

    fn ne(&self, other: &Self) -> bool {
        !self.eq(other)
    }
}
// 这里的 ne 有默认实现，则在下面实现特征时，只需要实现 eq
struct WrappingU8 {
    inner: u8,
}

impl PartialEq for WrappingU8 {
    fn eq(&self, other: &WrappingU8) -> bool {
        self.inner == other.inner
    }
    
    // No `ne` implementation here
}
```

## 四、派生宏 Derive macros

### 1. 解构语法

可以通过如下的方式得到一个 struct 的成员：

```rust
impl PartialEq for Ticket {
    fn eq(&self, other: &Self) -> bool {
        let Ticket {
            title,
            description,
            status,
        } = self;
        let Ticket {
            title: other_title,
            description: other_description,
            status: other_status,
        } = other;
        // [...]
    }
}
```

### 2. 派生宏

派生宏是 Rust 的一个特殊风格，他在结构体顶部指明。

```rust
#[derive(PartialEq)]
struct Ticket {
    title: String,
    description: String,
    status: String
}
```

派生宏用于自动生成常见的 trait 的实现。在上面的示例中，PartialEq trait 将自动实现。

```rust
#[automatically_derived]
impl ::core::cmp::PartialEq for Ticket {
    #[inline]
    fn eq(&self, other: &Ticket) -> bool {
        self.title == other.title && self.description == other.description
            && self.status == other.status
    }
}
```

编译器会提醒的，不必担心。

 ## 五、特征界限和泛型编程

### 1. 泛型编程

泛型支持我们在写代码时支持以类型作为参数，而不是使用特定的类型的参数。

```rust
fn print_if_even<T>(n: T)
where
    T: IsEven + Debug
{
    if n.is_even() {
        println!("{n:?} is even");
    }
}
```

`print_if_even`是一个泛型函数。他不和特定的输入类型绑定，相反，它在任意的满足以下条件的类型 T 下都可以工作：

- 实现了 IsEven 特征
- 实现了 Debug 特征

这项原则由特征界限表达：T: IsEven + Debug。

### 2. 特征界限

如果移除上面例子的特征界限：

```rust
fn print_if_even<T>(n: T) {
    if n.is_even() {
        println!("{n:?} is even");
    }
}

// 报错
error[E0599]: no method named `is_even` found for type parameter `T` in the current scope
 --> src/lib.rs:2:10
  |
1 | fn print_if_even<T>(n: T) {
  |                  - method `is_even` not found for this type parameter
2 |     if n.is_even() {
  |          ^^^^^^^ method not found in `T`

error[E0277]: `T` doesn't implement `Debug`
 --> src/lib.rs:3:19
  |
3 |         println!("{n:?} is even");
  |                   ^^^^^ `T` cannot be formatted using `{:?}` because it doesn't implement `Debug`
  |
help: consider restricting type parameter `T`
  |
1 | fn print_if_even<T: std::fmt::Debug>(n: T) {
  |                   +++++++++++++++++
```

没有特征界限，编译器不知道 T 能进行什么工作。不知道 T 是否有 is_even 方法，在编译器角度 T 根本没有任何可以进行的行为。特质界限限制了类型集，这些类型集中的类型确保实现了 function 中接下来所需的方法。

#### 内联特征界限 语法：

```rust
fn print_if_even<T: IsEven + Debug>(n: T) {
    //           ^^^^^^^^^^^^^^^^^
    //           This is an inline trait bound
    // [...]
}

// 可以为泛型指定特殊的含义的名字
fn print_if_even<Number: IsEven + Debug>(n: Number) {
    // [...]
}
```

## 六、字符串切片 String slice

字符串字面值就是一个字符串切片 string slice。

```rust
let s = "Hello, world!";
```

s 的类型就是一个 `&str`。

### 1. 内存

执行如下代码：

```rust
let mut s = String::with_capacity(5);
s.push_str("Hello");
// Create a string slice reference from the `String`, skipping the first byte.
let slice: &str = &s[1..];
```

这是内存模型：

```rust
                    s                              slice
      +---------+--------+----------+      +---------+--------+
Stack | pointer | length | capacity |      | pointer | length |
      |    |    |   5    |    5     |      |    |    |   4    |
      +----|----+--------+----------+      +----|----+--------+
           |        s                           |  
           |                                    |
           v                                    | 
         +---+---+---+---+---+                  |
Heap:    | H | e | l | l | o |                  |
         +---+---+---+---+---+                  |
               ^                                |
               |                                |
               +--------------------------------+

```

slice 会保存两个信息在栈中：

- 一个指向切片首地址的指针
- 切片的长度

切片并不持有数据，他只是指向数据。只关心指向的这部分数据。

### 2. `&str`和`&string`

当需要引用 String 数据，使用＆str而不是＆string。

- 如果一个方法返回 &string，那么说明堆中有一块内存 UTF8 编码的数据完全匹配所返回的引用。
- 如果一个方法返回 &str，那么说明只是在堆内存中有一个块内存的部分正好和返回的引用匹配，更为灵活。

## 七、解引用特征 Deref trait

尝试阅读下面这行代码：

```rust
impl Ticket {
    pub fn title(&self) -> &str {
        &self.title
    }
}
```

title 的类型为 String，这里应该是返回值为 &String，为什么返回 &str 没有编译错误？就是因为 Deref trait。在 Rust 的标准库中，deref 在 std::ops 模块：

```rust
// I've slightly simplified the definition for now.
// We'll see the full definition later on.
pub trait Deref {
    type Target;
    
    fn deref(&self) -> &Self::Target;
}
```

而 target 就是一个关联的类型。它是一种具体类型的占位符，在实现特征时必须指定。

### 1. 强制解引用

通过对一个类型 T 实现 `Deref<Target = U>`特征，这就会告诉编译器 &T 和 &U 在某种程度上是可以互换的。特别的，你可以得到下面两种行为：

- 对 T 的引用隐式地被转化为对 U 的引用（例如：&T 变成 &U）
- 你可以通过 &T 调用 U 中所有定义的方法，以 &self 作为输入。

通过 Deref 还可以做 *（解引用运算的重载）。

### 2. String 实现了 Deref

之所以可以编译通过就是这个原因：

```rust
impl Deref for String {
    type Target = str;
    
    fn deref(&self) -> &str {
        // [...]
    }
}
```

## 八、Sized

在我们之前讨论 &str 的内存是我们知道，&str 不仅仅只保存了指针，同时还保存了其他的数据

### 1. 动态大小的类型

str 是一个动态大小的类型（DST）。DST 就是在编译时不知道大小的类型。无论什么时候，我们持有一个DST 的引用，比如 &str。它必须还包含一些除了它指向的地址的额外的数据。他就是一个胖指针（fat pointer）。像 &str，就是保存了它指向的切片的长度。

### 2. sized trait

Rust 标准库中定义了一个 trait 叫做 Sized。

```rust
pub trait Sized {
    // This is an empty trait, no methods to implement.
}
```

如果一个类型是 Sized，那么它的大小在编译的时候就知道了，不是一个 DST。

### 3. marker trait 标记特征

Sized 就是一个标记特诊。标记特征就是一个没有任何方法需要实现的特征。它里面没有定义任何方法，他只是为了 type 的某些特性做了一个标记。标记只会被编译器发挥出作用，使得其发挥某个功能或者优化。

### 4. auto trait 自动特征

特别的，Sized 也是一个自动特征。我们不需要显式地实现它，编译器会自动根据类型定义实现。

## 九、泛型和关联类型

让我们重新审视一下 From 和 Deref 两个 trait：

```rust
pub trait From<T> {
    fn from(value: T) -> Self;
}

pub trait Deref {
    type Target;
    
    fn deref(&self) -> &Self::Target;
}
```

他们都将类型视作参数：

- 在 From 中，是泛型参数 T
- 在 Deref 中，是关联类型 Target

这两者的区别是什么？

- Deref 只能实现一次，也就是说 String 只能被解引用成 str，而不能是别的类型，不然编译器就会混乱。所以一个关联的类型，在实现之后，这个方法就是固定的，这个类型也就确定下来了。

- From 可以实现多次，每一次实现都会传递一个不同的类型参数，因为在调用 From 时我们也会知道参数的类型，自然编译器也会知道去调用哪一个方法。所以 From 中不同的 from 实现，可以视为一种多态，他们的函数签名是不一样的。

    ```rust
    impl From<u32> for WrappingU32 {
        fn from(value: u32) -> Self {
            WrappingU32 { inner: value }
        }
    }
    
    impl From<u16> for WrappingU32 {
        fn from(value: u16) -> Self {
            WrappingU32 { inner: value.into() }
        }
    }
    ```

### 1. Add 运算符重载

标准库中定义：

```rust
pub trait Add<RHS = Self> {
    type Output;
    
    fn add(self, rhs: RHS) -> Self::Output;
}
```

- RHS 作为泛型参数，默认是 Self（实现者本身的类型）
- Output 作为关联类型，返回 + 的结果

为了实现不同类型的相加：

```rust
impl Add<u32> for u32 {
    type Output = u32;
    
    fn add(self, rhs: u32) -> u32 {
      //                      ^^^
      // This could be written as `Self::Output` instead.
      // The compiler doesn't care, as long as the type you
      // specify here matches the type you assigned to `Output` 
      // right above.
      // [...]
    }
}

impl Add<&u32> for u32 {
    type Output = u32;
    
    fn add(self, rhs: &u32) -> u32 {
        // [...]
    }
}
```

如果把 output 也设置成泛型，那好了对于 u32 + u32 那么 Output 就有两种情况了，根本就不知道调用哪个方法。