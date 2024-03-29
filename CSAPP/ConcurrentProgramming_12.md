# 并发编程

## 一、基于进程的并发编程

构造并发程序最简单的方法就是用进程，使用那些熟悉的函数，像 fork、exec 和 waitpid。例如，一个构造并发服务器的自然方法就是，在父进程中接受客户端连接请求，然后创建一个新的子进程来为每个客户提供服务。为了了解这是如何工作的，假设我们有两个客户端和一个服务器，服务器正在监听一个监听描述符（比如描述符 3）上的连接请求。现在假设服务器接受了连接请求之后，服务器派生一个子进程，这个子进程获得服务器描述表的完整副本。子进程关闭他的副本中的监听描述符 3，而父进程关闭他的已连接描述符 4 的副本，因为不再需要这些描述符了。这就得到下图的状态，其中子进程正忙于为客户端提供服务。

![image-20240111145541477](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240111145541477.png)

因为父、子进程中的已连接描述符都指向同一个文件表表项，所以父进程关闭它的已连接描述符是至关重要的。否则，将永远不会释放已连接描述符 4 的文件表条目，而且由此引起的内存泄漏将最终消耗光可用的内存，使系统崩溃。

现在，假设在父进程为客户端 1 创建了子进程之后，它接受一个新的客户端 2 的连接请求，并返回一个新的已连接描述符（比如描述符 5），如下图所示。然后，父进程又派生另一个子进程，这个子进程用已连接描述符 5 为它的客户端提供服务，如下图所示。此时，父进程正在等待下一个连接请求，而两个子进程正在并发地为它们各自的客户端提供服务。

![image-20240111150214118](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240111150214118.png)

### 1. 基于进程的并发服务器

下面展示了一个基于进程的并发 echo 服务器的代码。关于这个服务器，有以下几点重要内容需要说明：

- 首先，通常服务器会运行很长的时间，所以我们必须要包括一个 SIGCHLD 处理程序，来回收僵死（zombie）子进程的资源。因为当 SIGCHLD 处理程序执行时，SIGCHLD 信号是阻塞的，而 Linux 信号是不排队的，所以 SIGCHLD 处理程序必须准备好回收多个僵死子进程的资源。
- 其次，父子进程必须关闭它们各自的 connfd 副本。就像前面提到过的一样，这对父进程而言尤为重要，他必须关闭它的已连接描述符，以避免内存泄漏。
- 最后，因为套接字文件表表项中的引用计数，直到父子进程中的 connfd 都关闭了，到客户端的连接才会终止。

```c
#include "csapp.h"
void echo(int connfd);

void sigchld_handler(int sig) //line:conc:echoserverp:handlerstart
{
    while (waitpid(-1, 0, WNOHANG) > 0)
	;
    return;
} //line:conc:echoserverp:handlerend

int main(int argc, char **argv) 
{
    int listenfd, connfd;
    socklen_t clientlen;
    struct sockaddr_storage clientaddr;

    if (argc != 2) {
	fprintf(stderr, "usage: %s <port>\n", argv[0]);
	exit(0);
    }

    Signal(SIGCHLD, sigchld_handler);
    listenfd = Open_listenfd(argv[1]);
    while (1) {
	clientlen = sizeof(struct sockaddr_storage); 
	connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
	if (Fork() == 0) { 
	    Close(listenfd); /* Child closes its listening socket */
	    echo(connfd);    /* Child services client */ //line:conc:echoserverp:echofun
	    Close(connfd);   /* Child closes connection with client */ //line:conc:echoserverp:childclose
	    exit(0);         /* Child exits */
	}
	Close(connfd); /* Parent closes connected socket (important!) */ //line:conc:echoserverp:parentclose
    }
}
/* $end echoserverpmain */
```

### 2. 进程的优劣

对于在父、子进程共享状态信息，进程有一个非常清晰的模型：共享文件表，但是不共享用户地址空间。进程有独立的地址空间既是优点也是缺点。这样一来，一个进程不可能不小心覆盖另一个进程的虚拟内存，这就消除了许多令人迷惑的错误-这是一个明显优点。

另一方面，独立的地址空间使得进程共享状态信息变得更加困难。为了共享信息，他们必须使用显式的 IPC（进程间通信）机制。基于进程的设计的另一个缺点是，它们往往比较慢，因为进程控制和 IPC 的开销很高。

> 在本书中，我们已经遇到好几个 IPC 的例子。在第 8 章的 waitpid 函数和信号是基本的 IPC 机制，它们允许进程发送小消息到同一主机上的其他进程。第 11 章的套接字接口是 IPC 的一种重要形式，它允许不同主机上的进程交换任意的字节流。然而，术语 Unix IPC 通常指的是所有允许进程和同一台主机上的其他进程进行通信的技术。其中包括管道、先进先出（FIFO）、系统 V 共享内存，以及系统 V 信号量（semaphore）。

## 二、基于 I/O 多路复用的并发编程

假设要求你编写一个 echo 服务器，它也能对用户从标准输入键入的交互命令作出反应。在这种情况下，服务器必须响应两个互相独立的 I/O 事件：

- 网络客户端发起连接请求
- 用户在键盘键入命令行

我们先等待哪个时间？没有哪个选择是理想的。如果在 accept 中等待一个连接请求，我们就不能响应输入的命令。类似地，如果在 read 中等待一个输入命令，我们就不能响应任何连接请求。针对这种困境的一个解决方法就是 I/O 多路复用（I/O multiplexing）技术。基本的思路就是使用 select 函数，要求内核挂起进程，只有在一个或者多个 I/O 事件发生以后，才将控制返回给应用程序，就像下面的示例中的一样：

- 当集合 {0, 4} 中任意描述符准备好读时返回。
- 当集合 {1, 2, 7} 中任意描述符准备好写时返回。
- 如果在等待一个 I/O事件发生时过了 152.13 秒，就超时。

select 函数是一个复杂的函数，有许多不同的使用场景。我们只讨论第一种场景：等待一组描述符准备好读。

```c
#include<sys/select.h>

int select(int n, fd_set *fdset, NULL, NULL, NULL);
// 返回已准备好的描述符的非 0 的个数，如出错则为 -1

FD_ZERO(fd_set *fdset);	// Clear all bits in fdset
FD_CLR(int fd, fd_set *fdset);	// Clear bit fd in fdset
FD_SET(int fd, fd_set *fdset);	// Turn on bit fd in fdset
FD_ISSET(int fd, fd_set *fdset);	// Is bit fd in fdset on?
// 处理描述符集合的宏
```

select 函数处理类型为 fd_set 的集合，也叫做描述符集合。逻辑上，我们将描述符集合看成一个大小为 n 的位向量：
$$
b_{n-1},···,b_1,b_0
$$
每个位 $b_k$ 对应于描述符 k。当且仅当 b_k = 1，描述符 k 才表明是描述符集合的一个元素。只允许你对描述符集合做三件事：

- 分配它们
- 将一个此种类型的变量赋值给另一个变量
- 用 FD_ZERO、FD_SET、FD_CLR 和 FD_ISSET 宏来修改和检查它们

针对我们的目的， select 函数有两个输入：一个称为读集合的描述符集合（fdset）和该读集合的基数（n）（实际上是任何描述符集合的最大基数）。select 函数会一直阻塞，直到读集合中至少一个描述符准备好可以读。当且仅当一个从该描述符读取一个字节的请求不会阻塞时，描述符 k 就表示准备好可以读了。select 有一个副作用，它修改参数 fdset 指向的 fd_set，指明读集合的一个子集，称为准备好集合（ready set），这个集合是由读集合中准备好可以读了的描述符组成的。该函数返回的值指明了准备好集合的基数。注意，由于这个副作用，我们必须在每次调用 select 时都更新读集合。

理解 select 函数的最好办法是研究一个例子。下图展示了可以如何利用 select 来实现一个迭代的 echo 服务器，它也可以接受标准输入上的用户命令。一开始，我们用 open_clientfd 函数打开一个监听描述符（第 16 行），然后使用 FD_ZERO 创建一个空的读集合（第 18 行）：

![image-20240111182744486](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240111182744486.png)

接下来，在第 19 和第 20 行中，我们定义由描述符 0（标准输入）和描述符 3（监听描述符）组成的读集合：

![image-20240111183334963](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240111183334963.png)

在这里，我们开始典型的服务器循环。但是我们不调用 accept 函数来等待一个连接请求，而是调用 select 函数，这个函数会一直阻塞，直到监听描述符或者标准输入准备好可以读（第 24 行）。例如，下面是用户按下回车键，因此使得标准输入描述符变为可读时，select 会返回的 ready_set 的值：

![image-20240111183908191](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240111183908191.png)

一旦 select 返回，我们就用 FD_ISSET宏指令来确定哪个描述符准备好可以读了。如果是标准输入准备好了（第 25 行），我们就调用 command 函数，该函数在返回到主程序之前，会读解析和响应命令。如果是监听描述符准备好了（第 27 行），我们就调用 accept 函数来得到一个已连接描述符，然后调用 echo 函数，他会将来自客户端的每一行又回送回去，直到客户端关闭这个连接中的它的那一端。

虽然这个程序是使用 select 得一个很好的示例，但是它仍然留下了一些问题待解决。问题是一旦连接到某个客户端，就会连续回送输入行，直到客户端关闭这个连接中的它的那一段。因此，如果键入一个命令到标准输入，你将不会得到响应，直到服务器和客户端之间结束。一个更好的办法是更细粒度的多路复用，服务器每次循环（至多）回送一个文本行。

![image-20240116172356691](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240116172356691.png)

### 1. 基于 I/O 多路复用的并发事件驱动服务器

I/O 多路复用可以用作并发事件驱动（event-driven）程序的基础，在事件驱动程序中，某些事件会导致流向前推进。一般的思路是将逻辑流模型转化为状态机。不严格的说，一个状态机（state machine）就是一组状态（state）、输入事件（input event）和转移（transition），其中转移是将状态和输入事件映射到状态。每个状态是将一个（输入状态、输入事件）对映射到一个输出状态。自循环（self-loop）是同一输入和输出状态之间的转移。通常把状态机画成有向图，其中节点表示状态，有向弧表示转移，而弧上的标号表示输入事件。一个状态机从某种初始状态开始执行。每个输入事件都会引发一个从当前状态到下一状态的转移。

对于每个新客户端 k，基于 I/O 多路复用的并发服务器会创建一个新的状态机 s_k，并将它和已连接描述符 d_k 联系起来。如图所示，每个状态机 s_k都有一个状态（等待描述符 d_k 准备好可读）、一个输入事件（描述符 d_k 准备好可以读了）和一个转移（从描述符 d_k 读一个文本行）。

![image-20240118163832349](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240118163832349.png)

服务器使用 I/O 多路复用，借助 select 函数检测输入事件的发生。当每个已连接描述符准备好可读时，服务器就为相应的状态机执行转移，在这里就是从描述符读和写回一个文本行。

下图展示了一个基于 I/O 多路复用的并发事件驱动器的完整示例代码。一个 pool 结构里维护着活动客户端的集合（第 3～11 行）。在调用 init_pool 初始化池（第 27 行）之后，服务器进入一个无限循环。在循环的每次迭代中，服务器调用 select 函数来检测两种不同类型的输入事件：

- 来自一个新客户端的连接请求到达
- 一个已存在客户端的的已连接描述符准备好可以读了

当一个连接请求到达时（第 35 行），服务器打开连接（第 37 行），并调用 add_client 函数，将该客户端添加到池里（第 38 行）。最后，服务器调用 check_clients 函数，把来自每个准备好的已连接描述符的一个文本行回送回去（第 42 行）。

![image-20240118164536296](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240118164536296.png)

![image-20240118170458817](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240118170458817.png)

init_pool 函数初始化客户端池。clientfd 数组表示已连接描述符的集合，其中整数 -1 表示一个可用的槽位。初始时，已连接描述符是空的（第 5～7 行），而且监听描述符是 select 读集合中唯一的描述符（第 10～12 行）。

![image-20240118170932328](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240118170932328.png)

add_client 函数（如下图），添加一个新的客户端到活动客户端池中。在 clientfd 数组中找到一个空槽位之后，服务器将这个已连接描述符添加到数组中，并初始化相应的 RIO 读缓冲区，这样一来我们就能够对这个描述符调用 rio_readlineb （第 8～9 行）。然后，我们将这个已连接描述符添加到 select 读集合（第 12 行），并更新该池的一些全局属性。maxfd 变量（第 15～16 行）记录了 select 的最大文件描述符。 maxi 变量（第 17～18 行）记录的是到 clientfd 数组的最大索引，这样 check_clients 函数就不需要搜索整个数组了。

![image-20240118175102241](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240118175102241.png)

下图中的 check_client 函数来回送来自每个准备好的已连接描述符的一个文本行。如果成功地从描述符读取了一个文本行，那么就将该文本行回送到客户端（第 15～18 行）。注意，在第 15 行我们维护着一个从所有客户端接收到的全部字节的累计值。如果因为客户端关闭这个连接中它的那一端，检测到 EOF，那么关闭这边的连接端（第 23 行），并从池中清除掉这个描述符（第 24～25 行）。

根据上图中的有限状态模型，select 函数检测到输入事件，而 add_client 函数创建一个新的逻辑流（状态机），check_clients 函数回送输入行，从而执行状态转移，而且当客户端完成文本行发送时，它还要删除这个状态机。

![image-20240118180250478](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240118180250478.png)

### 2. I/O 多路复用技术的优劣

上图中的服务器提供了一个很好的基于 I/O 多路复用的事件驱动编程的优缺点示例。事件驱动设计的一个优点就是，它比基于进程的设计给了程序员更多的对程序行为的控制。例如，我们可以设想编写一个事件驱动的并发服务器，为某些客户端提供他们需要的服务，而这对于基于进程的并发服务器来说，是很困难的。

另一个优点是：一个基于 I/O 多路复用的事件驱动服务器是运行在单一进程上下文中的，因此每个逻辑流都能访问该进程的全部地址空间。这使得在流之间共享数据变得容易。一个作为单个进程运行相关的优点是，你可以利用熟悉的调试工具，例如 GDB，来调试你的并发服务器，就像对顺序程序那样。最后，事件驱动的设计常常比基于进程的设计高效得多，因为他不需要进程上下文切换来调度新的流。

事件驱动的一个明显的缺点是编码复杂。我们的事件驱动的并发服务器需要的代码是基于进程的 echo 服务器的三倍，并且很不幸，随着并发粒度的减小，复杂性还会上升。这里的粒度是指每个逻辑流每个时间片执行的指令数量。例如，在示例并发服务器中，并发粒度就是读一个完整的文本行所需要的指令数量。只要某个逻辑流正忙于读一个文本行，其他逻辑流就不会有进展。对于我们的例子来说这没问题，但是它使得在“故意只发送部分文本行然后停止”的恶意客户端攻击面前，我们的基于事件驱动服务器显得很脆弱。修改事件驱动服务器来处理部分文本行不是一个简单的任务，但是基于进程的设计就能处理的很好，而且是自动处理的。基于事件的并发服务器的另一个缺点是不能充分利用多核处理器。

## 三、基于线程的并发编程

到目前为止，我们已经看到了两种创建并发逻辑流的方法。在第一种方法里，我们为每个流使用了单独的进程。内核会自动调度每个进程，而每个进程有它自己的私有地址空间，这使得流共享数据很困难。在第二种方法中，我们创建自己的逻辑流，并利用 I/O 多路复用来显式地调度流。因为只有一个进程，所有流共享整个地址空间。本节介绍第三种方法--基于线程，它是两种方法的混合。

线程（thread）就是运行在进程上下文中的逻辑流。在本书里迄今为止，程序都是由每个进程中的一个线程组成的。但是现代系统也允许我们编写一个进程里同时运行多个线程的程序。程序由内核自动调度。每个线程都有它自己的线程上下文（thread context），包括一个唯一的整数线程 ID（Thread ID， TID）、栈、栈指针、程序计数器、通用目的寄存器和条件码。所有运行在一个进程里的线程共享该进程的整个虚拟地址空间。

基于线程的逻辑流结合了基于进程和基于 I/O 多路复用的流的特性。同进程一样，线程由内核自动调度，并且内核通过一个整数 ID 来识别线程。同基于 I/O 多路复用的流一样，多个线程运行在单一进程的上下文中，因此共享这个线程虚拟地址空间的所有内容，包括它的代码、数据、堆、共享库和打开的文件。

### 1. 线程执行模型

多线程的执行模型在某些方面和多进程执行模型是相似的。每个进程开始生命周期时都是单一线程，这个线程称为主线程（main thread）。在某一时刻，主线程创建一个对等线程（peer thread），从这个时间点开始，两个线程就并发的执行。最后，因为主线程执行一个慢速系统调用，例如 read 或者 sleep，或者因为被系统的间隔计时器中断，控制就会通过上下文切换传递到对等线程。对等线程会执行一段时间，然后控制传递回主线程，依此类推。

在一些重要方面，线程执行是不同于进程的。因为一个线程的上下文要比一个进程的上下文小得多，线程的上下文切换比进程的上下文切换快得多。另一个不同就是线程不像进程那样，不是按照严格的父子层次来组织的。和一个进程相关的线程组成一个对等的线程池，独立于其他进程创建线程。主线程和其他线程的区别仅在于它总是进程中第一个运行的线程。对等线程池的主要影响是，一个线程可以杀死它的任何对等线程，或者等待它的任意对等线程终止。另外，每个对等线程都能读写相同的共享数据。

### 2. Posix 线程

Posix 线程（Pthreads）是在 C 程序中处理线程的一个标准接口。它最早出现在 1995 年，而且在所有的 Linux 系统上都可用。Pthreads 定义了大约 60 个函数，允许程序创建、杀死和回收线程，与对等线程安全的共享数据，还可以通知对等线程系统状态的变化。

下图展示了一个简单的 Pthreads 程序。主线程创建了一个对等线程，然后等待它的终止。对等线程输出 "hello,world!\n" 并且终止。当主线程检测到对等线程终止后，它就通过调用 exit 终止该进程。这是我们看到的第一个线程化的程序，所以让我们仔细地解析它。线程的代码和本地数据被封装在一个线程例程（thread routine）中。正如第 2 行里的原型所示，每个线程例程都以一个通用指针作为输入，并返回一个通用指针。如果想传递多个参数给线程例程，那么你就应该将参数放到一个结构中，并传递一个指向该结构的指针。相似的，如果想要线程例程返回多个参数，你可以返回一个指向一个结构的指针。

![image-20240128170837711](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128170837711.png)

### 3. 创建线程

线程通过调用 pthread_create 函数来创建其他线程。

![image-20240128172110729](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128172110729.png)

pthread_create 函数创建一个新的线程，并带着一个输入变量 arg，在新线程的上下文中运行线程例程 f。能用 attr 参数来改变新创建线程的默认属性。当 pthread_create 返回时，参数 tid 包含新创建线程的 ID。新线程可以通过调用 pthread_self 函数来获得他自己的线程 ID。

![image-20240128172401670](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128172401670.png)

### 4. 终止线程

一个线程是以下列方式之一来终止的：

- 当顶层的线程例程返回时，线程会隐式地终止。
- 通过调用 pthread_exit 函数，线程会显式地终止。如果主线程调用 pthread_exit，它会等待所有其他对等线程终止，然后再终止主线程和整个进程，返回值为 thread_return。

![image-20240128172901299](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128172901299.png)

- 某个对等线程调用 Linux 的 exit 函数，该函数终止进程以及所有与该进程相关的线程。
- 另一个对等线程通过以当前线程 ID 作为参数调用 pthread_cancel 函数来终止当前线程。

![image-20240128173119390](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128173119390.png)

### 5. 回收已终止线程的资源

线程通过调用 pthread_join 函数等待其他线程终止。

![image-20240128173344041](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128173344041.png)

pthread_join 函数会阻塞，直到线程 tid 终止，将线程例程返回的通用（void *）指针赋值为 thread_return指向的位置，然后回收已终止线程占用的内存资源。注意，和 Linux 的 wait 函数不同，pthread_join 函数只能等待一个置顶的线程终止。没有办法让 pthread_wait 等待任意一个线程终止。这使得代码更加复杂，因为它迫使我们去使用其他一些不那么直观的机制来检测进程的终止。

### 6. 分离线程

在任何一个时间点上，线程是可结合的（joinable）或者是分离的（detached）。一个可结合的线程能够被其他线程收回和杀死。在被其他线程回收之前，它的内存资源（例如 栈）是不释放的。相反，一个分离的线程是不能被其他线程回收或杀死的。它的内存资源在它终止时由系统自动释放。

默认情况下，线程被创建是可结合的。为了避免内存泄漏，每个可结合线程都应该要么被其他线程显式地回收，要么通过调用 pthread_detach 函数

被分离。

![image-20240128174755074](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240128174755074.png)

pthread_detach 函数分离可结合线程 tid。线程能够通过以 pthread_self（） 为参数的 pthread_detach 调用来分离它自己。尽管我们的一些例子会使用可结合线程，但是在现实程序中，有很好的理由要使用分离的线程。例如，一个高性能 Web 服务器可能在每次收到 Web 浏览器的连接请求时都创建一个新的对等线程。因为每个连接都是由一个单独的线程独立处理的，所以对于服务器而言，就很没有必要显式地等待每个对等进程终止。在这种情况下，每个对等线程都应该在它开始处理请求之前分离它自身，这样就能在它终止后回收它的内存资源了。

### 7. 初始化线程

pthread_once 函数允许你初始化与线程例程相关的状态。

```c
#include<pthread.h>
pthread_once_t once_control = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t once_control, void (* init_routine)(void));
// 总是返回 0
```

once_control 变量是一个全局或者静态变量，总是被初始化为 PTHREAD_ONCE_INIT。当你第一次使用参数 once_control 调用 pthread_once 时，它调用 init_routine，这是一个没有输入参数、也不返回什么的函数。接下来的以 once_control 为参数的 pthread_once 调用不做任何事。无论何时，当你需要动态初始化多个线程共享的全局变量时，pthread_once 函数将很有用的。

### 8. 基于线程的并发服务器

下图展示了基于线程的并发 echo 服务器的代码。整体结构类似于基于进程的设计。主线程不断等待连接请求，然后创建一个对等线程处理该请求。虽然代码看似简单，但是有几个普遍而且有点微妙的问题需要我们看一看。第一个问题是当我们调用 pthread_create 时，如何将已连接描述符传递给对等线程。最明显的方法就是传递一个指向这个描述符的指针，就像下面这样

```c
connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
Pthread_create(&tid, NULL, thread, &connfd);
```

然后，我们让对等线程间接引用这个指针，并将它赋值给一个局部变量，如下所示

```
void *thread(void *vargp) {
	int connfd = *((int *) vargp);
	·
	·
	·
}
```



![image-20240226165132843](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240226165132843.png)

然而，这样可能会出错，因为它在对等线程的赋值语句和主线程的 accept 语句间引入了竞争（race）。如果赋值语句在下一个 accept 之前完成，那么对等线程中的局部变量 connfd 就得到了正确的描述符值。然而，如果赋值语句是在下一个 accept 之后完成的，那么对等线程中的局部变量 connfd 就得到下一次连接的描述符值。那么不幸的结果就是，现在两个线程在同一个描述符上执行输入输出。为了避免这种潜在的致命竞争，我们必须将 accept 返回的每个已连接描述符分配它自己的动态分配内存的内存块，之后会在第七节讲述。

另一个问题是在线程例程中避免内存泄漏。既然不显式地回收线程，就必须分离每个线程，使得在它终止时它的内存资源能够被收回。更进一步，我们必须小心释放主线程分配的内存块。

## 四、多线程程序中的共享变量

从程序员的角度来看，线程很有吸引力的一个方面是多个线程很容易共享相同的程序变量。然而，这种共享也很棘手。为了编写正确的多线程程序，我们必须对所谓的共享以及它是如何工作的有很清楚的了解。

为了理解 C 程序中的一个变量是否是共享的，有一个基本的问题要解答：

- 线程的基础内存模型是什么
- 根据这个模型，变量实例是如何映射到内存的
- 最后，有多少线程引用这些实例

一个变量是共享当且仅当多个线程引用这个变量的某个实例。为了让我们对共享的讨论具体化，我们将使用下图所示的程序作为示例。示例程序由一个创建了两个对等线程的主线程构成。主线程传递一个唯一的 ID 给每个对等线程，每个对等线程利用这个 ID 输出一条个性化信息，以及调用该线程例程的总次数。

![image-20240226171638180](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240226171638180.png)

### 1. 线程内存模型

一组并发程序运行在一个进程的上下文中。每个线程都有它自己独立的线程上下文，包括线程ID、栈、栈指针、程序计数器、条件码和通用目的寄存器值。每个线程和其他线程一起共享进程上下文的剩余部分。这包括整个用户虚拟地址空间，它是由只读文本（代码）、读写数据、堆以及所有的共享库代码和数据区域组成的。线程也共享相同的打开文件的集合。

从实际操作的角度来说，让一个线程去读或者去写另一个线程的寄存器的值是不可能的。另一方面，任何线程都可以访问共享虚拟内存的任何位置。如果某一个线程修改了某个位置，那么其他线程最终都能在它读这个位置时发现这个变化。因此，寄存器从来不是共享的，而虚拟内存总是共享的。

各自独立的线程栈的内存模型不是那么整齐清楚的，这些栈被保存在虚拟地址空间的栈区域中，并且通常是被相应的线程独立地访问的。我们说通常而不是总是，是因为不同的线程栈是不对其他的线程设防的。所以，如果一个线程以某种方式得到一个指向其他线程栈的指针，那么他就可以读写这个栈的任何部分。示例程序在第 26 行展示了这一点，其中对等线程直接通过全局变量 ptr 间接引用主线程的栈的内容。

### 2. 将变量映射到内存

多线程的 C 程序中变量根据他们的存储类型被映射到虚拟内存：

- 全局变量。全局变量是定义在函数之外的变量。在运行时，虚拟内存的读/写区域只包含每个全局变量的一个实例，任何线程都可以引用。例如，第 5 行声明的全局变量 ptr 在虚拟内存的读/写区域中有一个运行的实例。当一个变量只有一个实例时，我们只用变量名（在这里就是 ptr）来表示这个实例。
- 本地自动变量。本地自动变量就是定义在函数内部但是没有 static 属性的变量。在运行时，每个线程的栈都包含它自己的所有本地自动变量的实例。即使多个线程执行同一个线程例程时也是如此。例如，有一个本地变量 tid 的实例，它保存在主线程的栈中。我们用 tid.m 来表示这个实例。再来看一个例子，本地变量 myid 有两个实例，一个在对等线程 0 的栈内，一个在对等线程 1 的栈内，我们将这两个实例分别表示为 myid.p1 和 myid.p0。
- 本地静态变量。本地静态变量是定义在函数内部并有 static 属性的变量。和全局变量一样，虚拟内存的读/写区域只包含在程序中声明的每个本地静态变量中的一个实例。例如，即使示例程序中每个对等线程都在第 25 行声明了 cnt，在运行时，虚拟内存的读/写区域也只有一个 cnt 的实例。每个对等线程都读和写这个实例。

### 3. 共享变量

我们说一个变量 v 是共享的，当且仅当它的一个实例被一个以上的线程引用。例如，示例程序中的变量 cnt 就是共享的，因为它只有一个运行时实例，并且这个实例被两个对等线程引用。然而，认识到像 msgs 这样的本地自动变量也能被共享是很重要的。

## 五、用信号量同步线程

共享变量是十分方便，但是它们也引入了同步错误（synchronnization error）的可能性。考虑下图中的程序 badcnt.c，它创建了两个线程，每个线程都对共享计数变量 cnt 加 1。

![image-20240226214052406](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240226214052406.png)

![image-20240226214120681](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240226214120681.png)

因为每个线程都对计数器增加了 niters 次，我们预计它的最终值是 2 x niters。这看上去简单而直接。然而，当在 Linux 系统上运行 badcnt.c 时，我们不仅得到错误的答案，而且每次得到的答案都还不相同！

> linux> ./badcnt 1000000
>
> BOOM! cnt=1445085
>
> 
>
> linux> ./badcnt 1000000
>
> BOOM! cnt=1915220
>
> 
>
> linux> ./badcnt 1000000
>
> BOOM! cnt=1404746



那么哪里出错了呢？为了清晰地理解这个问题，我们需要研究计数器循环（第40～41）的汇编代码，如下图所示。我们发现，将线程 i 的循环代码分解成五个部分是很有帮助的：

- $H_i$：在循环头部的指令块。
- $L_i$：加载共享变量 cnt 到累加寄存器 %rdx_i 的指令，这里 %rdx_i 是指线程 i 中寄存器 %rdx 的值。
- $U_i$：更新（增加）%rdx_i 的指令
- $S_i$：将 %rdx_i 的更新值存回到共享变量 cnt 的指令
- $T_i$：循环尾部的指令块

注意头和尾只操作本地栈变量，而 L_i、U_i 和 S_i 操作共享计数器变量的内容。当 badcnt.c 中的两个对等线程在一个单处理器上并发运行时，机器指令以某种顺序一个接一个地完成。因此，每个并发执行定义了两个线程中中的指令的某种全序（或者交叉）。不幸的是，这些顺序中的一些将会产生正确结果，但是其他的则不会。

![image-20240227144837711](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227144837711.png)

这里有一个关键点：一般而言，你没有办法预测操作系统是否将为你的线程选择一个正确的顺序。例如，下图展示了一个正确的指令顺序的分步操作。在每个线程更新了共享变量 cnt 之后，它在内存中的值就是 2，这正是期望的值。另一方面。下图 b 的顺序产生一个不正确的 cnt 的值。会发生这样的问题是因为，线程 2 在第 5 步加载 cnt，是在第 2 步线程 1 加载 cnt 之后，而在第 6 步线程 1 存储它的更新值之前。因此，每个线程最终都会存储一个值为 1 的更新后的计数器值。我们能够借助某种进度图（progress graph）的方法来阐明这些正确和不正确的指令顺序的概念。

![image-20240227153916688](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227153916688.png)

### 1. 进度图

进度图将 n 个并发线程的执行模型化为一条 n 维笛卡尔空间中的轨迹线。每条轴 k 对应于线程 k 的进度。每个点（I1，I2，···，In）已经完成了指令 Ik 这一状态。图的原点对应于没有任何线程完成一条指令的初始状态。下图展示了 badcnt.c 程序第一次循环迭代的二维进度图。水平轴对应于线程 1，垂直轴对应于线程 2。点（L1，S2）对应于线程 1 完成了 L1 而线程 2 完成了 S2 的状态。进度图将指令模型化为从一种状态到一种状态的转换。转换被表述为一条从一点到相邻点的有向边。合法的转换是向右（线程 1 中的一条指令完成）或者向上（线程 2 中的一条指令完成）的。两条指令不能在同一时刻完成--对角线转换是不允许的。程序绝不会反向运行，所以向下或者向左的转换也是不合法的。一个程序的执行历史被模型化为状态空间中的一条轨迹线。下图展示了下面指令顺序对应的轨迹线：

![image-20240227163054401](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227163054401.png)

对于线程 i，操作共享变量 cnt 内容的指令（Li，Ui，Si）构成了一个（关于共享变量 cnt 的）临界区，这个临界区不应该和其他进程的临界区交替执行。换句话说，我们想要确保每个线程在执行它的临界区中的指令时，拥有对共享变量的互斥的访问。通常这种现象称为互斥。

在进度图中，两个临界区的交集形成的状态空间区域称为不安全区，下图展示了变量 cnt 的不安全区。注意，不安全区和与它交界的状态相毗邻，但并不包括这些状态。绕过不安全区域的轨迹线叫做安全轨迹线，相反，接触到任何不安全区域的轨迹线叫做不安全轨迹线。

![image-20240227163847777](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227163847777.png)

任何安全轨迹线都将正确的更新共享计数器。为了保证线程化程序示例的正确执行（实际上任何共享全局数据结构的并发程序的正确执行）我们必须以某种方式同步线程，使它们总是有一条安全轨迹线。一个经典的方法就是基于信号量。

### 2. 信号量

Dijkstra，并发编程领域的先锋人物，提出了一种经典的解决同步不同执行线程问题的方法，这种方法是基于一种叫做信号量的特殊类型变量的。信号量 s 是具有非负整数值的全局变量，只能由两种特殊的操作来处理，这种操作称为 P 和 V：

- P(s)：如果 s 是非零的，那么 P 将 s 减 1，并且立即返回。如果 s 为 0，那么就挂起这个线程，直到 s 变为非零，而一个 V 操作后会重启这个线程。重启之后，P 操作将 s 减 1，并将控制返回给调用者。
- V(s)：V 操作将 s 加 1。如果有任何阻塞在 P 操作等待 s 变为非 0，那么 V 操作就会重启这些线程中的一个，然后该线程将 s 减 1，完成它的 P 操作。

P 中的测试和减 1 操作是不可分割的，也就是说，一旦预测信号量 s 变为 非零，就会将 s 减 1，是不可中断的。V 中的加 1 操作也是不可分割的，也就是加载、加 1 和存储信号量的过程没有中断。注意，V 的定义中没有定义等待线程被启动的顺序。唯一的要求是 V 必须只能重启一个正在等待的线程。因此，当有多个线程在等待同一个信号量时，你不能预测 V 操作要重启哪一个线程。

P 和 V 定义确保了一个正在运行的程序绝不可能进入这样一种状态，也就是一个正确初始化的信号量有一个负值。这个属性称为信号量不变性，为控制并发程序的轨迹线提供了强有力的工具。

Posix 标准定义了许多操作信号量的函数：

![image-20240227173055893](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227173055893.png)

sem_init 函数将信号量 sem 初始化为 value。每个信号量在使用前必须初始化。针对我们的目的，中间参数总是 0。程序分别通过调用下面两个函数进行 P 和 V 操作。为了简明，本书使用一下包装函数。

![image-20240227173255345](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227173255345.png)

### 3. 是用信号量实现互斥

信号量提供了一种很方便的方法来确保对共享变量的互斥访问。基本思想是将每个共享变量（或者一组相关的共享变量）与一个信号量 s（初始为 1）联系起来，然后用 P/V 操作将相应的临界区包围起来。以这种方式来保护共享变量的信号量叫做二元信号量，因为它的值总是 0 或 1。以提供互斥为目的的二元信号量常常也称为互斥锁。在一个互斥锁上执行 P 操作称为对互斥锁加锁。类似地，执行 V 操作称为对互斥锁解锁。对一个互斥锁加了锁但是还没有解锁的线程称为占用这个互斥锁。一个被用作一组可用资源的计数器的信号被称为计数信号量。

下图展示了我们如何利用二元信号量来正确地同步计数器程序示例。每个状态都标出了该状态中的信号量 s 的值。关键思想是这种 P 和 V 操作的结合创建了一组状态，叫做禁止区。其中 s<0。因为信号量的不变性，没有实际可行的轨迹能够包含禁止区中的状态。而且禁止区包含了不安全区，所以没有实际可行的轨迹线能够接触不安全区域的任何部分，因此每条实际可行的轨迹线都是安全的，而且不管运行时指令顺序是怎样的，程序都会正确的增加计数器值。

![image-20240227175011858](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227175011858.png)

从可操作的意义上来说，由 P 和 V 操作创建的禁止区使得在任何时间点上，在被包围的临界区中，不可能有多个线程在执行指令。换句话说，信号量操作确保了对临界区的互斥访问。总的来说，为了用信号量正确同步上图的计数器程序示例，我们首先声明一个信号量 mutex：

```c
volatile long cnt = 0;
sem_t mutex;
```

然后在主例程中，将 mutex 初始化为 1：

```c
Sem_init(&mutex, 0, 1);
```

最后，我们通过把在线程例程中对共享变量 cnt 的更新包围 P 和 V 操作，从而保护它们：

```c
for(i = 0; i<niters; i++) {
  P(&mutex);
  cnt++;
  V(&mutex);
}
```

当我们运行这个正确同步的程序时，现在它每次都能产生正确结果。

### 4. 利用信号量来调度共享资源

除了提供互斥之外，信号量的另一个重要作用是调度对共享资源的访问。这种场景中，一个线程用信号量操作来通知另一个线程，程序状态的某个条件已经为真了。两个经典的例子是生产者-消费者和读者-写者名问题。

#### (1) 生产者-消费者问题

下图给出了生产者-消费者问题。生产者和消费者线程共享一个有 n 个槽的有限缓冲区。生产者线程反复地生成新的项目，并把它们插入缓冲区中。消费者线程不断地从缓冲区中取出这些项目，然后消费（使用）它们。也可能有多个生产者和消费者的变种。

![image-20240227220323927](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227220323927.png)

#### （2）读者-写者问题
读者-写者问题是互斥问题的一个概括。一组并发的线程要访问一个共享对象，例如一个主存上的数据结构，或者一个磁盘上的数据库。有些线程只读对象，而其他的线程只修改对象。修改对象的线程叫做写者，只读对象的线程我们叫做读者。写者必须拥有对对象独占的访问，而读者可以和无限多个其他的读者共享这个对象。一般来说，有无限多个读者和写者。

### 5. 基于预线程化的并发服务器

我们已经知道了如何使用信号量来访问共享变量和调度对共享资源的访问。为了帮助你更清晰地理解这些思想，让我们把它们应用到一个基于称为预线程化技术的并发服务器上。在如下图所示的并发服务器中，我们为每一个新客户端创建了一个新线程。这种方法的缺点是我们为每一个新客户创建一个新线程，导致不小的代价。一个基于预线程化的服务器试图通过使用下图所示的生产者-消费者模型来降低这种开销。服务器是由一个主线程和一组工作者线程构成的。主线程不断地接收来自客户端的连接请求，并将得到的连接描述符放在一个有限缓冲区中。每一个工作者线程反复地从共享缓冲区中取出描述符，为客户端服务，然后等待下一个描述符。

![image-20240227230133443](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240227230133443.png)

## 六、使用线程提高并发性

到目前为止，在对并发的研究中，我们都假设并发线程是在单处理器系统上执行的。然而，大多数现代机器具有多核处理器。并发程序通常在这样的机器上运行得更快，因为操作系统内核在多个核上并行地调度这些并发程序，而不是在单个核上顺序地调度。在像繁忙的 web 服务器、数据库服务和大型科学计算代码这样的应用中利用这样的并行性是至关重要的。

## 七、其他并发问题

同步从根本上来说是很难得问题，它引出了在普通的顺序程序中不会出现的额问题。

### 1. 线程安全

当用线程编写程序时，必须小心地编写那些具有称为线程安全性属性的函数。一个函数被称为线程安全的，当且仅当被多个并发线程反复的调用时，他就一直产生正确的结果。如果一个函数不是线程安全的，我们就说它是线程不安全的。我们能够定义出四个线程不安全函数类：

- 第 1 类：不保护共享变量的函数。我们在图 12-16 的 thread 函数中就已经遇到了这样的问题，该函数对一个未受保护的全局计数器变量加 1。将这类线程不安全函数编程安全的，相对而言比较容易：利用像 P 和 V 操作这样的同步操作来保护共享的变量。这个方法的优点是在调用程序中不需要做任何修改。缺点是同步操作将减慢程序的执行时间。

- 第 2 类：保持跨越多个调用的状态的函数。一个伪随机数生成器是这类线程不安全函数的简单例子。参考下图的伪随机数生成器程序包。rand 函数是线程不安全的，因为当前调用的结果依赖于前次调用的中间结果。当调用 scrand 为 rand 设置了一个种子后，我们从一个单线程中反复地调用 rand，能够预期得到一个可重复的随机数字序列。然而如果都线程调用 rand 函数，这种假设就不再成立。

    ![image-20240228153723031](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240228153723031.png)

    使得像 rand 这样的函数线程安全的唯一方式就是重写它，使得它不再使用任何 static 数据，而是依靠调用者在参数中传递状态信息。这样做的缺点是，程序员现在还要被迫修改调用程序中的代码。在一个大的程序中，可能有成百上千不同的调用位置，做这样的修改将是非常麻烦的，而且容易出错。

- 第 3 类：返回指向静态变量的指针的函数。某些函数，例如 ctime 和 gethostname，将计算结果放在一个 static 变量中，然后返回一个指向这个变量的指针。如果我们从并发线程中调用这些函数，那么将可能发生灾难，因为正在被一个线程使用的结果会被另一个线程悄悄覆盖了。

    有两种方法来处理这类线程不安全函数。一种选择是重写函数，使得调用者传递存在结果的变量的地址。这就消除了所有的共享数据。但是它要求程序员能修改源代码。如果不行，则另一种选择就是加锁-复制技术。基本思想是将线程不安全函数与互斥锁联系起来。在每一个调用位置，对互斥锁加锁，调用线程不安全函数，将函数返回的结果复制到一个私有的内存位置，然后对互斥锁解锁。为了尽可能减少对调用者的修改，你应该定义一个线程安全的包装函数来取代所有对线程不安全函数的调用。

- 第 4 类：调用线程不安全函数的函数。如果函数 f 调用线程不安全函数 g，如果 g 是第 2 类函数，即依赖于跨越多次调用的状态，那么 f 也是线程不安全的，而且除了重写 g 以外，没有什么方法。然而，如果 g 是第 1 类或者第 3 类函数，那么你只要用一个互斥锁保护调用位置和任何得到的共享数据，f 仍然可能是线程安全的。

### 2. 可重入性

有一类重要的线程安全函数，叫做可重入函数。其特点在于它们具有这样一种属性：当它们被多个线程调用时，不会引用任何共享数据。尽管线程安全和可重入函数有时候会被不正确的视为同义词，但是他们之间还是有清晰的技术差别。如下图所示：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240228163549160.png" alt="image-20240228163549160" style="zoom:50%;" />

可重入函数一般比不可重入的线程安全的函数高效一些，因为它们不需要同步操作。更进一步来说，将第 2 类线程不安全函数转化为线程安全函数的唯一方法就是重写它，使之变成可重入的。如下图所示：

![image-20240228163811990](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240228163811990.png)

### 3. 在线程化的程序中使用已存在的库函数

大多数 Linux 函数包括定义在标准 C 库中的函数（例如 malloc、free、realloc、printf 和 scanf）都是线程安全的，只有一小部分是例外。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240228164202484.png" alt="image-20240228164202484" style="zoom:50%;" />

Linux 系统提供大多数线程不安全函数的可重入版本。可重入版本的名字总是以 “_r” 后缀结尾。

### 4. 竞争

当一个程序的正确性依赖于另一个线程到达 y 点之前到达它控制流中的 x 点时，就会产生竞争。通常发生竞争是因为程序员假设线程将按照某种特殊的轨迹线穿过执行状态空间，而忘记了另一条准则规定：多线程的程序必须对任何可行的轨迹线都正确工作。

### 5. 死锁

信号量引入了一种潜在的令人厌恶的运行时错误，叫做死锁，他指的是一组线程被阻塞了，等待一个永远也不会为真的条件。进度图对于理解死锁是一个无价的工具。下图展示了一对用两个信号量来实现互斥的线程的进度图。从这幅图中，我们可以得到关于死锁的一些重要知识：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240228170515176.png" alt="image-20240228170515176" style="zoom:50%;" />

- 程序员使用 P 和 V 操作顺序不当，以至于两个信号量的禁止区域重叠。如果某个执行轨迹线碰巧达到了死锁状态 d，那么就不可能有进一步的进展了，因为重叠的禁止区域阻塞了每个合法方向上的进展。
- 重叠的禁止区域引起一组称为死锁区域的状态。如果一条轨迹线碰巧到达了一个死锁区域的状态，那么死锁就是不可避免的了，轨迹线可以进入死锁区域，但是他们不可能离开。
- 死锁是一个相当困难的问题，因为它不总是可预测的。一些幸运的执行轨迹线将绕开死锁区域，而其他则会陷入这个区域，上图展示了每种情况的一个示例。

死锁，最糟糕的是错误常常是不可重复的，因为不同的执行有不同的轨迹线。程序死锁的原因有很多，要避免死锁一般而言很困难。然而使用二元信号量来实现互斥时，可以使用下面简单有效的规则来避免死锁：

> 互斥锁加锁顺序规则：给定所有互斥操作的一个全序，如果每个线程都是以一种顺序获得互斥锁，并以相反的顺序释放，那么这个程序就是无死锁的。

例如，可以这样解决上面的死锁问题：在每个线程中先对 s 加锁，然后再对 t 加锁。下图展示进度图：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240228171803961.png" alt="image-20240228171803961" style="zoom:50%;" />

