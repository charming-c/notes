# Occlum IO_Uring

[toc]

## 1. 总体设计

整个 occlum io_uring 的框架如下图所示：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241024151521676.png" alt="image-20241024151521676"  />

Occlum io_uring 的在整体架构主要可以分为三个部分：

- syscall 模块作为系统调用的 rust 封装，以调用内核中的 io_uring 功能

- io_uring 结构体封装，实现 linux io_uring 的高层抽象，简化原始接口的调用

- opcode 宏，应用可以直接通过 opcode 组装不同 IO 的 sqe，并交由上层 io_uring 与内核交互

整个调用过程可以分为 6 个步骤：

1. io_uring 结构体在内部创建 builder
2. 在 builder 中可选的设置不同的 param 参数，并最后通过 io_uring_setup 系统调用创建真正的 io_uring 实例
3. 在 builder 中完成内核和用户空间的 io_uring 相关内存的 mmap 映射，至此 io_uring 结构体创建完成
4. 应用可以通过 io_uring 的 SQ 以及 opcode 宏实现对应的 sqe，添加到队列尾
5. io_uring 通过 submitter 实现注册 buffer、提交 SQ 队列到内核的行为
6. 当从内核返回后，应用通过 io_uring 的 CQ 拿到对应的 IO 完成数据

为了保证内核和用户应用的 SQ、CQ 的数据一致性，实现了 sync 方法，用户需要在操作完 SQ tail 和 CQ head 后调用对应的 sync 方法从而实现内存一致。

## 2. system call 模块

首先分析 syscall 模块，了解如何实现对系统调用的 rust 封装。对于 syscall call 模块，其设计方案如下图所示：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20241024151632680.png" alt="image-20241024151632680"  />

首先通过 build.rs 利用 bindgen crate 将 libc 中和 io_uring 相关的函数和数据结构映射到 rust 中，生成 sys.rs 文件。利用 sys.rs 中的具体结构构造 mod.rs 两个系统调用模式：linux、sgx。

- 在没有 sgx 的 linux 中，通过 syscall 或者 sc::syscall 直接使用系统调用
- 在有 sgx 的情况下，将这里的 io_uring 系统调用定向到 ocall_io_uring。

这里的 ocall_io_uring 通过 edl 链接到外部可信任的函数，本质也是通过 syscall 直接调用的系统调用。

## 3. io_uring 模块

io_uring 模块将整个系统层面的数据抽象成三个部分：submitter、SQ、CQ。其中 SQ/CQ 负责和 io_uring 中对应的两个环形队列的操作，包括出入队列、判满等等。这里还抽象出独立的 submitter 作为应用程序向内核提交行为的中间层，包括向内核提交 register 的 buffer、fd 等等，同时还有通知内核 SQ 的组装完成，并等待 CQ 的事件交付。所以这里包含了两个`io_uring_enter`和`io_uring_register`系统调用的执行。

这里主要解析一下 sync 的使用：

```rust
pub fn push_to(&mut self, sq: &mut SubmissionQueue<'_>) {
    while self.count > 0 {
        unsafe {
            match sq.push(&self.entry) {
                Ok(_) => self.count -= 1,
                Err(_) => break,
            }
        }
    }

    sq.sync();
}

/// Synchronize this type with the real submission queue.
///
/// This will flush any entries added by [`push`](Self::push) or
/// [`push_multiple`](Self::push_multiple) and will update the queue's length if the kernel has
/// consumed some entries in the meantime.
#[inline]
pub fn sync(&mut self) {
    unsafe {
        (*self.queue.tail).store(self.tail, atomic::Ordering::Release);
        self.head = (*self.queue.head).load(atomic::Ordering::Acquire);
    }
}
```

对于 SQ，主要通过 push，push_multiple 将修改的 sqe 添加到 tail 处，在修改后，需要调用 sync 将此次修改刷新到内核中。同理 CQ 也是如此，以实现内存排序。

submitter 的主要函数功能为`submit_and_wait`、`register_files`，主要是提交 sqe 的更改到内核，以及注册对 fd 的持续引用：

**submit_and_wait:**

```rust
pub fn submit_and_wait(&self, want: usize) -> io::Result<usize> {
    let len = self.sq_len();
    let mut flags = 0;

    // This logic suffers from the fact the sq_cq_overflow and sq_need_wakeup
    // each cause an atomic load of the same variable, self.sq_flags.
    // In the hottest paths, when a server is running with sqpoll,
    // this is going to be hit twice, when once would be sufficient.

    if want > 0 || self.params.is_setup_iopoll() || self.sq_cq_overflow() {
        flags |= sys::IORING_ENTER_GETEVENTS;
    }

    if self.params.is_setup_sqpoll() {
        if self.sq_need_wakeup() {
            flags |= sys::IORING_ENTER_SQ_WAKEUP;
        } else if want == 0 {
            // The kernel thread is polling and hasn't fallen asleep, so we don't need to tell
            // it to process events or wake it up
            return Ok(len);
        }
    }

    unsafe { self.enter::<libc::sigset_t>(len as _, want as _, flags, None) }
}
```

这里主要是对 poll 模式以及内核侧轮询做了预处理，然后调用的 io_uring_enter 的系统调用。

**register_files:**

```rust
/// Register files for I/O. You can use the registered files with
/// [`Fixed`](crate::types::Fixed).
///
/// Each fd may be -1, in which case it is considered "sparse", and can be filled in later with
/// [`register_files_update`](Self::register_files_update).
///
/// Note that this will wait for the ring to idle; it will only return once all active requests
/// are complete. Use [`register_files_update`](Self::register_files_update) to avoid this.
pub fn register_files(&self, fds: &[RawFd]) -> io::Result<()> {
    execute(
        self.fd.as_raw_fd(),
        sys::IORING_REGISTER_FILES,
        fds.as_ptr() as *const _,
        fds.len() as _,
    )
    .map(drop)
}
```

execute 是对 io_uring_register 的简单封装，这里主要是针对 register 不同的类型，采用不同的 flags。

## 4. opcode 模块

通过宏编程，针对 io_uring 可操作的各种 IO 类型，实现对应的 Entry 的生成。例如：

```rust
opcode!(
    /// Accept a new connection on a socket, equivalent to `accept4(2)`.
    pub struct Accept {
        fd: { impl sealed::UseFixed },
        addr: { *mut libc::sockaddr },
        addrlen: { *mut libc::socklen_t },
        ;;
        flags: i32 = 0
    }

    pub const CODE = sys::IORING_OP_ACCEPT;

    pub fn build(self) -> Entry {
        let Accept { fd, addr, addrlen, flags } = self;

        let mut sqe = sqe_zeroed();
        sqe.opcode = Self::CODE;
        assign_fd!(sqe.fd = fd);
        sqe.__bindgen_anon_2.addr = addr as _;
        sqe.__bindgen_anon_1.addr2 = addrlen as _;
        sqe.__bindgen_anon_3.accept_flags = flags as _;
        Entry(sqe)
    }
);
```

通过 opcode 自动就生成了对应的 Accept 结构体，new 方法，以及可选择字段的设置方法和 build 方法。此时可以使用这样的代码构建一个 Entry：

```rust
let entry = opcode::Accept::new(types::Fd(fd), ptr::null_mut(), ptr::null_mut())
                .build()
// 可选		   .flags(...)
```
由于返回的就是 Entry，可以直接 push 到 SQ 中。