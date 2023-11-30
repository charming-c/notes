# 系统级 I/O

## 一、 UNIX I/O

一个 Linux 文件就是一个 m 个字节的序列：
$$
B_0, B_1, ···,B_k, ···, B_{m-1}
$$
所有的 I/O 设备（例如网络、磁盘和终端等）都被模型化为文件，而所有的输入和输出都被当作对相应文件的读和写来执行。这种将设备优雅的映射为文件的方式，允许 Linux 内核引出一个简单的、低级的应用接口，称为 Unix I/O。这使得所有的输入和输出都能用一种统一且一致的方式来执行：

- 打开文件：一个应用程序通过要求内核打开相应的文件，来宣告他想要访问一个 I/O 设备。内核返回一个小的非负整数，叫做**描述符**。它在后续对此文件的所有操作中标识这个文件。内核记录有关这个打开文件的所有信息。
- Linux shell 创建的每个进程开始时都有三个打开的文件：（头文件 <unistd.h> 定义了常量来代替显式的描述符）
    - 标准输入（描述符为0）：STDIN_FILENO
    - 标准输出（描述符为1）：STDOUT_FILENO
    - 标准错误（描述符为2）：STDERR_FILENO
- 改变当前文件的位置：对于打开的文件，内核中保持着一个文件位置 k，初始为 0。这个文件位置是从文件开头起始的字节偏移量。应用程序可以通过执行 seek 操作，显式地设置文件当前位置为 k。
- 读写文件：一个读操作就是从文件中复制 n（n>0）个字节到内存中，从当前文件位置 k 开始，然后从 k 开始到 k + n。给定一个大小为 m 的字节的文件，当 k >= m，执行读操作时会触发一个称为 EOF 的条件，应用程序能识别到这个条件，在文件结尾处并没有明确的 EOF 条件。
    - 类似的，写操作就是从内存复制 n > 0 个字节到文件，从当前位置 k 开始，然后更新 k。
- 关闭文件：当应用完成了对文件的访问之后，他就会通知内核关闭这个文件。作为响应，内核释放文件打开时创建的数据结构，并将这个描述符放入可用的描述符池中。无论一个进程因何原因终止了，内核都会关闭所打开的文件，并释放他们的内存资源。

## 二、文件

每个 Linux 文件都有一个类型来表明它在系统中的角色：

- 普通文件（regular file）包含任意数据。应用程序常常要区分文本文件（text file）和二进制文件（binary file），文本文件是只含有 ASCII 或 Unicode 字符的普通文件；二进制文件是所有其他的文件。对于内核而言，二者没有区别。
    - Linux 文本文件包含了一个文本行（text line）序列，其中每一行都是一个字符序列，以一个新行符（"\n"）结束，数字值为 0x0a（LR）
- 目录（directory）是包含一组链接（link）的文件，其中每个链接都将一个文件名映射到一个文件，这个文件可能是另一个目录。每个人目录至少含有两个条目：
    - "."：该目录自身的链接。
    - ".."：到目录层次结构中父目录的链接。
- 套接字（socket）是用来与另一个进程进行跨网络通信的文件。

其他文件类型暂不讨论。

## 三、打开和关闭文件

进程通过调用 open 函数来打开一个已经存在的文件或者创建一个新文件的：

```c
int open(char* filename, int flags, mode_t mode);
// 返回：若成功则返回新文件描述符，若出错为 -1。
```

open 函数将 filename 转换为一个文件描述符，并且返回描述符数字。返回的描述符总是在进程中当前**没有打开的最小描述符**。**flags 参数指明了进程如何访问这个文件**：

- O_RDONLY：只读。
- O_WRONLY：只写。
- O_RDWR：可读可写。

flags 参数也可以是一个或者跟多位掩码的或，为写提供一些额外的指示：

- O_CREAT：如果文件不存在，就创建它的一个截断的（truncated）（空）文件。
- O_TRUNC：如果文件已经存在，就截断它。
- O_APPEND：在每次写操作前，设置文件位置到文件的结尾处。

**mode 参数指定了新文件的访问权限位**。这些位的符号名字如下图所示：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231031235815334.png" alt="image-20231031235815334" style="zoom:50%;" />

作为上下文的一部分，每个进程都有一个 umask，它是通过调用 umask 函数来设置的，当进程通过带某个 mode 的参数的 open 函数调用来创建一个新文件时，文件的访问权限位被设置成为 **mode & ~umask**。例如：

```c
#define DEF_MODE S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH
#define DEF_UMASK S_IWGRP|S_IWOTH

umask(DEF_UMASK);
fd = Open("foo.txt", O_CREAT|O_TRUNC|O_WRONLY, DEF_MODE);
```

上面代码创建一个新文件，文件的拥有者有读写权限，而其他所有用户都有读权限。

进程通过调用 close 函数关闭一个打开的文件。

```c
int close(int fd);
// 若成功则为 0，若出错则为 -1。
```

> 练习题：
>
> ```c
> int main() {
>     int fd1, fd2;
> 
>     fd1 = Open("foo.txt", O_RDONLY, 0);
>     Close(fd1);
>     fd2 = Open("baz.txt", O_RDONLY, 0);
>     printf("fd2 = %d\n", fd1);
>     exit(0);
> }
> ```
>
> 输出：
>
> Fd2 = 3
>
> 因为每个进程在创建时已经打开了三个文件，分别是0，1，2

## 四、读和写文件

应用程序调用 write 和 read 函数来执行输入和输出。

```c
ssize_t read(int fd, void* buf, size_t n);
// 返回：若成功则为读取的字节数，遇到 EOF 则为 0，出错则为 -1。
ssize_t write(int fd, void* buf, size_t n);
// 返回：若成功则为写入的字节数，出错则为 -1。
```

​	read 函数从描述符为 fd 的当前文件位置**最多复制** n 个字节到内存位置 buf。返回值 -1表示一个错误，而返回值 0，表示 EOF。否则，返回值表示的是实际传送的字节数。

​	write 函数从内存位置 buf 复制至多 n 个字节到描述符 fd 的当前文件位置。通过调用 lseek 函数，应用程序能够显式的修改当前文件的位置，暂不介绍。

> size_t 与 ssize_t 的区别：
>
> size_t 被定义为 unsigned long，而 ssize_t 被定义为 long，因为出错时会返回 -1.

在某些情况下，read 和 write 函数所传送的字节数比应用程序要求的字节数要少，这些不足值（short count）不表示错误，出现的原因有：

- 读时遇到 EOF。假设准备读一个只有 20 个字节的文件，但是以 50 个字节的片来读取，这样一来，下一个 read 返回不足值 20，此后的 read 会通过返回不足值 0 来发送 EOF 信号。
- 从终端读取文本行。如果打开文件是与终端相关联的（如显示器和键盘），那么每个 read 函数将一次传送一个文本行，返回的不足值等于文本行的大小（不包括换行符）。
- 读和写网络套接字（socket）。如果打开的文件对应于网络套接字，那么内部缓冲约束和较长的网络延迟会引起 read 和 write 返回不足值。对 Linux 管道（pipe）调用 read 和 write 时，也可能出现不足值。

## 五、用 RIO 包健壮地读写

RIO 包提供了方便、健壮和高效的 I/O。RIO提供了两类不同的函数：

- 无缓冲的输入输出函数。这些函数直接在内存和文件直接传送数据，没有应用级缓冲。他们对将二进制数据读写到网络和从网络中读写二进制数据尤其有用。
- 带缓冲的输入函数。这些函数允许你高效地从文件读取文本行和二进制数据，这些文件的内容缓冲到应用级缓冲区内，类似于 printf 这样的标准 I/O 函数提供的缓冲区。同时，带缓冲的 RIO 输入函数是线程安全的，他在同一个描述符上可以被交错地调用。例如，你可以从一个描述符读取一些文本行，然后读取一些二进制数据，接着再多读一些文本行。

### 1. RIO 的无缓冲的输入输出函数

通过调用 rio_readn 和 rio_writen 函数，应用程序可以在内存和文件之间直接传送数据。

```c
ssize_t rio_readn(int fd, void *usrbuf, size_t n);
ssize_t rio_writen(int fd, void *usrbuf, size_t n);
// 若成功则为传送的字节，若 EOF（仅对于 rio_readn而言）返回 0， 出错则为 -1。
```

注意，如果 rio_readn 和 rio_writen 函数被一个从应用信号处理函数的返回中断，那么每个函数会重启 read 和 write 函数。

```c
ssize_t rio_readn(int fd, void* usrbuf, size_t n) {
	size_t nleft = n;	// 还剩下要读的字节数
	ssize_t nread;	// 一次读取所读取的字符串
	char *bufp = usrbuf;	// 读取缓冲区后，剩下缓冲区空余的头指针
	
  // 如果还有字节数没读完，继续进入循环读取
	while(nleft > 0) {
    // 读取出错
		if((nread = read(fd, bufp, nleft)) < 0) {
      // 因为信号而导致读取中断
      // 则此次读取失效，读取到 0 个字节，下次继续读取
      if(errno == EINTR)
        nread = 0;
      // 其他问题导致读取出错
      else 
        return -1;
    }
    // 若正常读读到 0 个字节，则是遇到 EOF
    else if(nread == 0) 
      break;
    // 读取到一定字节以后，所剩下的字节 nleft 减去 nread
    // 缓冲区后移，下次缓冲区空余的位置为 bufp += nread
    nleft -= nread;
    bufp += nread;
	}
  return (n - nleft);
}
```

```c
ssize_t rio_writen(int fd, void* usrbuf, size_t n) {
  size_t nleft = n;
  size_t nwrite;
  char* bufp = usrbuf;

  while(nleft > 0) {
    if((nwrite = write(fd, bufp, nleft)) < 0) {
      if(errno == EINTR)
        nwrite = 0;
      else
        return -1;
    }
    // 输出的函数 write 不会返回 0，所以不用处理
    nleft -= nwrite;
    bufp += nwrite;
  }
  return (n - nleft);
}
```



### 2. RIO 的带缓冲的输入函数

假设要编写一个程序来计算文本文件中文本行的数量，一种方法就是用 read 函数来一次一个字节地从文件传送进用户内存，检查每个字节来查找换行符，这个方法的缺点是效率不是很高，每读取文件中的一个字节都要求陷入内核。一种更好的方式就是引入缓冲区。利用包装函数（rio_readlineb），他从一个内部缓冲区复制一个文本行，当缓冲区变空时，会自动调用 read 重新填满缓冲区。对于既包含文本行也包含二进制数据的文件，我们也提供了一个 rio_readn 带缓冲区的版本，叫做 rio_readnb，它从和 rio_readlineb 一样的读缓冲区中传送字节。

```c
void rio_readinitb(rio_t *rp, int fd);
// 无返回值
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen);
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n);
// 返回：如成功则为读的字节数，若 EOF 则为 0，若出错则为 -1。
```

每打开一个描述符，都会调用一次 rio_readinitb 函数。它将 fd 和一个地址 rp 处的一个类型为 rio_t 的读缓冲区联系起来。

```c
#define RIO_BUFSIZE 8192
typedef struct {
	int rio_fd;	// 此缓冲区的描述符
	int rio_cnt;	// 缓冲区缓存的数据大小
	char *rio_bufptr;	// 缓冲区下一个可寸字节的地址
	char rio_buf[RIO_BUFSIZE];	// 整个缓冲区
}

void rio_readinitb(rio_t *rp, int fd) {
	rp->rio_fd = fd;
	rp->rio_cnt = 0;
	rp->rio_bufptr = rp->rio_buf; 
}
```

rio_readlineb 函数从文件 rp 读出下一个文本行（包括结尾的换行符），将它复制到内存位置 usrbuf，并且用 NULL（零）字符来结束这个文本行。rio_readlineb 函数最多读 maxlen - 1 个字节，余下的一个字符留给结尾的 NULL 字符。超过 maxlen - 1 的文本行被截断，并用一个 NULL 字符结束。

```c
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n) {
	int cnt;
	while(rp->rio_cnt <= 0) {
		rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, sizeof(rp->rio_buf));
		if(rp->rio_cnt < 0) {
			if(errno != EINTR)
				return -1;
		}
		else if(rp->rio_cnt == 0)
			return 0;
		else 
			rp->rio_bufptr = rp->rio_buf;
	}
	
	cnt = n;
	if(rp->rio_cnt < n)
		cnt = rp->rio_cnt;
	memcpy(usrbuf, rp->rio_buf, cnt);
	rp->rio_bufptr += cnt;
	rp->cnt -= cnt;
	return cnt;
}
```



```c
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen) {
  int n, rc;
  char c, *bufp = usrbuf;
  for(n = 1; n < maxlen; n++) {
    // 每次读取一个字符
    if((rc = rio_read(rp, &c, 1)) == 1) {
      // 将字符加入缓冲区
      *bufp++ = c;
      // 如果读到 \n 则直接跳出循环，读取到一行了
      if(c == '\n') {
        n++;
        break;
      }
    }
    // 如果读一个字符失败
    else if(rc == 0) {
      // 刚读就读到 EOF
      if(n == 1) 
        return 0;
      // 读取到一半，读到 EOF
      else break;
    }
    // 出错
    else return -1;
  }
  *bufp = 0;
  return n-1;
}
```



rio_readnb 函数从文件 rp 最多读 n 个字节到内存位置 usrbuf。对同一描述符，对 rio_readlineb 和 rio_readnb的调用可以交叉使用，然后带缓冲的不应该和不带缓冲的交叉使用。

```c
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n) {
  size_t nleft = n;
  ssize_t nread;
  char *bufp = usrbuf;
  
  while(nleft > 0) {
    if((nread = rio_read(rp, bufp, nleft)) < 0) 
      return -1;
    else if(nread = 0)
      break;
    nleft -= nread;
    bufp += nread;
  }
  return (n - nleft);
}
```

## 六、读取文件元数据

应用程序能够调用 stat 和 fstat 函数，检索到关于文件的信息（有时也称为文件的元数据）。

```c
int stat(const char *filename, struct stat *buf);
int fstat(int fd, struct stat *buf);
// 返回：成功则为 0，若出错则为 -1。
```

stat 函数以一个文件名为输入，填写如下图所示的一个 stat 数据结构中的各种成员。fstat 函数是相似的，只不过以文件描述而不是文件名作为输入，在讨论 Web 服务器时，会需要 stat 数据结构中的 st_mode 和 st_size 成员。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231103104203612.png" alt="image-20231103104203612" style="zoom:50%;" />

st_size 成员包含了文件的字节数大小。st_mode 成员则编码了文件访问许可位和文件类型。Linux 在 sys/stat.h 中定义了宏谓词来确定 st_mode 成员的文件类型：

- S_ISREG(m)：是一个普通文件吗？
- S_ISDIR(m)：是一个目录吗？
- S_ISSOCKET(m)：是一个网络套接字吗？

## 七、读取目录内容

应用程序可以用 readdir 系列函数来读取目录内容。

```c
DIR *opendir(const char *name);
// 若成功，则为处理的指针，若出错则为 NULL
```

函数 opendir 以路径名为参数，返回指向目录流的指针。流是对条目有序列表的抽象，在这里指目录项的列表。

```c
struct dirent *readdir(DIR *dirp);
// 返回：若成功，则为指向下一个目录项的指针；若没有更多的目录项或出错，则为 NULL
```

每次对 readdir 的调用返回的都是指向流 dirp 中的下一个目录项的指针，或者，如果没有更多的目录项则返回 NULL。每个目录项都有一个结构，其形式如下：

```c
struct dirent {
		ino_t d_nio; // 文件位置
		char d_name[256];	// 文件名
}

int closedir(DIR *dirp);
```

如果出错，则 readdir 返回 NULL，并设置 errno。可惜的是，唯一能区分错误和流结束情况的方法是检查自调用 readdir 以来 errno 是否被修改过。

函数 closedir 关闭流，并释放其所有的资源。

## 八、共享文件

可以用许多不同的方式来共享 Linux 文件。除非你很清楚内核是如何表示打开的文件，否则文件共享的概念相当难懂。内核用三个相关的数据结构来表示打开的文件：

- **描述符表**。每个进程都有它独立的描述符表，它的表项是由进程打开的文件描述符来索引的。每个打开的描述符表项指向文件表中的一个表项。
- **文件表**。打开文件的集合是由一张文件表来表示的，所有进程共享这张表。每个文件表的表项组成包括当前文件位置、引用计数（即当前指向该表项的描述符表项数），以及一个指向 v-node 表中对应表项的指针。关闭一个描述符会减少相应的文件表表项中的引用计数。内核不会删除这个文件表表项，直到它的引用计数为 0。
- **v-node 表**。同文件表一样，所有进程共享这张表。每个表项包含 stat 结构的大多数信息，包括 st_size 和 st_mode 成员。

如图，其中描述符 1 和 4 通过不同的打开文件表表项来引用两个不同的文件。这是一种典型的情况，没有共享文件，并且每个描述符对应一个不同的文件。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231103114314555.png" alt="image-20231103114314555" style="zoom:50%;" />

多个描述符也可以通过不同的文件表项来引用同一个文件。例如，如果以同一个 filename 调用 open 函数两次，就会发生这种情况，关键思想是每个描述符都有它自己的文件位置，所以对于不同描述符的读操作可以从文件的不同位置获取数据。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231103114552262.png" alt="image-20231103114552262" style="zoom:50%;" />

我们也可以理解父子进程是如何共享文件的。假设在调用 fork 以前，父进程有 图10-12 的打开文件。然后调用了 fork 之后，如下图所示，子进程有父进程描述符表的副本。父子进程共享相同的打开文件表的集合，因此共享相同的文件位置。一个很重要的结果是，在内核删除对应文件表表项之前，父子进程必须都关闭了它们的描述符。

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231103114926664.png" alt="image-20231103114926664" style="zoom:50%;" />

## 九、I/O 重定向

Linux shell 提供了 I/O 重定向操作符，允许用户将磁盘文件和标准输入输出联系起来。例如，键入：

> linux> ls > foo.txt

使得 shell 加载和执行 ls 程序，将标准输出重定向到磁盘文件 foo.txt。就如当一个 Web 服务器代表客户端运行 GCI 程序时，他就执行一种相似类型的重定向。I/O 重定向是如何工作的？一种方式时使用 dup2 函数。

```c
int dup2(int oldfd, int newfd);
// 返回：若成功则为非负的描述符，若出错则为 -1。
```

dup2 函数复制描述符表表项 oldfd 到描述符表项 newfd，覆盖描述符表项newfd 以前的内容。如果 newfd 已经打开， dup2会在复制 oldfd 之前关闭 newfd。例如，对于 图10-12 的描述符指向，调用 dup2(4,1) 以后，其结果如图所示：

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231103123315550.png" alt="image-20231103123315550" style="zoom:50%;" />

## 十、标准 I/O

<img src="https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231103124701928.png" alt="image-20231103124701928" style="zoom:50%;" />

标准 I/O 在某种意义上是双全工的，因为程序可以在一个流上进行执行输入和输出。然而，对流的限制和对套接字的限制，有时会互相冲突：

- 限制一：跟在输出函数之后的输入函数。如果中间没有插入对 fflush、fseek、fsetpos 或者 rewind 的调用，一个输入函数不可以跟在一个输出函数之后， fflush 函数清空与流相关的缓冲区，后三个函数根据 UNIX I/O 的 lseek 函数来重置当前文件的位置。
- 限制二：跟在输入函数之后的输出函数。如果中间没有插入对 fseek、fsetpos 或者 rewind 的调用，一个输出函数不能跟在一个输入函数之后，除非该输入函数遇到了一个文件结束。

这些限制给网络应用带来了一个问题，因为对套接字使用 lseek 函数是非法的。对于第一个限制能够采用在每次输入操作之前刷新缓冲区这样的规则来满足，但满足第二个限制的唯一办法是，对同一个打开的套接字描述符打开两个流，一个用来读一个用来写。但是这种方法也有问题，因为它要求应用程序在两个流上都要调用 fclose，这样才能释放每个流相关的内存资源，避免内存泄漏。

这些操作中的每一个都试图关闭同一个底层的套接字描述符，所以第二个 close 操作就会失败，对于顺序的程序不是问题，对于线程化的程序来说是灾难的，所以对于网络套接字上不要使用标准 I/O 函数来进行输入和输出，而使用健壮的 RIO 函数包。