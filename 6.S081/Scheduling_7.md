# Scheduling

​	几乎所有操作系统可能都可以同时运行比电脑 CPU 更多的程序，所以一个让 CPU 在进程之间共享的方案是十分需要的。理想情况下，这种共享对于用户程序来说应该是透明的。一种常见的方法是为每一个进程一个他拥有一个虚拟 CPU 的假象，通过进程在 CPU 硬件之间的多路复用实现。这一章节主要介绍 xv6 如何实现多路复用。

## 一、Multiplexing

​	Xv6 利用多路复用，将 CPU 从一个进程切换到另一个主要由两种情况。第一种，xv6 的 sleep 和 wakeup 机制，在一个进程等待一个硬件或者管道 I/O 完成或者等待一个子进程退出时，或者在 sleep 系统调用种等待时，发生切换。第二种，xv6 间断地强制发起一个切换，为了处理进程长时间占用 CPU 而不进行 sleep。这种多路复用的机制为每个进程创建了一种抽象，就是它们都有自己的 CPU 在执行，就像 xv6 使用 内存分配 和 硬件页表 是为了给进程创建它们都有自己的内存的抽象。

​	实现多路复用有一些挑战。首先，如果从一个进程切换到另一个进程。虽然切换上下文的想法很简单，但是实现却是 xv6 种最不透明的代码。第二，如何迫使切换是一种对于用户进程来说是透明的方式。xv6 使用标准的技术，也就是硬件计时器中断驱动上下文切换。第三，所有的 CPU 是在相同的共享的进程集和中进行切换，因此，一种避免竞争的锁机制是十分必要的。第四，一个进程的内存和其他的资源，在进程 free 的时候也必须释放，但是他不能都有它自己来完成，因为，比如说进程是无法释放自己的内核栈的，因为这个时候还在使用它。第五，对于一个多核机器的每一个核，必须记住他正在执行的进程，以至于系统调用可以影响正确的进程的内核栈。最终，sleep 和 wakeup 允许一个进程放弃 CPU 并且等待被其他的进程或者中断唤醒。需要注意竞争，因为会导致一些 wakeup 通知就丢失了。xv6 尝试解决这些问题尽可能简单，但是这无论如何都会导致代码有些难懂。

![image-20240830164323051](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240830164323051.png)

## 二、Context switching

上面的图给出了如何实现进程之间的上下文切换，从而达到进程调度。其中，kstack scheduler 的栈是一开始初始化的时候给 CPU 分配的，然后内核栈，是进程管理时为每个进程在内核创建的，映射到内核虚拟内存的高位。接下来我们看一下 context 结构体和 swtch 汇编代码，来阐述一下如何工作的：

**首先是 context：**

```c
struct context {
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

可以看到，context 中主要包括 ra 和 sp 两个寄存器，以及通用目的寄存器中的所有 callee saved 寄存器，这是为什么呢？首先看一下 swtch 的实现：

```assembly
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.	


.globl swtch
swtch:
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        
        ret

```

Xv6 中的 swtch 的作用就是第一个 context 的内存保存到内核栈，然后将另一个 context 中的内存恢复到寄存器中，由于 context 中是有写 ra 的，所以在 swtch 的最后一条指令 ret 执行以后，pc 自动加载到 ra 执行，也就是说，调用 swtch 后下一步的执行是由第二个 conetxt 的 ra 决定的。

<img src="/Users/charming/Library/Application Support/typora-user-images/image-20240830170138953.png" alt="image-20240830170138953" style="zoom:50%;" />

​	如图所示，在调用 swtch 后，由于 swtch 是一个函数调用，ra 会自动写入 swtch 函数后下一条指令的地址，然后在 swtch 中我们保存了 ra 和其他寄存器的值（由于这些是内核中执行的，为了保护内核和用户的环境隔离，用户进程在内核中执行需要使用内核栈，这也是为什么保存和改变 sp，不保存 caller-saved 寄存器的值是因为编译器在编译成汇编时，会自动在调用 swtch 的函数中就保存好这些寄存器的值）。这样就跳转到另一个进程执行。我们再换个角度，如果 p2 是我们之前保存过的一个进程，那么此时它的 ra 就是上一次 swtch 的下一条地址了，那么这里我们在 swtch 的 ret 后就自动执行到了 p2 上一次 swtch 调用后的地址了。对于 p1 和 p2 来说，每次的 swtch 都是一个函数调用，这个函数调用对自己没有任何影响，执行完后，继续执行 swtch 后面的指令，但是实际上我们知道，我们在这次调用中，CPU 去执行另一个进程了，然后才回来继续执行这个进程。

## 三、Scheduling

​	有了上面的用户进程无感知的上下文切换，那么我们就可以自然地实现分时调度进程的执行了。在 xv6 内核第一次调度的时候，也就是 main 中 scheduler 方法，会将 cpu 的 context 保存，然后 swtch 去执行第一个进程的 context，而第一个进程的内存目前来说是这样的，trapframe 只设置了简单的 epc 和 sp，其中讲 initcode 映射到第一个进程的第一页，第二页是 stack，context 也很简单，只设置了 ra 为 forkret 函数，sp 为此进程的 ksatck。所以第一次调度之后，swtch 之后会执行 forkret。forkret 是每一个新建的进程第一次 scheduling 调度都会执行的地方。forkret 调用 usertrapret 回到用户态，并且 epc 之前设置为 0，所以这个时候开始从头开始执行 initcode，然后 initcode 中就是 exec init 进程，通过 exec 系统调用创建进程 init，init 再 fork 出一个 shell。

​	在正常启动之后，scheduling 主要就通过时钟中断以及 sleep、wakeup 这样的系统调用实现了。在这里主要解释一下 xv6 中 sleep 和 wakeup 的实现。

```c
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();
  
  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won't miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.
  if(lk != &p->lock){  //DOC: sleeplock0
    acquire(&p->lock);  //DOC: sleeplock1
    release(lk);
  }

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}

// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == SLEEPING && p->chan == chan) {
      p->state = RUNNABLE;
    }
    release(&p->lock);
  }
}
```

需要注意的是，我理解的是，这里的 wakeup 相当于一个广播。sleep 根据设置 p->chan ，然后让 CPU 调度其他的进程执行，然后 wakeup 会针对所有的 p->chan 将对应的处于 sleep 的进程唤醒。这里会导致所有 sleep 的进程都会醒一次，但是醒了没有用，可能有多个进程在 sleep，能拿到这一个 chan 的只有一个进程，所以这里 sleep 出来以后，需要外套一个 while 循环，检查自己是不是抢到了 wakeup 广播的 chan，不然就继续 sleep 了。