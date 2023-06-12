---
layout: post
title: "xv6—中断和设备驱动"
subtitle: "[xv6] Interrupts and device drivers"
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

驱动程序是操作系统中管理特定设备的代码：**它配置硬件设备，告诉设备执行操作，处理由此产生的中断，并与可能正在等待设备输入/输出的进程进行交互**。编写驱动可能很棘手，因为驱动程序与它管理的设备同时运行。此外，驱动程序必须理解设备的硬件接口，这可能很复杂，而且缺乏文档。

需要操作系统关注的设备通常可以被配置为生成中断，这是trap的一种。内核trap处理代码识别设备何时引发中断，并调用驱动程序的中断处理程序。中断对应的场景很简单，就是硬件想要得到操作系统的关注。例如网卡收到了一个packet，网卡会生成一个中断；用户通过键盘按下了一个按键，键盘会产生一个中断。**操作系统需要做的是，保存当前的工作，处理中断，处理完成之后再恢复之前的工作**。这里的保存和恢复工作，与我们之前看到的系统调用过程非常相似。所以系统调用，page fault，中断，都使用相同的机制。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MN6E1B0gJ_TakRKmn1w%2F-MNOdUodbUkIyuC_PvVX%2Fimage.png?alt=media&token=e20766c8-d452-489e-b91c-118df66fec45">

但中断和系统调用又有一些不一样的地方：

- **asynchronous：**当硬件生成中断时，Interrupt handler与当前运行的进程在CPU上没有任何关联。但如果是系统调用的话，系统调用发生在运行进程的context下。
-  **concurrency：**对于中断来说，CPU和生成中断的设备是并行的在运行。网卡自己独立的处理来自网络的packet，然后在某个时间点产生中断，但是同时，CPU也在运行。所以我们在CPU和设备之间是真正的并行的，我们必须管理这里的并行。
- **program device：**网卡，UART，而这些设备需要被编程。每个设备都有一个编程手册，就像RISC-V有一个包含了指令和寄存器的手册一样。设备的编程手册包含了它有什么样的寄存器，它能执行什么样的操作，在读写控制寄存器的时候，设备会如何响应。不过通常来说，设备的手册不如RISC-V的手册清晰，这会使得对于设备的编程会更加复杂。

外设中断来自于主板上的设备，下图是一个SiFive主板，如果你查看这个主板，你可以发现有大量的设备（以太网卡、MicroUSB，MicroSD等）连接在或者可以连接到这个主板上，主板上的各种线路将外设和CPU连接在一起。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNOedUoWwf-B8e0x8xX%2F-MNQgPHLVuAReIvEfuv_%2Fimage.png?alt=media&token=64896819-7b22-4c88-b654-41786fe49cd9">

下图是来自于SiFive有关处理器的文档，图中的右侧是各种各样的设备，例如UART0。我们知道UART0会映射到内核内存地址的某处，而所有的物理内存都映射在地址空间的0x80000000之上。**类似于读写内存，通过向相应的设备地址执行load/store指令，我们就可以对例如UART的设备进行编程**。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNOedUoWwf-B8e0x8xX%2F-MNQifH2z_uxd8s1bR7j%2Fimage.png?alt=media&token=6fa45459-e903-4e90-9fff-191f21dbbea9">

所有的设备都连接到处理器上，**处理器上是通过Platform Level Interrupt Control，简称PLIC来处理设备中断，PLIC会管理来自于外设的中断（实际上PLIC的作用只是分发中断，具体的处理还是交给CPU）**。查看PLIC的结构图，从左上角可以看出，我们有53个不同的来自于设备的中断。这些中断到达PLIC之后，PLIC会路由这些中断。图的右下角是CPU的核，PLIC会将中断路由到某一个CPU的核。如果所有的CPU核都正在处理中断，PLIC会保留中断直到有一个CPU核可以用来处理中断。所以PLIC需要保存一些内部数据来跟踪中断的状态。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNOedUoWwf-B8e0x8xX%2F-MNQmPRa0bzPQ99lcE2p%2Fimage.png?alt=media&token=35741a0d-0fc6-42e9-a841-15537f9fdaf4">

**PLIC处理中断的流程可总结如下**：

- PLIC会通知当前有一个待处理的中断
- 其中一个CPU核会claim接收中断，这样PLIC就不会把中断发给其他的CPU处理
- CPU核处理完中断之后，CPU会通知PLIC
- PLIC将不再保存中断的信息

**许多设备驱动程序在两种环境中执行代码**：

1. 上半部分在进程的内核线程中运行：上半部分通过**系统调用**进行调用，如希望设备执行I/O操作的`read`和`write`。这段代码可能会要求硬件执行操作（例如，要求磁盘读取块）；然后代码等待操作完成。最终设备完成操作并引发中断。
2. 下半部分在中断时执行。驱动程序的**中断处理程序**充当下半部分，计算出已经完成的操作，如果合适的话唤醒等待中的进程，并告诉硬件开始执行下一个正在等待的操作。

**RISC-V有许多与中断相关的寄存器**：

- SIE（Supervisor Interrupt Enable）寄存器。
  - 这个寄存器中有一个bit（E）专门针对例如UART的外部设备的中断；
  - 有一个bit（S）专门针对软件中断，软件中断可能由一个CPU核触发给另一个CPU核；
  - 还有一个bit（T）专门针对定时器中断。

- SSTATUS（Supervisor Status）寄存器。这个寄存器中有一个bit来打开或者关闭中断。每一个CPU核都有独立的SIE和SSTATUS寄存器，除了通过SIE寄存器来单独控制特定的中断，还可以通过SSTATUS寄存器中的一个bit来控制所有的中断。
- SIP（Supervisor Interrupt Pending）寄存器。当发生中断时，处理器可以通过查看这个寄存器知道当前是什么类型的中断。
- SCAUSE寄存器，这个寄存器会表明当前状态的原因是中断。
- STVEC寄存器，它会保存当trap，page fault或者中断发生时，CPU运行的用户程序的程序计数器，这样才能在稍后恢复程序的运行。

## Code: Console input

控制台驱动程序（console.c）通过连接到RISC-V的UART串口硬件接受用户键入的字符，并一次累积一行输入，处理如backspace和Ctrl-u的特殊输入字符，Shell等用户进程使用read系统调用从控制台获取输入行。在xv6中，UART硬件是由QEMU仿真的16550芯片来管理连接到终端或其他计算机的RS232串行链路。UART的内存映射地址起始于0x10000000或UART0。存在一些RISC-V硬件连接到UART的物理地址，以便载入(load)和存储(store)操作与设备硬件而不是内存交互。控制台驱动程序通过管理这些UART控制寄存器来实现其功能。例如，LSR寄存器包含指示输入字符是否正在等待软件读取的位。这些字符（如果有的话）可用于从RHR寄存器读取。每次读取一个字符，UART硬件都会从等待字符的内部FIFO寄存器中删除它，并在FIFO为空时清除LSR中的“就绪”位。UART传输硬件在很大程度上独立于接收硬件；如果软件向THR写入一个字节，则UART传输该字节。

xv6的main函数调用consoleinit函数来初始化UART硬件。该函数配置UART：UART对接收到的每个字节的输入生成一个接收中断，对发送完的每个字节的输出生成一个发送完成中断。

xv6的shell通过打开的文件描述符从控制台读取输入。对read的调用实现了从内核流向consoleread的数据通路。consoleread等待输入到达（通过中断）并在cons.buf中缓冲，将输入复制到用户空间，然后（在整行到达后）返回给用户进程。如果用户还没有键入整行，任何读取进程都将在sleep系统调用中等待。

当用户输入一个字符时，UART硬件要求RISC-V发出一个中断，从而激活xv6的陷阱处理程序。陷阱处理程序调用devintr，它查看RISC-V的scause寄存器，发现中断来自外部设备。然后它要求一个称为PLIC的硬件单元告诉它哪个设备中断了。如果是UART，devintr调用uartintr。

uartintr从UART硬件读取所有等待输入的字符，并将它们交给consoleintr；它不会等待字符，因为未来的输入将引发一个新的中断。consoleintr的工作是在cons.buf中积累输入字符，直到一整行到达。consoleintr对backspace和其他少量字符进行特殊处理。当换行符到达时，consoleintr唤醒一个等待的consoleread（如果有的话）。

一旦被唤醒，consoleread将监视cons.buf中的一整行，将其复制到用户空间，并返回（通过系统调用机制）到用户空间。

> 写不动了，这一章学着挺枯燥，也很难，涉及很多硬件，还是直接看视频吧orz ：(