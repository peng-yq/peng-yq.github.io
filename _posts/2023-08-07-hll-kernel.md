---
layout: post
title: "MIT 6.S081—高级编程语言实现操作系统的优劣势"
subtitle: "The benefits and costs of writing a
POSIX kernel in a high-level language"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - OS
---

## Benefits and costs of writing a POSIX kernel in C

现在很多操作系统都是用C实现的，除了熟悉的xv6外，一些更流行的运行在我们电脑和手机的操作系统：Windows、mac os、Linux、Android和iOS都是用C实现的。我们可能都问过应该用什么样的编程语言来实现操作系统内核？尤其时在发现操作系统中有Bug的时候，然后你会想，如果我使用一种其他的编程语言，或许我就不会有这些Bug了。这是一个经常出现的问题，这也是操作系统社区里的一个热烈争论的问题，但是并没有很多事实来支撑这里的讨论。针对这个问题，Robert和Morris选择了一种带garbage collector（GC）的高级编程语言（high-level language，HLL）编写了一个遵循POSIX标准的内核[Biscuit](https://pdos.csail.mit.edu/6.828/2020/readings/biscuit.pdf)，测量出高级编程语言的优劣势，并从safety，programmability和性能损失角度，探索使用高级编程语言而不是C语言实现内核的效果。这篇论文贡献了一些数据使得可以对应该使用什么样的编程语言来实现内核这个话题，有一些更深入的讨论。

用C语言编写操作系统的优点是什么呢？

- C提供了大量的控制能力，C可以完全控制内存分配和释放（malloc和free）；高级编程语言一般有GC，而不需要开发者去主动的申请和释放内存
- C语言几乎没有隐藏的代码，你几乎可以在阅读C代码的时候想象到对应的机器指令是什么
- 通过C可以有直接内存访问能力，你可以读写PTE的bit位或者是设备的寄存器
- 使用C会有极少的依赖，因为你不需要一个大的程序运行时。几乎可以直接在硬件上运行C程序。你们可以在XV6启动过程中看到这一点， 只通过几行汇编代码，你就可以运行C代码

但使用C语言编写操作系统也有一些缺点。在过去几十年已经证明了，很难写出安全的C代码，这里存在各种各样的Bug。

- buffer overruns，比如说数组越界，撑爆了stack等等
- use-after-free bugs，你可能会释放一些仍然在使用的内存，之后其他人又修改了这部分内存
- threads sharing dynamic memory，当线程共享内存时，很难决定内存是否可以被释放

查看CVE的网站，你可以发现有很多个Linux Bugs可以让攻击者完全接管机器显，这些都是非常严重的Bugs，这些Bug是由buffer overrun和一些其他memory-safety bug引起。

## Benefits and costs of writing a POSIX kernel in HLL

高级编程语言吸引人的一个原因是它提供了memory-safety，所以前面CVEs提到的所有Bugs，都将不再存在：要么当它们发生时程序运行时会检查数组是否越界，如果越界了就panic；要么高级编程语言不允许你写出引起Bug的代码。当然，高级编程语言还有一些其他的优点：

- 首先是Type safety，类型安全。高级编程语言在编译时或运行时能够检查类型的一致性，以防止类型错误和潜在的运行时错误
- 通过GC实现了自动的内存管理，所以free更容易了，GC会为你完成所有的内存释放工作
- 对并发更友好
- 有更好的抽象，接口和类等面向对象的语法使得可以写出更加模块化的代码

我们知道基本所有的操作系统都是通过C语言编写的，HLL有这么多的优点，那为什么这些操作系统不通过HLL编写呢？

- HLL通常有更差的性能，即有一些额外的代价，称为HLL tax，比如说索引一个数组元素时检查数据边界，比如说检查空指针，比如说类型转换。此外，GC也是有代价的。
- 除了性能之外，高级编程语言与内核编程本身不兼容：
	- HLL没有直接访问内存的能力，这从原则上违反了类型安全
	- HLL不能集成汇编语言，而在内核中的一些场景你总是需要一些汇编程序，比如说两个线程的context switching，或者系统启动
	- HLL本身支持的并发与内核需要的并发并不一致，比如我们在调度线程的时候，一个线程会将锁传递给另一个线程。一些并发管理模式在用户程序中不太常见，但是在内核中会出现。

**关于biscuit**

虽然历史上也有使用HLL编写的操作系统内核，但没有很多论文做HLL和C编写操作系统内核的对比测试（有一些使用HLL和C编写用户程序的对比，但内核设计还是有一些不一样）。一个原因可能是这里的工作有点棘手：如果你想得到正确的结果，你需要与产品级别的C内核进行对比，例如Linux，Windows等等。同时，你也需要构建一个产品级别的内核。很明显，这对于一个小的团队来说很难，因为有许多许多的Linux开发人员日复一日做了许多许多的更新才创造了Linux，所以很难用高级编程语言实现同样的功能并构建同样的内核。所以Robert他们构建了一个功能稍微少的系统内核。但这毕竟不是Linux，因此也不能给出一个十分清晰的答案。

- 用高级编程语言构建内核
- 保留与Linux中最重要的部分对等的功能
- 优化性能使得其与Linux基本接近，即使这里的功能与Linux并不完全一致，但在可接受范围内

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MXzMBDVZ4wNflk5sSH0%2F-MXzvIJN4Y690d_P13Km%2Fimage.png?alt=media&token=c0497c34-6ca3-4320-b4c8-5bc1731fee7e">

**对比方法的核心**

Biscuit，是一个为了论文专门用Golang写的内核，它以大概类似的方式提供了Linux中系统调用的子集。Biscuit和Linux的系统调用有相同的参数和相同的调用方式。并且我们在内核之上运行的是相同的应用程序，这里的应用程序是NGINX，这是一个web server，这里我们将相同的应用程序分别运行在Biscuit和Linux之上，应用程序会执行相同的系统调用，并传入完全相同的参数，Biscuit和Linux都会完成涉及系统调用的相同操作。之后，我们就可以研究高级编程语言内核和Linux之间的区别，并讨论优劣势是什么。因为Linux和Biscuit并不完全一样，它们会有一些差异，所以作者花费了大量的时间来使得这里的对比尽可能得公平。

**Why Golang**

- 这是一个静态编译的编程语言，和Python不同，这里没有解释器。静态编译的语言的性能通常更好，实际上Go编译器就非常好，所以基本上来说这是一种高性能编程语言

- Golang被设计成适合系统编程，而内核就是一种系统编程所以Golang也符合这里的场景。例如：
	- Golang非常容易调用汇编代码，或者其他的外部代码

	- Golang能很好地支持并发
	
	- Golang非常的灵活
	
- 另一个原因是Golang带有Garbage Collector。使用高级编程语言的一个优点就是你不需要管理内存，而GC是内存管理的核心

Robert也提到了rust，表示在写这个论文时，rust并不十分流行、稳定和成熟。但rust也是为系统编程而设计的，并且rust不带GC，这也跳过了使用GC带来的代价有多少这个问题。

## Biscuit

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MXzvXvUT30ptJeorKm4%2F-MY-B73XS4AOujw5jwIK%2Fimage.png?alt=media&token=fd3f837d-a975-44c4-978a-dc041e820cc9">

就像Linux和XV6一样，Biscuit是经典的monolithic kernel，它也有用户空间和内核空间。为了进行性能测试，用户空间程序统一使用C版本的应用程序，并且大部分应用程序都是多线程的（而xv6中一个用户程序只有一个线程）。对于每个用户空间线程，都有一个对应的位于内核的内核线程，这些内核线程是用Golang实现的，在Golang里面被称为goroutine。你可以认为goroutine就是普通的线程，就像xv6内核里的线程一样。区别在于，xv6中线程是由内核实现的，而这里的goroutine是由Go runtime提供。所以Go runtime调度了goroutine，Go runtime支持sleep/wakeup/conditional variable和同步机制以及许多其他特性，所以这些特性可以直接使用而不需要Biscuit再实现一遍。

Biscuit中的Go runtime直接运行在硬件上，当机器启动之后，就会启动Go runtime。但Go runtime通常是作为用户空间程序运行在用户空间，并且依赖内核提供服务，比如说为自己的heap向内核申请内存。所以Biscuit提供了一个中间层，使得即使Go runtime运行在裸机之上，它也认为自己运行在操作系统之上，这样才能让Go runtime启动起来。

Biscuit特性：

- 支持多核CPU。Golang对于并发有很好的支持，所以Biscuit也支持多核CPU，相比xv6有更好的同步协调机制
- 支持用户空间多线程，而XV6并没有
- 一个相比xv6更高性能的Journaled File System（注，Journaled就是指log，可以实现Crash Recovery）
- 它有在合理范围内较为复杂的虚拟内存系统，使用了VMAs并且可以支持mmap和各种功能
- 有一个完整的TCP/IP栈，可以与其他的服务器通过互联网连接在一起。
- 它还有两个高性能的驱动，一个是Intel的10Gb网卡，以及一个非常复杂的磁盘驱动AHCI，这比virtIO磁盘驱动要复杂的多

Biscuit支持的用户程序中：

- 每个用户程序都有属于自己的Page Table
- 用户空间和内核空间的内存是由硬件隔离的，也就是通过PTE的User/Kernel bit来区分
- 每个用户线程都有一个对应的内核线程，这样当用户线程执行系统调用时，程序会在对应的内核线程上运行。如果系统调用阻塞了，那么同一个用户地址空间的另一个线程会被内核调度起来
- 内核线程是由Go runtime提供的goroutine实现的

Biscuit中的系统调用：

- 用户线程将参数保存在寄存器中，通过一些小的库函数来使用系统调用接口
- 之后用户线程执行SYSENTER。现在Biscuit运行在x86而不是RISC处理器上，所以进入到系统内核的指令与RISC-V上略有不同。
- 但是基本与RISC-V类似，控制权现在传给了内核线程。
- 最后内核线程执行系统调用，并通过SYSEXIT返回到用户空间。

##  Evaluation: HLL benefits

> 关于作者如何解决heap exhaustion的内容这里就不记录了

Biscuit使用了Golang提供的高级编程语言特性，而不是为了得到好的性能而避开使用它们。

HLL的使用简化了Biscuit代码，例如gc和使用map从而避免线性扫描。

HLL可以避免一些内核相关的漏洞。

## Evaluation: HLL performance cost

测试环境

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MYAbAYCuNZb_gqyzfAB%2F-MYCr27qd6ZE2AHxfXDM%2Fimage.png?alt=media&token=ae88836c-bfb2-4b42-a529-f097cd26e356">

### Is biscuit's perf roughly similar to Linux?

作者将上面三个程序同时运行在Linux (选用的是debian9.4 Linux4.9.82)和biscuit上，并关闭了biscuit不提供的功能，使得对比尽可能的公平。吞吐量测试可以看到，Biscuit总是会比Linux更慢，mailbench可能差10%，nginx和redis差10%到15%。但是可以看出两个系统基本在同一个范围内，而不是差个2倍或者10倍。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MYAbAYCuNZb_gqyzfAB%2F-MYCtNqCH3YxBP3pIDa8%2Fimage.png?alt=media&token=dc021f2a-58c1-4e63-b6c4-01e0d2b1418c">

### HLL tax

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MYAbAYCuNZb_gqyzfAB%2F-MYCw9KpL9t3AdusrhMK%2Fimage.png?alt=media&token=25fb0869-2319-4980-aeef-148c155c8920">

## Conclusion

老实说这一节听到后面听懵逼的（，干脆直接省略了很多，直接搬出结论。

从研究中得出一些结论：

- 如果性能真的至关重要，比如说你不能牺牲15%的性能，那么你应该使用C
- 如果你想最小化内存使用，你也应该使用C
- 如果安全更加重要，那么应该选择高级编程语言
- 或许在很多场景下，性能不是那么重要，那么使用高级编程语言实现内核是非常合理的选择



