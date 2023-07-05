---
layout: post
title: "xv6—文件系统"
subtitle: "[xv6] File System"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - xv6
  - OS
---

## Introduction

> *此为*[MIT 6.S081](https://www.bilibili.com/video/BV19k4y1C7kA/?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click)*课程和*[xv6-Books](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf)*的学习笔记*

文件系统的目的是组织和存储数据。文件系统通常支持用户和应用程序之间的数据共享，以及持久性，以便在重新启动后数据仍然可用，文件系统是操作系统中除了shell以外最常见的用户接口。如下图所示，xv6文件系统实现分为七层：

- disk layer读取和写入virtio硬盘上的block
- buffer cache layer缓存block并同步其数据，确保某一时间点只有一个内核进程可以修改block中的数据
- logging layer允许更高层在一次事务（transaction）中将更新多个块，并确保在遇到崩溃时原子性的更新这些块
- inode layer是对文件的抽象，每个文件表示为一个inode，其中包含唯一的索引号（i-number）和一些保存文件数据的block
- directory layer将每个目录实现为一种特殊的索引结点，其内容是一系列目录项（directory entries），每个目录项包含一个文件名和索引号
- pathname layer提供了分层路径名，例如/usr/rtm/xv6/fs.c，并通过递归查找来解析文件名
- file descriptor layer使用文件系统接口抽象了许多Unix资源（管道、设备、文件等），简化了程序员的工作

![image-20230630101219792](https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202306301012467.png)

> 文件系统并不是唯一构建一个存储系统的方式。如果只是在磁盘上存储数据，你可以想出一个完全不同的API。举个例子，数据库也能持久化的存储数据，但是数据库就提供了一个与文件系统完全不一样的API。所以记住这一点很重要：还存在其他的方式能组织存储系统。文件系统通常由操作系统提供，而数据库如果没有直接访问磁盘的权限的话，通常是在文件系统之上实现的（注，早期数据库通常直接基于磁盘构建自己的文件系统，因为早期操作系统自带的文件系统在性能上较差，且写入不是同步的，进而导致数据库的ACID不能保证。不过现代操作系统自带的文件系统已经足够好，所以现代的数据库大部分构建在操作系统自带的文件系统之上）。

首先，最重要的可能就是**inode，这是代表一个文件的对象，并且它不依赖于文件名**。实际上，inode是通过自身的编号来进行区分的，这里的编号就是个整数。所以文件系统内部通过一个数字，而不是通过文件路径名引用inode。同时，基于之前的讨论，inode必须有一个link count来跟踪指向这个inode的文件名的数量。一个文件（inode）只能在link count为0的时候被删除。实际的过程可能会更加复杂，实际中还有一个openfd count，也就是当前打开了文件的文件描述符计数。一个文件只能在这两个计数器都为0的时候才能被删除。

文件系统中核心的数据结构就是inode和file descriptor。后者主要与用户进程进行交互。

## Disk

实际中有两种磁盘最常见，即SSD（固态硬盘）和HDD（机械硬盘），这两类存储虽然有着不同的性能，但是都在合理的成本上提供了大量的存储空间。SSD通常是0.1到1毫秒的访问时间，而HDD通常是在10毫秒量级完成读写一个disk block。

磁盘的一些术语：

- sector通常是磁盘驱动可以读写的最小单元，它过去通常是512字节
- block通常是操作系统或者文件系统视角的数据，它由文件系统定义，通常来说一个block对应了一个或者多个sector。在xv6中它是1024字节（2个sector）

存储设备连接到了电脑总线之上，总线也连接了CPU和内存。文件系统运行在CPU上，数据会以读写block的形式存储在SSD或者HDD。在内部，SSD和HDD工作方式完全不一样，但是对于硬件的抽象屏蔽了这些差异。磁盘驱动通常会使用一些标准的协议，例如PCIE，与磁盘交互。从上向下看磁盘驱动的接口，大部分的磁盘看起来都一样，你可以提供block编号，在驱动中通过写设备的控制寄存器，然后设备就会完成相应的工作。这是从一个文件系统的角度的描述。尽管不同的存储设备有着非常不一样的属性，从驱动的角度来看，你可以以大致相同的方式对它们进行编程。

从文件系统的角度来看磁盘还是很直观的。因为对于磁盘就是读写block或者sector，我们可以将磁盘看作是一个巨大的block的数组，数组从0开始，一直增长到磁盘的最后。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRhzbAZwhuzp63wWdRE%2F-MRibTsYtaP0N-PN_5wB%2Fimage.png?alt=media&token=22b056b7-2a33-44ac-8cdc-f684845ffad3)

而文件系统的工作就是将所有的数据结构以一种能够在重启之后重新构建文件系统的方式，存放在磁盘上。虽然有不同的方式，但是xv6使用了一种非常简单，但是还挺常见的布局结构：

- block0被用作boot sector来启动操作系统，通常包含了足够启动操作系统的代码，之后再从文件系统中加载操作系统的更多内容（BIOS 会读取硬盘驱动器上的 boot block，并将控制权转交给 boot block 中的引导程序）
- block1通常被称为super block，包含有关文件系统的元数据（文件系统大小（以块为单位）、数据块数、索引节点数和日志中的块数），超级块由一个名为mkfs的单独的程序填充，该程序构建初始文件系统，描述了文件系统的布局

- 在xv6中，log block从block2开始，到block32结束。实际上log的大小可能不同，这里在super block中会定义log就是30个block
- 接下来在block32到block45之间，xv6存储了inode block。多个inode会打包存在一个block中，一个inode是64字节（前面说了xv6中一个block是1024字节，也就是16个inode）
- 之后是bitmap block，这是我们构建文件系统的默认方法，它只占据一个block。它记录了数据block是否空闲。
- 之后就全是data block了，数据block存储了文件的具体内容和目录的内容。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRhzbAZwhuzp63wWdRE%2F-MRielGcbrHOzPCrxHcO%2Fimage.png?alt=media&token=f685aafe-7c22-4965-9936-d811b090023d">

通常来说，log blocks、inode blocks和bitmap block被统称为metadata block，它们虽然不存储实际的数据，但是它们存储了能帮助文件系统完成工作的元数据。

```c
// mkfs computes the super block and builds an initial file system. The
// super block describes the disk layout:
struct superblock {
  uint magic;        // Must be FSMAGIC
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
  uint logstart;     // Block number of first log block
  uint inodestart;   // Block number of first inode block
  uint bmapstart;    // Block number of first free map block
};

#define FSMAGIC 0x10203040 
//在xv6中，当操作系统加载文件系统时，它会首先读取存储设备的引导块（boot block），然后根据引导
//块中的信息找到文件系统的超级块。接着，操作系统会检查超级块的magic字段是否等于FSMAGIC，以确认
//文件系统的有效性。如果文件系统的magic字段与FSMAGIC不匹配，操作系统可能会认为文件系统损坏或不
//支持该文件系统类型，并采取相应的处理措施，如尝试加载其他文件系统或显示错误信息。
//不同的文件系统通常会使用不同的magic number
```

xv6使用的磁盘是virtio_disk，相关操作代码在virtio_disk.c中，就不细看了（。

## Buffer cache Layer

Buffer cache主要有两个任务：

- 同步对block的访问，以确保block在内存中只有一个副本，并且一次只有一个内核线程使用该副本

- 缓存经常访问的block，以便不需要从磁盘重新读取它们（内存的速度比磁盘快得多的多）

Buffer cache layer提供的接口主要是bread和bwrite：bread获取一个buf（一个可以在内存中读取或修改的block的副本，上锁的）；bwrite将修改后的buf写入磁盘上的相应block。Buffer cache layer使用睡眠锁来实现同步机制和控制对每个buf的共享访问，以确保每个buf（也是每个block）每次只被一个线程使用。此外，内核线程在使用完一个buf后必须通过调用brelse释放。

Buffer cache layer中保存磁盘块的buf数量固定，这意味着如果文件系统请求还未存放在buffer cache中的块时，buffer cache会回收最近使用最少的缓冲区（局部性原则，LRU）。

### Code

buffer cache部分的代码还是很容易看懂，难点主要在于睡眠锁。首先所有进程都是通过访问bcache这个双向链表来对buf进行读取和修改，其中有个head结点只用于快速索引，不存储相关数据。

每个sector最多只有一个buf。文件系统使用缓冲区上的锁进行同步，buf的睡眠锁保护buf内容的读写，而bcache.lock保护bcache链表的信息。bget在bcache.lock临界区域之外获取buf的睡眠锁是安全的，因为非零b->refcnt防止buf被重新用于不同的磁盘块。如果所有buf都不空闲，表示太多进程同时执行文件系统调用，bget将会panic。PS：xv6这个比较简单，没做太多容错处理，一个更优雅的响应可能是在缓冲区空闲之前休眠，尽管这样可能会出现死锁。一旦bread将buf返回给其调用者，调用者就可以独占使用缓冲区（因为bget返回的是locked的buf），并可以读取或写入数据字节。如果调用者确实修改了buf，则必须在释放buf之前调用bwrite将更改的数据写入磁盘。当调用方使用完buf后，它必须调用brelse来释放缓冲区(brelse是b-release的缩写，起源于Unix，也用于BSD、Linux和Solaris）。brelse释放睡眠锁并将刚使用完的buf移动到bcache.head->next（这样就按使用频率排序了）。

```c
struct buf {
  int valid;   // has data been read from disk?1表示存储有磁盘某一block的数据
  int disk;    // does disk "own" buf?缓冲区数据是否更新到磁盘block
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;      // 引用次数
  struct buf *prev; // LRU cache list
  struct buf *next;
  uchar data[BSIZE]; // Block Size 1024B
};

struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head; // 头节点
} bcache;

// 比较常规的bcache链表初始化过程
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}

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

  // Not cached. 代码能运行到这已说明缓存中没有这个block，所以下面直接就b->prev
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    // 找当前没有用的buf
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
    // buf中无有效数据，可见不管咋样内存中也没有某块的buf，都得先读内存中的buf
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

// Release a locked buffer.
// Move to the head of the most-recently-used list.
// 释放的时候更新buffer cache队列，刚释放的就放head->next
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.lock);
  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.（也就是没有进程需要用这个buf/block了）
    b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
  
  release(&bcache.lock);
}
```

## Logging Layer

许多文件系统操作都涉及到对磁盘的多次写入，但部分写操作崩溃后可能会使磁盘上的文件系统处于不一致的状态，因此文件系统设计中最有趣的问题之一是崩溃恢复。例如，假设在文件截断（将文件长度设置为零并释放其内容块）期间发生崩溃。根据磁盘写入的顺序，崩溃可能会留下对标记为空闲的内容块的引用的inode，也可能留下已分配但未引用的内容块。后者相对来说是良性的，但引用已释放块的inode在重新启动后可能会导致严重问题。重新启动后，内核可能会将该块分配给另一个文件，现在我们有两个不同的文件无意中指向同一块。如果xv6支持多个用户，这种情况可能是一个安全问题，因为旧文件的所有者将能够读取和写入新文件中的块，而新文件的所有者是另一个用户。

>在文件截断过程中，文件系统通常需要执行以下操作：
>
>1. 更新inode：文件系统会更新文件的inode，将其长度设置为零，并释放之前分配的内容块。
>2. 释放内容块：文件系统会标记之前分配给文件的内容块为free状态，以便可以重新分配给其他文件使用。
>
>当发生崩溃时，可能会出现以下情况：
>
>情况一：崩溃发生在更新inode之前。在这种情况下，文件系统无法将文件的长度设置为零，因此文件仍然保留着之前的长度和内容块。这可能导致文件系统中存在已分配但未被引用的内容块，这些块无法通过正常的文件路径进行访问。
>
>情况二：崩溃发生在释放内容块之前。在这种情况下，文件系统无法将之前分配的内容块标记为自由状态。这可能导致inode仍然引用着被标记为free的内容块，这些内容块可以被其他文件重新分配使用。

xv6通过简单的日志记录形式解决了文件系统操作期间的崩溃问题：xv6系统调用不会直接写入磁盘上的文件系统数据结构。相反，它会在磁盘上的log中放置它希望进行的所有磁盘写入的描述。一旦系统调用记录了它的所有写入操作，它就会向磁盘写入一条特殊的commit记录，表明日志包含一个完整的操作。此时，系统调用将写操作修改到磁盘上的文件系统数据结构。完成这些写入后，系统调用将擦除该日志。

如果系统崩溃并重新启动，则在运行任何进程之前，文件系统代码将按如下方式从崩溃中恢复：如果日志标记为包含完整操作，则恢复代码会通过写操作修改磁盘文件系统中的相应block。如果日志没有标记为包含完整操作，则恢复代码将忽略该日志并擦除。

> 从上面的描述可以看到xv6通过日志实现对磁盘block同步的原子操作，即要么操作没有发生，要么操作全部完成。

### Design

Logging layer由一个header block和一系列更新块的副本（logged block，logged block表示已经记录了操作信息的日志块，而log block仅表示日志块）组成，共分配了30个block。Header block包含一个扇区号数组（每个logged block对应一个扇区号）以及日志块的计数。Header block的计数为零表示日志中没有事务；非零表示日志包含一个完整的已提交事务，并具有指定数量的logged block。只有在事务commit时xv6才向header block写入数据，并在将logged blocks修改到文件系统后将计数设置为零。

考虑到崩溃恢复，磁盘写入序列必须具有原子性。为了允许不同进程并发执行文件系统操作，日志系统可以将多个系统调用的写入累积到一个事务中。因此，单个提交可能涉及多个完整系统调用的写入。为了避免在事务之间拆分系统调用，**日志系统仅在没有文件系统调用（对应代码中的outstanding为0）进行时提交**。

同时提交多个事务的想法称为组提交（group commit）。组提交减少了磁盘操作的数量，因为多个事务分摊了成本固定的一次提交，组提交还同时为磁盘系统提供更多并发写操作。

xv6保存日志的空间是固定的，也就是说事务中系统调用写入的块总数必须可容纳于该空间。这可能导致两个后果（这部分对应begin op中 else if判断）：

- 任何单个系统调用都不允许写入超过日志空间的不同块。这对于大多数系统调用来说都不是问题，但write和unlink可能会写入许多块：一个大文件的write可以写入多个数据块和多个位图块以及一个inode块；unlink大文件可能会写入许多位图块和inode。因此，xv6的write系统调用将大的写入分解为适合日志的多个较小的写入。unlink不会导致此问题，因为实际上xv6文件系统只使用一个位图块。
- 除非确定系统调用的写入将可容纳于日志中剩余的空间，否则日志系统无法允许启动系统调用。

### Code

如果只看上面的文字描述其实挺懵的（，细读了log的代码后会好很多。就像上面描述的，xv6中磁盘中log区域分为一个header block和若干logged block，对于logged block的修改是通过header block进行的，由于我们的操作肯定先是在内存中，所以又xv6定义了一个log结构来控制log分区。其次，需要注意的是关于对磁盘的修改都是先修改buf，然后再修改到日志，如果要提交了，再修改到磁盘。

> log部分的代码还是有点难度的

系统调用中典型的日志使用如下：

```shell
 begin_op();
 ...
 bp = bread(...);
 bp->data[...] = ...;
 log_write(bp);
 ...
 end_op();
```
源代码
```c
struct logheader {
  int n;              // 当前日志中的条目数量
  int block[LOGSIZE]; // 对应cache block
};

struct log {
  struct spinlock lock;
  int start;       // start blocknumber
  int size;
  int outstanding; // how many FS sys calls are executing. 正在进行的操作数量
  int committing;  // in commit(), please wait. 是否有其他线程在提交日志
  int dev;
  struct logheader lh;
};
struct log log;    // 内存中的log区域的抽象，包含log的元数据和logheader

// 初始化也就是通过sb和logheader block的相关信息把内存中的log数据结构中的每一项填满
// 第一个用户进程运行之前的引导期间由fsinit调用的
void
initlog(int dev, struct superblock *sb)
{
  if (sizeof(struct logheader) >= BSIZE)
    panic("initlog: too big logheader");

  initlock(&log.lock, "log");
  log.start = sb->logstart;
  log.size = sb->nlog;
  log.dev = dev;
  recover_from_log();
}

static void
recover_from_log(void)
{
  read_head();
  install_trans(); // if committed, copy from log to disk
  log.lh.n = 0;
  write_head(); // clear the log
}

// Read the log header from disk into the in-memory log header
// 读取磁盘中的logheader block到内存中的log
static void
read_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *lh = (struct logheader *) (buf->data);
  int i;
  log.lh.n = lh->n;
  for (i = 0; i < log.lh.n; i++) {
    log.lh.block[i] = lh->block[i];
  }
  brelse(buf);
}

// Copy committed blocks from log to their home location
// 将日志中的条目（已经提交了的log block）更新到对应的磁盘上的cache block
// PS：晦涩的命名？
static void
install_trans(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *lbuf = bread(log.dev, log.start+tail+1); // read log block
    struct buf *dbuf = bread(log.dev, log.lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    bunpin(dbuf);
    brelse(lbuf);
    brelse(dbuf);
  }
}

// Write in-memory log header to disk.
// This is the true point at which the
// current transaction commits.
// 将内存中的log更新到磁盘中的log header block
static void
write_head(void)
{
  struct buf *buf = bread(log.dev, log.start);
  struct logheader *hb = (struct logheader *) (buf->data);
  int i;
  hb->n = log.lh.n;
  for (i = 0; i < log.lh.n; i++) {
    hb->block[i] = log.lh.block[i];
  }
  bwrite(buf);
  brelse(buf);
}

// called at the start of each FS system call.
void
begin_op(void)
{
  acquire(&log.lock);
  while(1){
    // 有其他线程在提交日志，睡眠等待
    if(log.committing){
      sleep(&log, &log.lock);
    } else if(log.lh.n + (log.outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // +1是先加上当前这个操作看是否已经超过了系统预定的LOGSIZE大小
      // 保守地假设每个系统调用最多可以写入MAXOPBLOCKS个不同的块
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log.lock);
    } else {
      log.outstanding += 1;
      release(&log.lock);
      break;
    }
  }
}

// called at the end of each FS system call.
// commits if this was the last outstanding operation.
void
end_op(void)
{
  int do_commit = 0;  // 是否需要提交日志的标志

  acquire(&log.lock);
  // 这个函数是修改操作的最后才调用的，前面有log_write，因此这里outstanding-1
  log.outstanding -= 1;
  if(log.committing)
    panic("log.committing");
  if(log.outstanding == 0){
    do_commit = 1;      // 需要提交日志
    log.committing = 1; // 一个提交操作正在进行
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log.outstanding has decreased
    // the amount of reserved space.
    // 这里调用wakeup是因为前面outstanding-1了，前面调用begin_op可能因为LOGSIZE大小导致
    // sleep，因此这里wakeup
    wakeup(&log);
  }
  release(&log.lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit();
    acquire(&log.lock);
    log.committing = 0;
    wakeup(&log);
    release(&log.lock);
  }
}

// Copy modified blocks from cache to log.
// 将提交的修改从cache（buf）复制到log block
static void
write_log(void)
{
  int tail;

  for (tail = 0; tail < log.lh.n; tail++) {
    struct buf *to = bread(log.dev, log.start+tail+1); // log block
    struct buf *from = bread(log.dev, log.lh.block[tail]); // cache block
    memmove(to->data, from->data, BSIZE);
    bwrite(to);  // write the log
    brelse(from);
    brelse(to);
  }
}

// Caller has modified b->data and is done with the buffer.
// Record the block number and pin in the cache by increasing refcnt.
// commit()/write_log() will do the disk write.
//
// log_write() replaces bwrite(); a typical use is:
//   bp = bread(...)
//   modify bp->data[]
//   log_write(bp)
//   brelse(bp)
// 这个函数主要作用是更新header中的block数组（记录的是对应的cache block）
// 也就是当我们更新一个block时，首先会更新其对应的buf，这个时候我们需要将这个buf添加到log 
// header中的block数组（如果无）
void
log_write(struct buf *b)
{
  int i;

  if (log.lh.n >= LOGSIZE || log.lh.n >= log.size - 1)
    panic("too big a transaction");
  if (log.outstanding < 1)
    panic("log_write outside of trans");

  acquire(&log.lock);
  for (i = 0; i < log.lh.n; i++) {
    if (log.lh.block[i] == b->blockno)   // log absorbtion
      break;
  }
  log.lh.block[i] = b->blockno;
  if (i == log.lh.n) {  // Add new block to log?
    // 防止这个block被换出，修改到data block后(install_trans)会调用unpin解除
    // bpin和bunpin的代码在缓存部分没有细讲，其实也很简单，无非是增减refcnt，本质还是LRU
    bpin(b);
    log.lh.n++;
  }
  release(&log.lock);
}
```

日志提交的关键代码如下（具体调用的时机为end_op函数）：

- 将缓存中被修改的cache block写入对应的log block
- 将内存中log的头部信息更新写入log header block
- log block -> cache block -> data block，完成文件系统的更新
- 更新log header中条目数量为0
- 清空日志，表示事务完成

```c
static void
commit()
{
  if (log.lh.n > 0) {
    write_log();     // Write modified blocks from cache to log
    write_head();    // Write header to disk -- the real commit
    install_trans(); // Now install writes to home locations
    log.lh.n = 0;
    write_head();    // Erase the transaction from the log
  }
}
```



## Inode Layer

接下来我们看一下磁盘上存储的inode究竟是什么？

- 通常来说它有一个type字段，区分文件、目录和特殊文件（设备），type为零表示磁盘inode是空闲的
- nlink字段，也就是link计数器，用来跟踪究竟有多少文件名指向了当前的inode
- size字段，表明了文件数据有多少个字节
- 不同文件系统中的表达方式可能不一样，不过在xv6中接下来是一些block的编号，例如编号0，编号1，等等。xv6的inode中总共有12个block编号，这些被称为direct block number，这12个block指向了构成文件的前12个block。举个例子，如果文件只有2个字节，那么只会有一个block编号0，它包含的数字是磁盘上文件前2个字节的block的位置。
- 之后还有一个indirect block number，它对应了磁盘上一个block，这个block包含了256（一个block number是4字节，1024/4=256）个block number，这256个block number包含了文件的数据。所以inode中block number 0到block number 11都是direct block number，而block number 12保存的indirect block number指向了另一个block。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRhzbAZwhuzp63wWdRE%2F-MRiq3PDZ1MKm5xRPATf%2Fimage.png?alt=media&token=b690c6fe-e665-4ded-adc7-91be326015d0">

基于上面的内容，xv6中文件最大的长度计算可得(256+12) * 1024 = 268KB。这是个很小的文件长度，实际的文件系统，文件最大的长度会大的多得多。**实际中可以用类似page table的方式，构建一个双重indirect block number指向一个block，这个block中再包含了256个indirect block number，每一个又指向了包含256个block number的block**。这里修改了inode的数据结构，可以使用类似page table的树状结构，也可以按照B树或者其他更复杂的树结构实现。xv6这里极其简单，基本是按照最早的Uinx实现方式来的，不过可以实现更复杂的结构。

> 假设我们需要读取文件的第8000个字节，那么你该读取哪个block呢？从inode的数据结构中该如何计算呢？
>
> 8000 / 1024 = 7 < 12 8000 % 1024 = 832 
>
> 也就是在第7个block（direct block number）的第832个字节偏移量处就是第8000个字节

在xv6中，文件目录结构极其简单，每一个目录包含了directory entries，每一条entry大小为16个字节：

- 前2个字节包含了目录中文件或者子目录的inode编号
- 接下来的14个字节包含了文件或者子目录名

```c
// Directory is a file containing a sequence of dirent structures.
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

> 对于实现路径名查找，上述信息就足够了。假设我们要查找路径名“/y/x”，我们该怎么做呢？
>
> 从路径名我们知道，应该从root inode开始查找。通常root inode会有固定的inode编号，在xv6中，这个编号是1。inode从block 32开始，如果是inode1，那么必然在block 32中的64到128字节的位置。所以文件系统可以直接读到root inode的内容。
>
> 对于路径名查找程序，接下来就是扫描root inode包含的所有block，以找到“y”，也就是读取所有的direct block number和indirect block number。结果可能是找到了，也可能是没有找到。如果找到了，那么目录y也会有一个inode编号，假设是251。我们可以继续从inode 251查找，先读取inode 251的内容，之后再扫描inode所有对应的block，找到“x”并得到文件x对应的inode编号，最后将其作为路径名查找的结果返回。

xv6对于目录和文件的查找设计的很简单，即通过线性扫描实现。实际的文件系统会使用更复杂的数据结构来使得查找更快。

关于xv6中关于inodes的具体过程，直接看[Frans的讲解](https://www.bilibili.com/video/BV19k4y1C7kA?p=13&vd_source=1eef42fa21ab7c4e759ac52299a8dfb1)，比较清晰。

### Code

```c
#define BSIZE 1024  // block size
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
// 磁盘上的inode的定义，64字节
```

```c
struct superblock sb; 

// Read the super block.
static void
readsb(int dev, struct superblock *sb)
{
  struct buf *bp;

  bp = bread(dev, 1);
  memmove(sb, bp->data, sizeof(*sb));
  brelse(bp);
}

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

struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;

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


```

