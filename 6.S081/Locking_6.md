# Locking

​	大多数内核，包括 xv6 ，会交错执行多个活动。交错执行操作的一个来源是多处理器硬件：具有多个cpu独立执行的计算机，例如xv6的RISC-V。这些多核 CPU 共享物理主存--RAM，并且 xv6 利用这种共享维护所有 CPU 对数据结构的读写。这种共享增加了一个 CPU 读取数据而另一个 CPU 正在更新数据的可能性，甚至多个 CPU 同时更新相同的数据；如果不仔细设计，这种并行访问很可能产生不正确的结果或损坏的数据。即使是单处理器，内核可能会在大量的线程中切换 CPU，导致他们的执行是交错的。最后，当一个中断出现在错误的时刻，设备中断处理程序修改与某些可中断代码相同的数据可能会损坏数据。并行（concurrency）指的就是当多个指令流交错执行的情况，这种情况一般由多处理器、线程切换或中断导致。

​	内核充满了并行访问的数据。例如，两个 CPU 可以同时调用 kalloc，因此并行的 pop 出 freelist 的 head。内核设计者们喜欢允许大量的并行，因为这可以通过并行提高性能，并且还可以提高健壮性。然而，结果就是内核设计者需要花费巨大的努力去实现正确性，尽管存在这样的并发性。有许多方法可以达到正确的代码，其中一些方法比其他方法更容易推理。以并发性下的正确性为目标的策略，以及支持这些策略的抽象，被称为并发控制技术。

​	xv6 根据情况，使用大量的并发控制技术，同时还有很多的可能。这一章聚焦于一个广泛使用的技术：锁（lock）。锁提供互斥性，确保一次只有一个 CPU 可以持有该锁。如果程序员将锁与每个共享数据项关联，并且代码在使用项时总是持有关联的锁，那么该项一次将仅由一个 CPU 使用。在这种情况下，我们说锁保护数据项。尽管锁是一种易于理解的并发控制机制，但锁的缺点是它们会降低性能，因为它们序列化并发操作。

​	接下来解释为什么 xv6 需要锁，并且如何实现它，以及如何使用它。

![image-20240826160150874](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240826160150874.png)
