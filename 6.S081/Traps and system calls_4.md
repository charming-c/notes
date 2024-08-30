# Traps and system calls

对于 OS，一般有三种事件会导致 CPU 将普通的指令暂停，并将控制移交给特殊的代码去处理对应的事件。

- 一种情况就是系统调用（system call）：当一个 user 程序执行 ecall 指令请求 内核 执行一些操作。
- 另一种情况是异常（exception）：一个指令（user 或者 kernel）导致了不合法的操作，例如除 0 或者使用了一个不合法的虚拟内存地址。
- 最后一种情况是中断（interrupt）：当一个 device 需要得到 CPU 关注，例如一个磁盘硬件完成了一个读写请求。

​	本书使用 trap 作为这三种情况的统一术语。通常，在 trap 发生时正在执行的代码稍后将需要恢复，并且不需要知道发生了什么特殊情况。也就是说，我们通常希望 trap 是透明的；这对于设备中断尤其重要，因为被中断的代码通常不希望发生这种中断。通常的顺序是 trap 强制将控制转移到内核；内核保存寄存器和其他状态（保存上下文），以便可以恢复执行（恢复现场）；内核执行适当的处理程序代码(例如，系统调用实现或设备驱动程序)；内核恢复保存的状态并从陷阱中返回；原始代码会从停止的地方恢复。

​	xv6 在内核处理所有的 trap，trap 不会交给 user code 处理。对于系统调用来说，在内核中处理陷阱是很自然的。这对于中断是有意义的，因为 os 的隔离要求只允许内核使用设备，而且内核是在多个进程之间共享设备的方便机制。它对异常也有意义，因为 xv6 通过杀死违规程序来响应来自用户空间的所有异常。

​	xv6 trap 处理分四个阶段进行：RISC-V CPU 采取的硬件操作，为内核 C 代码做准备的一些汇编指令，决定如何处理 trap 的 C 函数，以及系统调用或设备驱动程序服务例程。虽然三种陷阱类型之间的通用性表明内核可以用单个代码路径处理所有的 trap，但事实证明，为三种不同的情况使用单独的代码是很方便的：来自用户空间的 trap、来自内核空间的 trap 和计时器中断。处理陷阱的内核代码（汇编程序或C语言）通常称为处理程序（handler）；第一个处理程序指令通常是用汇编语言（而不是C语言）编写的，有时被称为向量（vector）。

## 一、RISC-V Trap 硬件机制

每个 RISC-V CPU 都有一组控制寄存器，给内核写入以告诉 CPU 怎么去处理 Traps，这样内核就能够读出并且发现一个 trap 发生了。以下是一些重要的寄存器及其作用：

- stvec：kernel 在这里写入 trap handler 的地址；RISC-V 会跳转到 stvec 中的地址执行去处理一个 trap。
- sepc：当一个 trap 发生时，RISC-V 将 PC 保存到这个寄存器，然后 PC 就被写入 stvec 去处理 trap 了。sret 指令会讲 sepc 的值复制回 pc。内核可以通过写 sepc 去控制 sret 返回到哪里。
- scause：RISC_V 会在这里放一个 number，用来描述 trap 的原因。
- sscratch：内核在这里放置了一个值，这个值在 trap handler 开始时会派上用场。
- sstatus：寄存器中的 SIE 位控制是否允许 device 中断，如果内核清除了SIE, RISC-V将延迟设备中断，直到内核设置了SIE。SPP位表示trap来自U mode还是s mode，并控制 sret 返回哪种模式。

以上的寄存器都是在 S mode 才可以使用，在 U mode 不能被读写。在处理 M mode 的 trap 时，也有一组相似的寄存器，xv6 只有在时钟中断的特殊情况下才会使用它们。每一个 CPU 核都有一组这样的寄存器，并且在任何时刻都可以处理 trap。当必须要进行 trap 时，RISC-V 硬件对于任何类型的 trap（除了 timer interrupts），都会执行如下操作：

- 如果 trap 是一个硬件中断，并且 sstatus 的 SIE 位被清除，就不执行下面的操作。
- 清除 sstatus 的中断位以关闭中断响应。
- 将 pc 复制给 sepc。
- 保存当前的 mode 位（U 或者 S）到 sstatus 的 SPP。
- 将 scause 设置为 trap 的 cause。
- 将 mode 设置为 S。
- 将 stvec 复制给 pc。
- 在新的 pc 处开始执行。

需要注意的是，CPU 并没有切换内核页表，没有切换内核中的 stack，并且没有保存除了 pc 以外的任何寄存器。内核必须执行这些任务。一个原因就是，trap 时 CPU 只做最少的工作，以供为软件提供更多的灵活性。例如，一些 os 在一些情况下会省略页表的切换以提高 trap 的性能。

值得思考的是，以上列出的所有步骤中，在为了更快的 trap 执行时，是否有能够省略的部分。虽然在某些情况下，更简单的顺序可以工作，但通常省略许多步骤将是危险的。例如，假设CPU没有切换pc。然后，来自用户空间的 trap 可以切换到S mode，同时仍然运行用户指令。这些用户指令可能会打破用户/内核隔离，例如，通过修改satp寄存器，使其指向一个允许访问所有物理内存的页表。因此，将CPU切换到内核指定的指令地址(即stvec)是很重要的。

## 二、来自用户空间的 Trap

​	xv6 对于来自 内核 和用户 的 trap 有着不同的处理。这一节讨论来自用户的 trap。用户程序在进行 系统调用、不合法操作或者 设备中断都会导致来自用户空间的 trap。一个高层次的执行路线是：`uservec->usertrap->usertrapret->userret`。对于 xv6 这样的 trap 处理流程最主要的限制就是，RISC-V 硬件在 trap 时并不会切换页表。这意味着 trap handler 的地址（写入 stvec 中的）必须在用户页表中有一个合法的映射，因为当 trap 处理的代码执行时，这是有效的页表。更长远看，xv6 的 trap 处理代码还需要切换内核页表，为了能够在执行这样的切换后，trap 处理代码能够继续执行，内核页表也需要一份这样的合法映射。

​	xv6 使用 trampoline page 来满足这样的需求。在 trampoline 中有 uservec（之后要设置在 stvec 寄存器中），这个 trampoline page 映射到了每一个进程的用户页表，同时也映射到了内核的内核页表。

​	当uservec启动时，所有32个寄存器都包含被中断的用户代码所拥有的值。这32个值需要保存在内存的某个地方，以便当trap返回到用户空间时可以恢复它们。存储到内存时，CPU需要使用一个寄存器来保存地址，但是在这一点上没有通用寄存器可用！幸运的是，RISC-V在表单中提供了 sscratch来帮助实现。uservec开头的csrrw指令交换a0和sscratch的内容。现在用户代码的 a0 被保存在 sscratch；uservec 有一个寄存器(a0)使用；a0包含内核之前放入sscratch 中的值。

​	uservec 接下来的任务就是保存剩下的 32 个通用寄存器。在进入用户空间之前，内核就已经设置了 sscratch 指向每个进程的 trapframe 结构体，trapframe 结构体包含了保存 32 个寄存器的内存。因为 satp 此时还是指向用户的页表，所以 trapframe 需要映射在用户页表地址空间中。当创建每一个进程的时候，xv6 会为每一个进程 alloc 一个 trapframe 的 page，并且映射在用户的虚拟地址空间 TRAPFRAME 中，就在 TRAMPOLINE 的下面，同时还有一个 p->trapframe 指向它，因为是物理地址，所以内核可以直接使用它。

​	因此，交换完 a0 和 sscratch 的值以后，a0 持有一个当前进程 trapframe 的指针。uservec 现在也保存了所有用户空间的寄存器的值，包括用户空间的 a0，从 sscratch 中读到。

​	trapframe 含有当前进程内核栈的地址，当前 CPU 的 hartid，usertrap 函数的地址，以及内核页表的地址。uservec 检索这些值，将 satp 转化为内核页表，并且调用 usertrap 函数。

​	usertrap 的作用是，确定 trap 的原因，处理它，并且返回。首先它改变了 stvec，以至于在内核空间的 trap 会被 kernelvec 而不是 uservec 处理。它保存 sepc 寄存器的值（保存了 用户空间的 PC），因为 usertrap 可能会调用 yeild，切换到另一个进程的内核线程，并且那个进程可能会返回到用户空间，在这个过程就会修改 sepc 寄存器。如果这个 trap 是一个 syscall，usertrap 会调用 syscall 函数来处理它，如果是一个 硬件中断，devintr；否则就是一个异常，并且内核会杀死错误的进程。整个 system call 路径会给保存的用户空间的 PC 加 4，因为 RISC-V，在执行 system call 情况下，在程序执行 ecall 时会离开用户空间，但是用户代码需要在 ecall 后的子序列继续执行。在退出时，usertrap检查进程是否已被终止或应该放弃CPU（如果此trap是计时器中断）。 

​	返回到用户空间的第一步是调用 usertrap 函数。这个函数会设置 RISC-V 控制寄存器，已准备将来来自用户空间的 trap。这包括将 stvec 改为 uservec，准备 trapframe 的值，设置 sepc 为追安保存的用户空间的 PC。在最后，usertrapret 调用 userret（在 trampoline中），原因是 userret 会切换页表。

​	usertrap 调用 userret，传递 TRAPFRAME 在 a0，用户页表的指针在 a1。userret 切换 satp 到用户页表。（回想，用户页表和内核页表中 trampoline 页映射到了相同的地址，这使得 uservec 在切换了页表后仍旧可以执行）。userret 复制 a0 中保存的 trapframe 到 sscratch 中，为之后的 TRAPFRAME 交换做准备。从这个时候开始，userret 可以使用的数据只有来自寄存器和 trapframe 页。下一步，userret 恢复保存在 trapframe 寄存器的值，最终交换 a0 和 sscratch，恢复 用户空间的 a0，并且保存 TRAPFRAME 为下一次 trap，并且执行 sret 返回到用户空间。

## 三、内核 trap

​	xv6 在处理 CPU trap 的时候取决于执行环境是在 user 还是 kernel。当内核在 CPU 执行时，内核会将 stvec 写入 kernelvec 的地址。因为内核 trap 只会来自内核，所以 satp 已经是内核页表，就不需要切换，并且 sp 应该也已经是内核的合理 stack 位置，所以 kernelvec 需要做的就是将 32 个寄存器保存到栈上，之后可以没有干扰的恢复被中断的内核代码的上下文环境。

​	kernelvec 负责将寄存器的值保存到对应线程的栈上，需要注意的是 trap 会不会导致线程的切换，因为切换了线程后，sp 会指向不同的栈地址，从而无法恢复之前的上下文环境。

​	kernelvec 最后跳转执行 kerneltrap。kerneltrap 为剩下两种类型的 trap 做准备：硬件中断和异常。调用 devintr 来检查并处理。如果 trap 不是一个硬件中断，那就一定是一个异常，并且这一定是一个致命的错误如果出现在 xv6 中，内核会调用 panic 并且停止执行。

​	如果 kerneltrap 是因为时钟中断被调用，并且一个进程的内核线程正在运行，kerneltrap 会调用 yield 让出 CPU 交给其他的线程执行。在某个时刻，那个线程也会 yield，就会让我们这个线程和 kerneltrap 继续执行下去。

​	当 kerneltrap 工作做完，他需要回到被 trap 中断的代码处继续执行。因为 yield 可能已经修改了 sepc 和 sstatus 的 mode，kerneltrap 需要在一开始就保存他们的值，并且现在要恢复这写控制寄存器的值，并返回 kernelvec。kernelvec 再将之前保存的上下文恢复到寄存器中，并且执行 sret，从而将 sepc 中的值拷贝到 pc，恢复执行被中断的内核代码。

​	值得思考的是，如果是因为时钟中断导致 kerneltrap 调用 yield，那么 trap 是如何返回的。

​	xv6 设置 CPU 的 stvec 到 kernelvec，是当 CPU 从用户空间进入内核空间的时候。但是这中间是有一个时间窗的，此时内核正在执行，但是 kernel 还是 uservec，在这个时间窗内没有硬件中断发生是至关重要的。幸运的是，RISC-V 总是禁用中断，当开始执行 trap 流程，直到再次设置 stvec 才开始启用。

## 四、缺页中断（page fault）

这部分 lazy allocation 和 cow fork 实验已经写过了。