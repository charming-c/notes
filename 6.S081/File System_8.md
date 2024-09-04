# File system

​	文件系统的目的是为了组织和存储数据。文件系统主要支持在用户和应用之间共享数据，同时还有持久化，以至于在重启之后数据仍然是可用的。

​	xv 的文件系统提供类 Unix 的文件、目录和路径名，并且保存这些数据到虚拟磁盘以供持久化。文件系统提供了这些挑战：

- 文件系统需要磁盘上的数据结构，去代表目录和文件的树结构，去记录 blocks 的有效性（也就是持有每个文件的内容），去记录磁盘中哪些空间是空闲的。
- 文件系统必须支持故障恢复。也就是，如果一个故障（例如断电）发生了，文件系统必须在重启之后还可以正确的工作。风险就是一次故障可能会中断一连串的更新并且保留不一致的磁盘数据结构。（例如，一个 block 同时被使用，且被标记为 free）。
- 不同的进程可能会同时在文件系统上操作，所以文件系统代码必须处理去维护一下不变量。
- 访问磁盘要比访问内存慢几个数量级，因此文件系统必须在内存中维护常用块的缓存。

接下来介绍 xv6 如何处理这些问题的。

## 一、总览

​	文件系统由如下的七层结构组成。disk 层读和写 block 在 virtio 硬件驱动上。buffer cache 层缓存 disk 的 blocks 并且同步的访问它们，确保在同一个时间只有一个内核进程可以修改在某个特定 block 中的数据。logging 允许更高层将更新包装到事务中的几个块，并确保在面对崩溃时自动更新 blocks(即，所有块都更新或没有更新)。inode 层提供相互独立的文件，每个文件由一个 inode （通过一个独特的 i-number 表示）和保存文件数据的 blocks 表示。directory 层实现了将每个目录看成是一种特殊种类的 inode（这种 inode 是一个 directory 条目的序列，每个都包含一个文件名和一个 i-number）。pathname 层提供结构化的路径名比如（x/y/z），并且通过递归查找来处理它们。file descriptor 层抽象出许多 Unix 资源（比如 pipes，devices，files 等等），通过文件系统的接口，简化了应用程序员的工作。

<img src="/Users/charming/Library/Application Support/typora-user-images/image-20240830215527106.png" alt="image-20240830215527106" style="zoom:50%;" />

​	磁盘（disk）硬件通常将磁盘中的数据表示成一个 512 字节编码的 block 序列（也被称为 sector 扇区）。secotr 0 是最开始的 512 字节，sector 是第二个，等等。操作系统为文件系统使用的 block size 可能和磁盘使用的不一样，不过一般是 sector 的整数倍。xv6 通过 struct buf 结构体类型持有从内存中读取的 bolcks 的副本。在这个结构体中保存的数据有时候和磁盘不同步：buf 可能还没有从磁盘读到数据（disk 正在使用它，并且还有返回对应扇区的内容），也可能是 buf 被软件改写了但是没有写回进磁盘。

​	文件系统必须要有一个规划，就是到底在哪里保存 inode 以及磁盘中的 blocks。为了实现它，xv6 将 disk 分成了几个 section，如下图所示：

![image-20240830220520497](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20240830220520497.png)

文件系统不使用 block0（它持有 boot sector），block1 称为 superblock（超级块），它保存了文件系统的元数据（metadata）（文件系统中的 blocks 的大小，数据 blocks 的数量，log 中的 blocks 的数量）。从 block2 开始持有 log，log 后面是 inodes，每个 block 都有很多个 inodes。后面是 bitmap block（哪些 data blocks 正在使用）。剩下的所有 blocks 就是存储数据的了，要么是在 bitmap 中被标记成 free，或者持有文件或者目录的内容。超较块被一个单独的程序填写，叫做 mkfs，它创建一个初始化后的文件系统。下面介绍文件系统的每一层，注意每一个低层的 layer 都如何简化了高层的 layer 抽象。

## 二、Buffer cache layer

​	buffer cache 层主要有两项工作：

1. 同步访问磁盘 bolcks，确保只有一个 blocks 的副本拷贝在内存中并且在同一时间只有一个内核线程可以使用这个副本。
2. 缓存最常被访问的 blocks，以至于不需要重复的从磁盘中缓慢的读取到内存中。

​	buffer cache 暴露的接口重要有两个：bread 和 bwrite。前者获取一个包含可在内存中读取或修改的 bolck 副本的 buf，后者将修改后的 buffer 写入磁盘上相应的block。内核线程必须通过 brelse 释放一个 buffer，当对这个 buffer 的操作完成之后。buffer cache 对每个 buffer 使用一个 sleep-lock 确保一次只有一个线程使用每个 buffer（也就是磁盘 block），bread 返回一个上锁的 buffer， brelse 释放这个锁。

​	buffer cache 有一个确定的发小缓存磁盘的 blocks，也就意味着如果文件系统请求一个不在缓冲区的 block，buffer cache 就必须回收一个 之前已经缓存其他 block 的 buffer。buffer cache 使用 LRU 回收算法。因为我们预设最近使用的 buffer 很可能很快又会被使用（CPU 的时间局部性）。

**xv6 代码解读**

数据结构设置：

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;

struct buf {
  int valid;   // has data been read from disk?
  int disk;    // does disk "own" buf?
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE];
};
```

buffer cache 主要分成两个方面：

- bcache：包含整个 cache 的 buffer 的集合，持有一个 bcache 的大锁，一个 buf 数组，和一个 buf 的双向链表（用于 LRU）
- buf：包含有效位 valid 标识一个 block 的信息，也就是 data，还有一个 refcnt 表示被引用的次数，lock 对 buf 的引用上锁，以及双向链表的结构。

bread 和 bwrite：

```c
// Look through buffer cache for block on device dev.
// If not found, allocate a buffer.
// In either case, return locked buffer.
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}

// Return a locked buf with the contents of the indicated block.
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}

// Write b's contents to disk.  Must be locked.
void
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  virtio_disk_rw(b, 1);
}
```

在 bread 时会先检查（此时的访问是要上整个 bcache 锁的，因为可能会有两个线程同时检查，同时调用 bget，从而就会缓存同一个 block在相同的地方，refcnt 出错）是否 cache 了这个 block，没有的话，就 LRU 查找一个替换。之后调用 bwrite 对这个 buf 写，真正的读写在 virtio_disk_rw 中实现。

这里 LRU 的实现是：

- 每次释放时，将释放的 buf 插入链表头
- 每次在 bcache 中查找 buf 时，从 head 开始查找，这样保证之前释放的总能被优先使用，方便重新利用
- 在查找一圈需要驱逐的时候就倒序寻找 head，往前查，因为最近被使用的肯定在之前就被 brelse 放到前面去了

## 三、Logging layer

​	在文件系统的设计中一个最有趣的问题就是故障恢复（crash recovery）。问题的出现是因为许多文件系统的操作涉及到许多对磁盘的写入，并且在一个写入的子集操作中发生故障，会导致在磁盘中的文件系统数据结构陷入不可持续的状态。例如，假设故障发生在对文件的 truncation（截断：将文件的长度设置为 0，并且释放它的块的内容）时。取决于磁盘写入操作的顺序，故障要么导致一个 inode 指向一个被标记为 free 的内容块，要么导致一个被分配了但是没有引用指向他的内容块。

​	后者故障是良性的，但是一个指向 free 过的 block 的 inode 很可能在重启后会导致严重的问题。在重启之后，内核可能会给另一个文件分配那个 block，并且现在我们有两个文件无意中指向同一个 block。如果 xv6 支持多用户，这种情形会导致安全问题，因为前一个文件的所有者将可以新的文件（属于另一个用户）中读写 blocks。

​	xv6 解决文件系统操作中故障问题采用一种简单形式的 logging。一个 xv6 系统调用并不会直接对磁盘上的文件系统数据结构进行写。而是，它将所有的想要进行的磁盘写操作设成一个磁盘上的 log。一旦系统调用已经全部记录了他所有的写操作，它就会为磁盘写一个特别的 commit record，表明这个 log 包含一个完整的操作。从这之后，系统调用才会将这些写复制到磁盘上的文件系统数据结构。当所有这些写完成以后，系统调用又会将 disk 上的 log 给擦除掉。

​	如果系统故障并且重启了，在运行任何程序之前，文件系统代码从故障中恢复的操作如下：

- 如果 log 被标记为包含一个完整的操作，那么恢复代码会将这些写复制到他们所属的磁盘中的文件数据结构。
- 如果 log 被标记为不包含完整的操作，那么恢复代码就会忽略这个 log。
- 恢复代码最后通过擦出 log 结束。

## 四、Log design

log 驻留在一个已知的固定的位置上，指定为 superblock。它由一个 head block 以及后面跟着的一连串更新过的 block 副本组成（logged blocks）。head block 包含一个 sector number 的数组，每个对应一个 log block，还有 log block 的计数。磁盘上 head block 中的计数为 0，表示日志中没有事务，或者非 0，表示日志中包含一个完整的已提交事务，并且含有指定的 log block 数。xv6 写入 header block 当一个事务 commit 时，而不是之前，并且会把计数设置为 0 在将 logged blocks 复制到文件系统之后；因此，如果故障发生在一个事务中途，会导致 log 的 head block 为 0；一个在 commit 以后的故障，会使得一个不为 0 的计数。

​	每一个系统调用的代码指示 write sequence 的开始和结束，这写 wirte sequence 必须是相对于故障来说是原子的。为了允许不同程序并发的执行文件系统的操作，logging 系统能够积累多个系统调用的写在一个事务上。因此一个 commit 可能会涉及多个完整的系统调用的写操作。为了避免跨事务分割系统调用，日志系统只在没有文件系统系统调用进行时提交。

​	这种将多个事务一起提交的设计叫做组提交（group commit）。组提交减少磁盘操作的数量，因为它将提交的固定成本分摊到多个操作上。组提交还同时为磁盘系统提供了更多的并发写操作，可能允许磁盘在单个磁盘旋转期间写所有这些操作。Xv6 的 virtio 驱动程序不支持这种批处理，但是 Xv6 的文件系统设计允许这样做。

​	Xv6 在磁盘上指定了固定数量的空间来保存日志。系统调用在事务中写入的块总数必须适合该空间。这有两个后果。不允许单个系统调用写入超过日志空间的不同块。这对大多数系统调用来说不是问题，但是其中两个调用可能会写入许多块：write 和 unlink。一个大的文件写入可以写入许多数据块和许多 bitmap 块以及一个 inode 块；取消一个大文件的 link 可能会写入许多 bitmap 块和一个 inode。Xv6的写系统调用将大的写操作分解成多个适合日志的小的写操作，unlink 不会导致问题，因为实际上Xv6文件系统只使用一个位图块。日志空间有限的另一个后果是，日志系统不允许启动系统调用，除非它确定系统调用的写操作能够容纳日志中剩余的空间。

## 五、Inode layer

inode 在这里可以有两个意思：它可能指代一个在磁盘上的数据结构，包含了文件的大小和数据 block 的编号列表；或者指的是内存中的 inode，包含了一个磁盘 inode 的副本，以及其他内核要使用的信息。

​	磁盘上的 inode 被打包进一个连续的磁盘的区域里，被称为 inode blocks。每个 inode 都是相同大小的，所以这很简单，给一个 number n，就能找到磁盘上的第 n 个 inode。事实上，这个数字 n，叫做 inode 的 i-number，是 inode 标识如何在 xv6 中实现的体现。

​	磁盘上的 inode 声明为 struct dinode。 

```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

type 表示文件、目录或者特殊文件（例如驱动），type 为 0 表示磁盘上的 inode 是空的。nlink 表示目录对这个 inode 的引用次数，为了分辨何时一个磁盘 inode 和他的数据块应该被释放。size 记录了文件内容的字节数。addrs 数组记录了持有文件内容的块的数量。

​	内核保持一组 inode 存活在内存中，并放在 itable 里面。struct inode 是 磁盘上 struct dinode 在内存上的副本。

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

内核只会在有一个 c 指针指向这个 inode 的时候才会保存这个 inode 到内存里。并且内核在这个 inode 的 ref 降为 0 的时候丢弃它。ref 就保存了内存中 c 指针指向这个 inode 的次数。iget 和 iput 函数请求和盛放一个指向 inode 的指针，修改这个引用计数。 inode 的指针可以来自文件描述符、当前工作目录、瞬时的内核代码例如 exec。

​	在 xv6 的 inode 代码中有四种锁或者类锁机制。itable.lock 保护索引节点在索引节点表中最多出现一次的不变性，以及内存中索引节点的 ref 字段对指向该索引节点的内存指针数量进行计数的不变性。每个内存中的 inode 都有一个包含睡眠锁的锁字段，这可以确保对inode 的字段（如文件长度）以及 inode 的文件或目录内容块的独占访问。如果索引节点的 ref 大于零，则系统将在表中维护该索引节点，而不会为其他索引节点重用该表项。最后，每个 inode 包含一个 nlink 字段(在磁盘上，如果在内存中，则在内存中复制)，该字段计算引用文件的目录条目的数量;如果索引节点的链接数大于零，Xv6 就不会释放它。

​	一个 struct inode 指针通过 iget() 返回，会确保是合法的直到对应的 iput() 调用。inode 不会被删除，并且指针指向的内存也不会被另一个 inode 重用。iget 提供一个不是独有的 inode 访问，以至于可以有很多指针指向同一个 inode。文件系统代码的很多部分都依赖于 iget 的这一种特性，不仅持有对 inode 的长期的引用（比如打开的文件和当前的目录），而且防止竞争同时也避免在同时操作多个 inode 的代码产生死锁（例如查找 pathname）。

## 六、Directory layer

一个目录的内部实现就和文件一样。它的 inode 类型为 T_DIR，并且他的数据是一个目录条目的序列。每一个条目是一个 struct dirent，包含了一个 name 和一个 inode 序号。name 最长为 DIRSIZ（14）个字符，如果更短，则会以 0 终止。目录条目的 inode 编号为 0 则表示空闲。

​	dirlookup 函数查找一个目录下面给定 name 的条目。如果找到了，返回对应 inode 的指针，解锁的 并且设置 *poff 为目录下条目的偏移量，主要是为了实现调用者编辑它。如果 dirlookup 找到一个 正确 name 的 entry，它更新 *poff 并且通过 iget 返回一个没有上锁的 inode。dirlookup 是 iget 返回一个不上锁的 inode 的原因。调用者有一个上锁的 dp，所以如果查找是为了找 .，一个当前目录的别名，企图在返回之前对 inode 上锁，会导致对 dp 重复上锁进而死锁（还有很多别的导致死锁的时间序列）。调用者可以解锁 dp 然后 锁上 ip，确保它一个时间段内只持有一个锁。

​	dirlink 函数将一个 name 和 inode 写入一个新的 directory 条目，并放入 dp 里。如果 name 已经存在了，dirlink 回返回一个错误。主循环读取目录条目去寻找一个还有分配的 entry。当它找到以后，会提前结束循环，然后设置 off，写到 off 位置。



​	