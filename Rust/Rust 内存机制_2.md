# Rust 内存机制

从内存的角度，解析 Rust 的 所有权 Ownership 和 引用 Reference。

## 一、栈 stack

栈是 LIFO（后进先出）的数据结构。在 CSAPP 中的内存模型中已经知道，当调用一个函数时，一个新的栈帧（stack frame）就会被添加到栈顶。栈帧中保存了函数的所有参数、局部变量和一些其他的值。当函数返回时，栈帧就会从栈顶弹出。

```rust
                                 +-----------------+
                       func2     | frame for func2 |   func2
+-----------------+  is called   +-----------------+  returns   +-----------------+
| frame for func1 | -----------> | frame for func1 | ---------> | frame for func1 |
+-----------------+              +-----------------+            +-----------------+

```

从操作的角度来看，栈内存的分配和销毁是十分快速的。我们总是 push 和 pop 数据到栈顶中。所以我们不需要去查找空闲内存，同时我们也不需要担心内促碎片，<u>栈本身就是一个连续的内存块。</u>

Rust 经常需要在 stack 中分配数据。如果函数有一个 `u32`的输入参数，这 32 位数据分配在栈上。如果定义了一个局部变量 `i64`，那么 64 位的数据也会分配在栈上。这些都会运作的很好在编译时，因为编译器在编译时知道这些 integer 的大小。因此，编译程序知道在栈上分配多少空间去保存它们。

### 1. std::mem::size_of

我们可以通过使用 std::mem::size_of 函数确认一个类型将会在占中占用多少空间。对于 `u8`，如下：

```rust
// We'll explain this funny-looking syntax (`::<u8>`) later on.
// Ignore it for now.
assert_eq!(std::mem::size_of::<u8>(), 1);
```

因为 `u8`是 8 位长的，占 1 个字节。

## 二、堆 heap

stack 设计很棒，但是不能解决我们所有的问题。如果我们在编译时不知道数据的大小呢？集合、String和其他动态大小的数据是不能（完全）被栈分配内存的，这就引入了 heap 堆。

可以把堆想象成一个巨大的内存片--数组。无论何时需要将数据保存在堆中，我们请求一个特殊的程序 allocator，去维护一个堆的小块（子集），这就是堆内存的分配。如果分配成功，allocator 会给你一个指向这个内存块首地址的指针 pointer。

### 1. 没有自动内存销毁

堆内存的组织和栈非常不一样。堆内存的分配不是连续的，他们可以分配到堆内存中的任何位置。

```rust
+---+---+---+---+---+---+-...-+-...-+---+---+---+---+---+---+---+
|  Allocation 1 | Free  | ... | ... |  Allocation N |    Free   |
+---+---+---+---+---+---+ ... + ... +---+---+---+---+---+---+---+
```

追踪堆中的每一个部分是 使用 还是 空闲 是 allocator 的工作。allocator 不会自动释放你请求分配的内存。需要可以使用分配器进行内存的释放。

### 2. 性能 Performance

堆的伸缩性产生了对应的代价：堆的 allocator 比栈的内存分配效率要低。在性能调优领域，人们总是建议无论何时尽可能的使用栈的内存分配。

### 3. Rust String 内存展示

无论何时创建一个类型为 String 的局部变量，Rust就会强制在堆中分配内存：由于不能提前知道需要把多少 text 添加到 String 中，所以不能在栈中维护一块特定大小的空间。但是，一个 String 也不全是堆分配的，它也会保留一些数据在栈上，如：

- 一个指向堆内存的指针
- String 的长度（例如：String 的 字节数）
- String 的容量（例如：String 在 heap 中的整个内存字节数）

让我们看一个例子：

```rust
let mut s = String::with_capacity(5);
```

如果运行这行代码，内存会如下：

```rust
      +---------+--------+----------+
Stack | pointer | length | capacity | 
      |  |      |   0    |    5     |
      +--|------+--------+----------+
         |
         |
         v
       +---+---+---+---+---+
Heap:  | ? | ? | ? | ? | ? |
       +---+---+---+---+---+
```

如果往 String 中增加一些 text

```rust
s.push_str("Hey");
```

情况会改变成：

```rust
      +---------+--------+----------+
Stack | pointer | length | capacity |
      |  |      |   3    |    5     |
      +--|  ----+--------+----------+
         |
         |
         v
       +---+---+---+---+---+
Heap:  | H | e | y | ? | ? |
       +---+---+---+---+---+
```

### 4. usize 类型

多少内存用于保存指针呢？这取决于具体的机器的架构。机器中的每一个内存位置都有一个地址，通常表示成一个无符号的整数（usigned integer）。取决于内存空间的最大的大小，这个整数有不同的 大小（大多数是 32 位或者 64 位的内存空间）。

Rust 将这个依据架构的整数抽象成 usize：一个无符号整数，尽可能大的可以定位机器内存的字节数，在 32 位系统中为 u32，在 64 位系统中为 u64。String 中的 capacity、length 和 pointer 的类型都是 usize。

### 5. heap 没有 std::mem::size_of

std::mem::size_of 返回的是类型在栈上的内存空间的大小，也叫做类型的大小。但是堆中的内存不被视为 String 类型的大小的一部分。std::mem::size_of 不知道也不关心额外分配的堆内存。在堆中，没有提供对应的检测堆内存或者某个值的内存分配的大小的函数。

## 三、再谈引用 reference

Rust 中，引用，像 `&String`和`&mut String`，是如何在内存中表示的呢？

大多数引用（有 fat pointer 和 thin pointer 之分，所以不是全部），在 Rust 中，被表示成一个指向特定内存位置的指针，他和指针的大小一样，是一个 usize。可以通过 size_of 方法确认内存大小：

```rust
assert_eq!(std::mem::size_of::<&String>(), 8);
assert_eq!(std::mem::size_of::<&mut String>(), 8);
```

一个 `&String`，特别地，及时一个指向保存 String 元数据的内存位置的指针。如果运行下面的代码：

```rust
let s = String::from("Hey");
let r = &s;
```

内存状态如下：

```rust
           --------------------------------------
           |                                    |
      +----v----+--------+----------+      +----|----+
Stack | pointer | length | capacity |      | pointer |
      |  |      |   3    |    5     |      |         |
      +--|  ----+--------+----------+      +---------+
         |          s                           r
         |
         v
       +---+---+---+---+---+
Heap   | H | e | y | ? | ? |
       +---+---+---+---+---+
```

`&mut String` 同理。

**<u>不是所有的指针都指向堆内存位置！！！</u>**

## 四、解构器 Destructors

介绍堆时，提到，我们还要负责释放堆内存的分配。

介绍 borrow checker 时，提到我们在 Rust 中是不需要直接管理内存的。

上面两个陈述貌似是冲突，Rust 使用 域 Scopes 和 解构器 destructors 将两者结合到一起。

### 1. 作用域 Scopes

一个变量的作用域就是这个变量 合法 或者说 存活 的 Rust 代码范围。一个作用域开始于变量声明，结束于下面两种情况的一种：

1. 声明变量的代码块结束（{} 之间的代码块）

    ```rust
    fn main() {
       // `x` is not yet in scope here
       let y = "Hello".to_string();
       let x = "World".to_string(); // <-- x's scope starts here...
       let h = "!".to_string(); //   |
    } //  <-------------- ...and ends here
    ```

2. 变量的所有权被传递到其他地方了

    ```rust
    fn compute(t: String) {
       // Do something [...]
    }
    
    fn main() {
        let s = "Hello".to_string(); // <-- s's scope starts here...
                    //                    | 
        compute(s); // <------------------- ..and ends here
                    //   because `s` is moved into `compute`
    }
    ```

### 2. 解构器 Destructors

当数据项的所有者作用域结束后，Rust 会调用解构器 destructor。destructor 会尝试清理该数据项所分配的所有内存。可以通过`std::mem::drop`手动调用 destructor。

#### 可视化 drop 的调用

```rust
fn main() {
   let y = "Hello".to_string();
   let x = "World".to_string();
   let h = "!".to_string();
}

fn main() {
   let y = "Hello".to_string();
   let x = "World".to_string();
   let h = "!".to_string();
   // Variables are dropped in reverse order of declaration
   drop(h);
   drop(x);
   drop(y);
}
```

上下两段代码是等价的。下面也是：

```rust
// 1
fn compute(s: String) {
   // Do something [...]
}

fn main() {
   let s = "Hello".to_string();
   compute(s);
}

// 2
fn compute(t: String) {
    // Do something [...]
    drop(t); // <-- Assuming `t` wasn't dropped or moved 
             //     before this point, the compiler will call 
             //     `drop` here, when it goes out of scope
}

fn main() {
    let s = "Hello".to_string();
    compute(s);
}
```

需要注意的是，在 main 中的 compute 调用后，即使 s 已经不再合法了，但是 main 中不会再调用 drop 了，因为它的所有权已经移交到函数 compute 中了，同时也移交了清理其内存的权限。这样的设计保证了对于一个 valule 的 destructor 调用至多只会一次，不会造成二次释放的内存错误。

#### drop 引用

如果 drop 掉一个 value 的引用会发生什么？例如：

```rust
let x = 42i32;
let y = &x;
drop(y);
```

调用`drop(y)`什么都不会发生，如果你尝试编译这一段代码，会得到这样的一个 warning：

```rust
warning: calls to `std::mem::drop` with a reference 
         instead of an owned value does nothing
 --> src/main.rs:4:5
  |
4 |     drop(y);
  |     ^^^^^-^
  |          |
  |          argument has type `&i32`
  |
```

这就回到了之前所说的，我们只会调用 destructor 一次。对于一个 value 可以有多个 引用--如果我们对引用指向的 value 调用 destructor，那么剩余的引用指向的内存地址就不合法了，会造成悬空指针，一个和 use-after-free bug 十分相似的情况。Rust 所有权系统通过设计排除这种 bug。