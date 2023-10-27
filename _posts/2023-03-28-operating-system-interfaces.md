---
layout: post
title: "MIT 6.S081—操作系统接口"
subtitle: "Operating system interfaces"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - xv6
  - OS
---

## Introduction

> 此为[MIT 6.S081](https://www.bilibili.com/video/BV19k4y1C7kA/?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click)课程和[xv6-Books](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf)的学习笔记

**About xv6** 

xv6 is **modeled on Dennis Ritchie’s and Ken Thompson’s Unix Version 6 (v6)**. xv6 loosely **follows the structure and style of v6**, but is implemented in **ANSI C** for a multi-core **RISC-V**.

**Operating system purposes**

- Abstraction：管理和抽象底层硬件资源
- Multiplex：多个程序之间共享一台计算机（硬件资源）
- Isolation：程序之间的隔离
- Sharing：不同程序进行数据交互，协同完成任务
- Security (Access Control System/Permission System)：文件的准入控制、访问控制和权限等
- Performance：发挥硬件的高性能
- Range of Uses：通用OS

**Operating system structure**

- 硬件资源：CPU、RAM、DISK、NET etc.
- Kernel (space)：管理硬件资源、进程等
- User (space)：vim、cc、shell etc.

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MH_0vtckm44OL-Ry80u%2F-MHfa6qidpjQGh_XpuRm%2Fimage.png?alt=media&token=13f0cc46-16b5-4e7e-bfc6-498ff3c6449a">

操作系统通过接口向用户程序提供服务。xv6采用传统的内核形式（**内核是一个特殊的程序，管理程序并为正在运行的程序提供服务**。当你打开计算机时，Kernel总是第一个被启动）。每个正在**运行的程序，称为进程**，都有包含指令、数据和堆栈的内存。

- 指令实现了程序的运算

- 数据是计算所依赖的变量

- 堆栈组织程序的过程调用

一台给定的计算机通常有许多进程，但只有一个内核。当一个进程需要调用一个内核服务时，它会调用一个系统调用，这是操作系统接口中的一个调用（**系统调用进入内核；内核执行服务并返回）**。因此，一个**进程在用户空间和内核空间之间交替执行**。内核使用CPU提供的硬件保护机制来确保每个在用户空间执行的进程只能访问它自己的内存。内核程序的执行拥有操控硬件的权限，它需要实现这些保护；而用户程序执行时没有这些特权。当用户程序调用系统调用时，硬件会提升权限级别，并开始执行内核中预先安排好的函数。

<img src="http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c1/p1.png">

**System call**

下面这些系统调用看起来就跟普通的函数调用一样。系统调用不同的地方是，它最终会跳到系统内核中（权限）。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MH_0vtckm44OL-Ry80u%2F-MHfh9WHGNw0YtnOk3tD%2Fimage.png?alt=media&token=c328c980-f7a0-4c10-9e97-caaee8e16520">

## Processes and memory

xv6进程由**内存（user space，包括指令、数据和堆栈）**和对kernel私有的每个**进程状态**组成。当一个进程没有执行时，**xv6保存它的CPU寄存器，并在下一次运行该进程时恢复它们**。内核用PID (process identifier) 标识每个进程。

**Fork syscall**

一个进程可以使用fork syscall创建一个新的进程。Fork创建了一个新的进程（子进程），其内存内容与调用进程（称为父进程）完全相同。在父进程中，fork返回子进程的PID；在子进程中，fork则返回零。

在xv6中，除了fork的返回值，两个进程是一样的。两个进程的指令是一样的，数据是一样的，栈是一样的，文件描述符的表单也是一样的。同时，两个进程又有各自独立的地址空间，它们都认为自己的内存从0开始增长，但这里是不同的内存。

```c
int pid = fork();
// 父进程
if(pid > 0) {
    printf("parent: child=%d\n", pid);
    // 父进程等待子进程结束（只用于父进程等待子进程）；传入参数为子进程结束状态status（int *），返回子进程PID
    pid = wait((int *) 0);
    printf("child %d is done\n", pid);
// 子进程
} else if(pid == 0) {
    printf("child: exiting\n");
    // 结束进程并释放资源（如内存和打开的文件），参数为结束状态status（会传给wait()（地址）），无需返回
    exit(0);
} else {
    printf("fork error\n");
}
```

> 上述程序的输出顺序可能会有很多种，这取决于父子进程谁先完成printf()

尽管最初子进程与父进程有着相同的内存内容，但是二者在运行中拥有**不同的内存空间和寄存器**：在一个进程中改变变量不会影响到另一个进程。例如当wait的返回值存入父进程的变量pid中时，并不会影响子进程中的pid，子进程中pid仍然为0。

**Exec syscall**

exec syscall使用从文件系统中存储的文件所加载的新内存替换调用进程的内存（根据指定的文件名找到可执行文件，并用它来取代调用进程的内容）。该文件必须有特殊的格式，它指定文件的哪部分存放指令，哪部分是数据，以及哪一条指令用于启动等等（即可执行文件，xv6使用ELF格式）。当exec执行成功，它不向调用进程返回数据（仅在发生错误时返回信息，因为exec会完全替换调用进程的内存，相当于当前进程不复存在了），而是使加载自文件的指令在ELF header中声明的程序入口处开始执行。exec有两个参数：可执行文件的文件名和字符串参数数组。

```c
char* argv[3];
argv[0] = "echo";
argv[1] = "hello";
argv[2] = 0;
exec("/bin/echo", argv);
printf("exec error\n");
```

> 上述代码片段将调用程序替换为了参数列表为echo hello的/bin/echo程序运行，多数程序忽略参数数组中的第一个元素，它通常是程序名

**Shell**

xv6的shell程序：
```c
int main(void){
  static char buf[100];
  int fd;

  // Ensure that three file descriptors (0 stdin; 1 stdout; 2 stderr) are open.
  while((fd = open("console", O_RDWR)) >= 0){
    if(fd >= 3){
      close(fd);
      break;
    }
  }

  // Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
    if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
      // Chdir must be called by the parent, not the child.
      buf[strlen(buf)-1] = 0;  // chop \n
      if(chdir(buf+3) < 0)
        fprintf(2, "cannot cd %s\n", buf+3);
      continue;
    }
    if(fork1() == 0)
      runcmd(parsecmd(buf));
    wait(0);
  }
  exit(0);
}
```

主循环使用getcmd函数从用户的输入中读取一行，然后调用fork创建一个shell进程的副本。父进程调用wait，子进程执行命令。例如：当用户向shell输入echo hello时，runcmd将以echo hello为参数被调用来执行实际命令。对于“echo hello”，它将调用exec。如果exec成功，那么子进程将从echo而不是runcmd执行命令，在某刻echo会调用exit，这将导致父进程从main中的wait返回。

**Fork & Exec**

从shell程序中可以看到，父进程在fork创建子进程后，子进程随即调用了exec执行程序。那其实可以把fork和exec进行组合成为一个系统调用，但各大操作系统并没有这样做。shell在其I/O重定向的实现中利用了这种分离，为了避免创建一个重复的进程然后立即替换它（使用exec会对当前进程进行完全替换）的浪费，操作系统内核通过使用虚拟内存技术（如copy-on-write （随着进程的复杂，fork()需要复制的越来越多））优化 fork 在这个用例中的实现。

可不可以不fork直接exec执行程序呢？很明显是不能的，那样新的程序就把shell给替换了，当当前进程结束后，一切就结束了。

> 有一些当时的历史原因

[知乎上关于这个问题的一些回答（写的还挺好的）](https://www.zhihu.com/question/66902460)

## I/O and File descriptors

**File descriptors**

文件描述符是一个小的整数（0，1，2...），**表示进程可以读取或写入的由内核管理的对象**。进程可以通过以下方法来获得一个文件描述符

- 打开一个文件、目录、设备

- 创建一个管道

- 复制一个已存在的描述符

我们通常将**文件描述符所指的对象称为“文件”（abstraction），文件描述符接口将文件、管道和设备之间的差异抽象出来，使它们看起来都像字节流；我们将输入和输出称为 I/O**。

**文件描述符本质上对应了内核中的一个表单数据**。内核维护了每个运行进程的状态，**内核会为每一个运行进程保存一个表单，表单的key是文件描述符**。这个表单让内核知道，每个文件描述符对应的实际内容是什么。这里比较关键的点是，**每个进程都有自己独立的一个从零开始的文件描述符空间**，所以如果运行了两个不同的程序，对应两个不同的进程，如果它们都打开一个文件，它们或许可以得到相同数字的文件描述符，但是因为内核为每个进程都维护了一个独立的文件描述符空间，这里相同数字的文件描述符可能会对应到不同的文件。

按照惯例，进程从文件描述符0读取（标准输入），将输出写入文件描述符1（标准输出），并将错误消息写入文件描述符2（标准错误）。正如我们将看到的，shell利用这个约定来实现I/O重定向和管道。shell确保它始终有三个打开的文件描述符，这是控制台的默认文件描述符。此外，新分配的文件描述符总是当前进程中编号最小的未使用描述符。

**Read syscall**

read(fd，buf，n)从文件描述符fd读取最多n字节，将它们复制到buf，并**返回读取的字节数**，引用文件的每个文件描述符都有一个与之关联的偏移量。read从当前文件偏移量开始读取数据，然后将该偏移量前进所读取的字节数。当没有更多的字节可读时，read返回0来表示文件的结束。

**Write syscall**

write(fd，buf，n)将buf中的n字节写入文件描述符，并**返回写入的字节数**。只有发生错误时才会写入小于n字节的数据。与读一样，write在当前文件偏移量处写入数据，然后将该偏移量向前推进写入的字节数：每个write从上一个偏移量停止的地方开始写入。

> 关于文件偏移量：如果两个文件描述符是通过一系列fork和dup调用（**复制一个现有的文件描述符，返回一个引用自同一个底层I/O对象的新文件描述符**）从同一个原始文件描述符派生出来的，那么它们共享一个偏移量。否则，文件描述符不会共享偏移量，即使它们来自于对同一文件的打开调用。

**Open syscall & Close syscall**

open(char* file, int flags)用于打开文件file，第二个参数由一组标志组成，这些标志以位表示，用于控制打开的操作，open**返回该文件的文件描述符**。

close(int fd)用于释放一个文件描述符，使其可以被未来使用的open、pipe或dup系统调用重用。

程序cat的本质是将数据从其标准输入复制到其标准输出，所以当在shell中只输入cat时会出现下列输出。

```bash
➜ ~ cat
abcd
abcd
xv6
xv6
```

**I/O Redirection**

文件描述符和fork相互作用，使I/O重定向更容易实现。fork复制父进程的文件描述符表及其内存，以便子进程可以与父进程在开始时拥有完全相同的打开文件。系统调用exec替换了调用进程的内存，但保留其文件表。此行为允许shell通过fork实现I/O重定向，在子进程中重新打开选定的文件描述符，然后调用exec来运行新程序。下面是shell运行命令cat < input.txt的代码的简化版本。

```c
char* argv[2];
argv[0] = "cat";
argv[1] = 0;
if (fork() == 0) {
    close(0);
    open("input.txt", O_RDONLY);
    exec("cat", argv);
}
```

在子进程关闭文件描述符0之后，open保证使用新打开的input.txt：0的文件描述符作为当前最小的可用文件描述符，cat然后执行文件描述符0(标准输入)，但引用的是input.txt（open为其分配文件描述符）。父进程的文件描述符不会被这个序列改变，因为它只修改子进程的描述符。

回到fork和exec分离这个问题：在这两个调用之间，shell有机会对子进程进行I/O重定向，而不会干扰主shell的I/O设置。我们可以想象一个假设的forkexec系统调用组合，但是用这样的调用进行I/O重定向是很笨拙的：Shell可以在调用forkexec之前修改自己的I/O设置(然后撤销这些修改)；或者forkexec可以将I/O重定向的指令作为参数；或者可以让每个程序(如cat)执行自己的I/O重定向。

## Pipes

管道是作为**一对文件描述符公开给进程的小型内核缓冲区，一个用于读取，一个用于写入**。将数据写入管道的一端使得这些数据可以从管道的另一端读取。管道**为进程提供了一种通信方式**。xv6中pipe(int p[])系统调用将创建一个管道，并将读/写文件描述符分别放入p[0]和p[1]。

下面的示例代码使用连接到管道读端的标准输入来运行程序wc。

```c
int p[2];
char *argv[2];
argv[0] = "wc";
argv[1] = 0;
pipe(p);
if (fork() == 0) {
    close(0);
    dup(p[0]);
    close(p[0]);
    close(p[1]);
    exec("/bin/wc", argv);
} else {
    close(p[0]);
    write(p[1], "hello world\n", 12);
    close(p[1]);
}
```

程序调用pipe，创建一个新的管道，并在数组p中记录读写文件描述符（p[0]用于读，p[1]用于写）。在fork之后，父子进程都有指向管道的文件描述符。子进程调用close和dup使文件描述符0指向管道的读取端（这里先使用close关闭了文件描述符0，接着用dup创建了一个新的文件描述符（因为0是此时最小未分配的文件描述符，因此这里实际分配的是文件描述符0）用于指向p[0]即管道的读端），然后关闭p中所存的文件描述符，并调用exec运行wc。当wc从它的标准输入读取时，就是从管道读取。父进程关闭管道的读取端，写入管道，然后关闭写入端。

**如果没有可用的数据，则管道上的read操作将会进入等待，直到有新数据写入或所有指向写入端的文件描述符都被关闭**，在后一种情况下，read将返回0，就像到达数据文件的末尾一样。事实上，read在新数据到达前会一直阻塞，这也是为什么上述程序中子进程在执行wc之前关闭管道的写入端：如果wc的文件描述符之一指向管道的写入端，wc将永远看不到文件的结束。

管道相比临时文件的优势：

- 首先，管道会自动清理自己；在文件重定向时，shell使用完临时文件后必须小心删除
- 其次，管道可以任意传递长的数据流，而文件重定向需要磁盘上足够的空闲空间来存储所有的数据。
- 第三，管道允许并行执行管道阶段，而文件方法要求第一个程序在第二个程序启动之前完成。
- 第四，如果实现进程间通讯，管道的块读写比文件的非块语义更有效率。

## File system

xv6中创建新文件和目录的系统调用：

- mkdir创建一个新目录
- open中若使用O_CREATE标志将会创建一个新的数据文件
- mknod创建一个新的设备文件

mknod创建一个引用设备的特殊文件。**与设备文件相关联的是主设备号和次设备号(mknod的两个参数)，它们唯一地标识了一个内核设备**。当进程稍后打开设备文件时，内核将使用内核设备实现read和write系统调用，而不是使用文件系统。

一个文件的名字和文件本身是不同的，**同一个底层文件（叫做inode，索引结点）可以有多个名字（叫做link，链接）。每个链接都由目录中的一个条目组成：包含一个文件名和一个inode引用**。Inode保存有关文件的元数据（用于解释或帮助理解信息的数据），包括其类型(文件/目录/设备)、长度、文件内容在磁盘上的位置以及指向文件的链接数。

fstat系统调用从文件描述符所引用的inode中检索信息，它填充一个stat类型的结构体，struct stat在stat.h(kernel/stat.h)中定义如下。

```c
#define T_DIR 1    // Directory
#define T_FILE 2   // File
#define T_DEVICE 3 // Device
struct stat {
    int dev;     // 文件系统的磁盘设备
    uint ino;    // Inode编号 （唯一标识）
    short type;  // 文件类型
    short nlink; // 指向文件的链接数
    uint64 size; // 文件字节数
};
```

link系统调用创建另一个文件名，该文件名指向与现有文件相同的inode。`link("a", "b")`表示创建了一个名字既为a又为b的新文件（省略b的创建过程）。

unlink系统调用从文件系统中删除一个名称。只有当文件的链接数为零且没有文件描述符引用时，文件的inode和包含其内容的磁盘空间才会被释放。

## Real world

Unix系统调用接口已经通过便携式操作系统接口(POSIX)标准进行了标准化。xv6与POSIX不兼容：它缺少许多系统调用(包括lseek等基本系统调用)，并且它提供的许多系统调用与标准不同。xv6的主要目标是简单明了，同时提供一个简单的类unix系统调用接口。现代内核比xv6提供了更多的系统调用和更多种类的内核服务。例如，它们支持网络工作、窗口系统、用户级线程、许多设备的驱动程序等等。现代内核不断快速发展，提供了许多超越POSIX的特性。

Unix通过一组文件名和文件描述符接口统一访问多种类型的资源(文件、目录和设备)。这个想法可以扩展到更多种类的资源；一个很好的例子是Plan9，它将“资源是文件”的概念应用到网络、图形等等。然而，大多数unix衍生的操作系统并没有遵循这条路。

xv6没有提供一个用户概念或者保护一个用户不受另一个用户的伤害;用Unix的术语来说，所有的Xv6进程都作为root运行。