# Tokio 异步运行时

## 一、Task

tokio 通过 tokio::**spawn** 创建一个 task，但是 task 是不会立刻执行的。tokio::spawn 返回一个 JoinHandle，调用者可以通过它与 task 交互，例如获得 task 的返回值，或者等待 task 完成。

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Do some async work
        "return value"
    });

    // Do some other work

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}

//输出：GOT return value
```

对一个 JoinHandle 调用 await，会返回一个 Result。当一个 task 在执行的时候遭遇一个错误，JoinHandle 会返回一个 Err。这只会在一个 task 要么 panic 或者 task 被 tokio 运行时强制取消了。



 Task 是 tokio 中 scheduler 管理的执行的一个单元。Spawning task 会将 这个 task 提交给 Tokio scheduler，其会确保 task 在它需要 work 的时候运行。spawned task 可能会在它 spawn的同一个线程执行，或者也会在不同的 runtime thread 执行。



Task 在 tokoi 中是非常轻量级的。深入内层，他们只需要 64bytes 内存大小。

### 1. 'static 限制

当你在 tokio 运行时 spawn 一个 task，task 中的类型的声明周期必须是 'static 的。这意味着 spawned task 不能含有任何引用或者数据是属于 task 以外持有的。

> 这里的 'static 不是意味着永远存活 [解释链接](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program)
>
> 一般而言，static 修饰的变量有下面的性质：
>
> - 只能在编译时创建
> - 应该是不可变的，修改他们是 unsafe 的
> - 在整个程序生命周期中是合法的
>
> 但是 T: 'static 不是说 T 必须遵守上面的原则，而是说 T 可以被安全的持有无限长的时间，最多可以到程序结束的时候。**T 可以至少存活一个 'static 生命周期**
>
> - `T: 'static` should be read as *"`T` can live at least as long as a `'static` lifetime"*
>
> - if `T: 'static` then `T` can be a borrowed type with a `'static` lifetime *or* an owned type
>
> - since`T: 'static`includes owned types that means`T`
>
>     - can be dynamically allocated at run-time
>     - does not have to be valid for the entire program
>     - can be safely and freely mutated
>- can be dynamically dropped at run-time
>     - can have lifetimes of different durations

在 tokio 中，这样的代码编译不通过：

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

这是因为，默认情况下，变量不会将所有权 move 到异步代码块中。`v` vertor 继续被 `main` function 所有。`println!` 借用 `v`，rust 编译器就会报错，让我们 `async move { ... }`，这就会让编译器把 `v` move 到 spawned task。现在 task 就有了他所有数据的所有权，并且是 `'static` 的。

如果有一块数据必须在不止一个 task 中是可并发访问的，那么他就必须使用同步原语去共享，例如`Arc`.

### 2. Send 限制

Task 被 `tokio::spawn` 启动必须实现 Send。这使得 tokio 运行时能够在 task 阻塞是在 threads 之间切换 task。

Task 是 send 的，只有当 task 持有的所有跨越 await 的 data  都是 send 的。这是因为当  task 遇到 await 时，会让 scheduler 让出自己。下一次再执行 task 时，必然是从别的 task 在转移回来的，那么就要求自己的上下文是可以保存的，那么 await 之后的所有的变量应该都是要保存的。所以 task 中所有的跨越了 await 的变量都必须是 send 的。

## 二、 Shared state

在 tokio 中共享状态的方式有很多

- 使用 Mutex 保护共享变量
- 启动一个 task 管理共享状态，并且使用消息传递操作他

总的来说，第一种情况用于简单的数据，第二种方法用于一些异步任务例如 I/O 原语。

### 1. Arc && Mutex

对于共享变量，由于异步运行时所操作成的执行顺序的不确定，所以需要将共享的变量保护起来，常见的保护方式是：

```rust
type Db = Arc<Mutex<HashMap<String, Bytes>>>;
```

其中 Arc 保证了 Send 限制，可以在 task 切换时安全的传递，Mutex 保证了一次只会有一个 task 可以修改共享变量。

其中 std 和 tokio 都有 Mutex 的实现，不过区别是：

- std Mutex 会一直等待这个数据
- tokio Mutex 不会等待，而是切换到其他的 task 执行

### 2. Channels

Tokio 提供了多种 channel 可以进行 task 通信。

- [mpsc](https://docs.rs/tokio/1/tokio/sync/mpsc/index.html): multi-producer, single-consumer channel. Many values can be sent.
- [oneshot](https://docs.rs/tokio/1/tokio/sync/oneshot/index.html): single-producer, single consumer channel. A single value can be sent.
- [broadcast](https://docs.rs/tokio/1/tokio/sync/broadcast/index.html): multi-producer, multi-consumer. Many values can be sent. Each receiver sees every value.
- [watch](https://docs.rs/tokio/1/tokio/sync/watch/index.html): multi-producer, multi-consumer. Many values can be sent, but no history is kept. Receivers only see the most recent value.

也有一些其他的，比如 std 的 channel，不过他们一般都是通过阻塞当前线程，等待 message 传入。