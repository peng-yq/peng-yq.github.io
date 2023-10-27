---
layout: post
title: "MIT 6.S081—调度（一）"
subtitle: "scheduling Part One"
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
>
> 进程调度这部分看了好几天，懂了后很有成就感

任何操作系统都可能运行比CPU数量更多的进程，所以需要一个进程间分时共享CPU的方案。这种共享最好对用户进程透明。一种常见的方法是，通过将进程多路复用到硬件CPU上，使每个进程产生一种错觉，即它有自己的虚拟CPU。

## Processes & Threads

> 进程是资源隔离的单位，并不是执行单位，线程才是调度器的调度对象。区别于一个进程下的多线程，这里xv6的教学中一个进程只有一个线程。

**当一个程序运行时，操作系统会创建一个进程来运行该程序，并在该进程内创建一个或多个线程来执行程序中的代码。每个线程都会共享进程的资源，如内存、文件描述符等，但拥有自己的堆栈和程序计数器(PC)，因此可以独立执行**。在多核处理器上，操作系统可以将不同的线程分配到不同的CPU核心上，以实现并行执行。线程之间的协作和同步需要特殊的机制来确保正确性。例如，线程之间可以使用锁、条件变量等机制来共享资源和协调执行。此外，线程之间也需要避免竞态条件和死锁等问题，以确保程序的正确性和稳定性。

在操作系统中，分配CPU的最小单位是线程，是程序执行的最小单位，线程是轻量级的进程，它可以与同一进程中的其他线程共享内存空间和其他资源，因此线程之间的通信和同步比进程之间的通信和同步更加高效。线程可以独立执行，也可以与其他线程协作完成任务；而分配内存的最小单位是进程。进程之间是相互独立的，它们不能直接访问其他进程的内存空间，需要通过进程间通信机制来进行通信。

## Threads

为什么计算机需要运行多线程？可以归结为以下原因：

- 首先，人们希望他们的计算机在同一时间不是只执行一个任务。有可能计算机需要执行分时复用的任务，例如MIT的公共计算机系统Athena允许多个用户同时登陆一台计算机，并运行各自的进程。甚至在一个单用户的计算机上，你会运行多个进程，并期望计算机完成所有的任务而不仅仅只是一个任务。(PS：Robert在课上是举了这个例子，但我觉得这个是多进程，多进程和多线程还是有区别的)
- 其次，多线程可以让程序的结构变得简单。线程在有些场合可以帮助程序员将代码以简单优雅的方式进行组织，并减少复杂度。
- 最后，使用多线程可以通过并行运算，在拥有多核CPU的计算机上获得更快的处理速度。常见的方式是将程序进行拆分，并通过线程在不同的CPU核上运行程序的不同部分。如果你足够幸运的话，你可以将你的程序拆分并在4个CPU核上通过4个线程运行你的程序，同时你也可以获取4倍的程序运行速度。

如果你写了一个程序只是按顺序执行代码，那么你可以认为这个程序就是个单线程程序，这是对于线程的一种宽松的定义。虽然人们对于线程有很多不同的定义，在这里，我们认为**线程就是单个串行执行代码的单元，它只占用一个CPU并且以普通的方式一个接一个的执行指令**。

**线程还具有状态，我们可以随时保存线程的状态并暂停线程的运行，并在之后通过恢复状态来恢复线程的运行**。线程的状态包含了三个部分：

- 程序计数器（Program Counter），它表示当前线程执行指令的位置。
- 保存变量的寄存器。
- 程序的Stack。通常来说每个线程都有属于自己的Stack，Stack记录了函数调用的记录，并反映了当前线程的执行点。

多线程的并行运行主要有两个策略：

- 第一个策略是在多核处理器上使用多个CPU，每个CPU都可以运行一个线程，如果你有4个CPU，那么每个CPU可以运行一个线程。每个线程自动的根据所在CPU就有了程序计数器和寄存器。
- 第二个策略，也就是一个CPU在多个线程之间来回切换。例如xv6能够先运行一个线程，之后将线程的状态保存，再切换至运行第二个线程，然后再是第三个线程，依次类推直到每个线程都运行了一会，再回来重新执行第一个线程。

不同线程系统之间的一个主要的区别就是，线程之间是否会共享内存：

- 一种可能是你有一个地址空间，多个线程都在这一个地址空间内运行，并且它们可以看到彼此的更新。比如说共享一个地址空间的线程修改了一个变量，共享地址空间的另一个线程可以看到变量的修改。所以当多个线程运行在一个共享地址空间时，我们需要用到锁。xv6内核共享了内存，并且xv6支持内核线程的概念，对于每个用户进程都有一个内核线程来执行来自用户进程的系统调用。所有的内核线程都共享了内核内存，所以xv6的内核线程的确会共享内存。
- 另一方面，xv6还有另外一种线程。每一个用户进程都有独立的内存地址空间，并且包含了一个线程，这个线程控制了用户进程代码指令的执行。所以xv6中的用户线程之间没有共享内存，你可以有多个用户进程，但是每个用户进程都是拥有一个线程的独立地址空间。xv6中的进程不会共享内存。

在一些其他更加复杂的系统中，例如Linux，允许在一个用户进程中包含多个线程，进程中的多个线程共享进程的地址空间。当你想要实现一个运行在多个CPU核上的用户进程时，你就可以在用户进程中创建多个线程。

## Multiplexing

xv6通过在两种情况下将每个CPU从一个进程切换到另一个进程来实现多路复用（Multiplexing）。

- 第一：当进程等待设备或管道I/O完成，或等待子进程退出，或在sleep系统调用中等待时，xv6使用**睡眠（sleep）和唤醒（wakeup）机制**切换。
- 第二：xv6**周期性地强制切换以处理长时间计算而不睡眠的进程**。这种多路复用产生了每个进程都有自己的CPU的错觉，就像xv6使用内存分配器和硬件页表来产生每个进程都有自己内存的错觉一样。

实现多路复用带来了一些挑战。

- 首先，如何从一个进程切换到另一个进程？尽管上下文切换的思想很简单，但它的实现是xv6中最不透明的代码之一。
- 第二，如何以对用户进程透明的方式强制切换？xv6使用标准技术，通过定时器中断驱动上下文切换。
- 第三，许多CPU可能同时在进程之间切换，使用一个用锁方案来避免争用是很有必要的。
- 第四，进程退出时必须释放进程的内存以及其他资源，但它不能自己完成所有这一切，例如它不能在仍然使用自己内核栈的情况下释放它。
- 第五，多核机器的每个核必须记住它正在执行哪个进程，以便系统调用正确影响对应进程的内核状态。
- 最后，sleep允许一个进程放弃CPU，wakeup允许另一个进程唤醒第一个进程。需要小心避免导致唤醒通知丢失的竞争。Xv6试图尽可能简单地解决这些问题，但结果代码很复杂。

## xv6 Thread Scheduling

实现内核中的线程系统存在以下挑战：

- 第一个是**如何实现线程间的切换**。停止一个线程的运行并启动另一个线程的过程通常被称为线程调度（Scheduling），xv6为每个CPU核都创建了一个线程调度器（Scheduler）。
- 第二个挑战是当从**一个线程切换到另一个线程时，需要保存并恢复线程的状态，所以需要决定线程的哪些信息是必须保存的，并且在哪保存它们**。
- 最后一个挑战是如何**处理运算密集型线程（compute bound thread）**。对于线程切换，很多直观的实现是由线程自己自愿的保存自己的状态，再让其他的线程运行。但是如果我们有一些程序正在执行一些可能要花费数小时的长时间计算任务，这样的线程并不能自愿的出让CPU给其他的线程运行。所以这里**需要能从长时间运行的运算密集型线程撤回对于CPU的控制，将其放置于一边，稍后再运行它**。

对于运算密集型线程的处理，就是利用定时器中断。**在每个CPU核上，都存在一个硬件设备，它会定时产生中断**。xv6与其他所有的操作系统一样，将这个中断传输到了内核中。所以**即使我们正在用户空间计算π的前100万位，定时器中断仍然能在例如每隔10ms的某个时间触发，并将程序运行的控制权从用户空间代码切换到内核中的中断处理程序（注，因为中断处理程序优先级更高）**。哪怕这些用户空间进程并不配合工作（注，也就是用户空间进程一直占用CPU），内核也可以从用户空间进程获取CPU控制权。**位于内核的定时器中断处理程序，会自愿的将CPU出让（yield）给线程调度器，并告诉线程调度器说，你可以让一些其他的线程运行了**。这里的出让其实也是一种线程切换，它会保存当前线程的状态，并在稍后恢复。

> 在计算机科学中，yield是一个关键字，用于暂停当前线程的执行，并让出CPU给其他线程或进程执行。当一个线程调用yield时，它将进入等待状态，直到其他线程或进程执行完毕，并且CPU再次分配给该线程。yield通常用于多线程编程中，以实现线程间的协作和资源共享。

在执行线程调度的时候，操作系统需要能区分几类线程，这里不同的线程是由状态区分，但是实际上线程的完整状态会要复杂的多（注，线程的完整状态包含了程序计数器，寄存器，栈等等）：

- 当前在CPU上运行的线程：RUNNING，程序计数器和寄存器位于正在运行它的CPU硬件中，当线程调度器决定要运行一个RUNABLE线程时，这里涉及了很多步骤，但是其中一步是将之前保存的程序计数器和寄存器拷贝回调度器对应的CPU中
- 一旦CPU有空闲时间就想要运行在CPU上的线程: RUNABLE，因为并没有CPU与之关联，所以对于每一个RUNABLE线程，当我们将它从RUNNING转变成RUNABLE时，我们需要将它还在RUNNING时位于CPU的状态（程序计数器和寄存器）拷贝到内存中的某个位置
- 以及不想运行在CPU上的线程，因为这些线程可能在等待I/O或者其他事件: SLEEPING

## xv6 Thread Switching

> PS：这部分以及后面部分会多次描述进程/线程切换，可能会有些重复，但为了更好的理解和加深印象就不精简了

我们或许会运行多个用户空间进程，例如C compiler（CC），LS，Shell。在用户空间，每个进程有自己的内存和用户程序栈（user stack），并且当进程运行的时候，它在CPU中会有程序计数器和寄存器。当用户程序在运行时，实际上是用户进程中的一个用户线程在运行。**如果程序执行了一个系统调用或者因为响应中断走到了内核中，那么相应的用户空间状态（程序计数器、寄存器）会被保存在程序的trapframe中，同时属于这个用户程序的内核线程被激活**。涉及到的内核代码为trampoline和usertrap，前者用于在用户进程和内核切换，后者用于处理异常或中断。之后内核会运行一段时间处理系统调用或者执行中断处理程序。在处理完成之后，如果需要返回到用户空间，trapframe中保存的用户进程状态会被恢复。

> Trapframe（陷阱帧）是操作系统中的一个数据结构，用于在处理器执行中断或异常时保存当前进程的上下文信息。当处理器遇到中断或异常时，它会将当前进程的执行状态保存到Trapframe中，包括寄存器值、程序计数器、堆栈指针等。操作系统可以利用Trapframe来恢复进程的执行状态，使其能够在中断或异常处理完成后继续执行。在xv6中，当一个进程被中断或异常打断时，处理器会自动将当前进程的执行状态保存到Trapframe中，并将控制权转移到内核的中断处理程序中。

> 在xv6中，内核线程是一种特殊的进程，它们没有用户空间代码，只在内核空间中运行。内核线程被设计用于在内核中执行一些特定的任务，例如处理中断、调度进程、管理内存等。在xv6中，内核线程的实现使用了轻量级进程（LWP）的概念，每个内核线程都对应一个LWP。当内核需要执行某个任务时，它会创建一个新的内核线程，该线程会被添加到可运行队列中，等待调度器将其调度到CPU上执行。当任务完成后，内核线程会被销毁。

除了系统调用，用户进程也有可能是因为CPU需要响应类似于定时器中断走到了内核空间。在定时器中断程序中，如果xv6内核决定从一个用户进程切换到另一个用户进程，**首先在内核中第一个进程的内核线程会被切换到第二个进程的内核线程。之后再在第二个进程的内核线程中返回到用户空间的第二个进程，这里返回也是通过恢复trapframe中保存的用户进程状态完成**。

### Overview

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MPas2O4FxEvLUjjQpZq%2F-MPdb6wxbjfn3h1q5Kyl%2Fimage.png?alt=media&token=93fdc313-49d2-49e7-9b3a-f4ea54028b79">

例如xv6从CC程序的内核线程切换到LS程序的内核线程时：

1. xv6会首先会将CC程序的**内核线程的内核寄存器保存在一个context对象**中（注意区别于trapframe）
2. 因为要切换到LS程序的内核线程，那么LS程序现在的状态必然是RUNABLE，表明LS程序之前运行了一半。这同时也意味着LS程序的用户空间状态已经保存在了对应的trapframe中，更重要的是，LS程序的内核线程对应的内核寄存器也已经保存在对应的context对象中（这些都是前置条件）。所以接下来，**xv6会恢复LS程序的内核线程的context对象，也就是恢复LS程序的内核线程的寄存器**。
3. 之后**LS会继续在它的内核线程栈上，完成它的中断处理程序**（注，假设之前LS程序也是通过定时器中断触发进入的内核）。
4. 然后通过恢复LS程序的trapframe中的用户进程状态，返回到用户空间的LS程序中。
5. 最后恢复执行LS。

这里核心点在于，在xv6中，任何时候都需要经历：

1. 从一个用户进程切换到另一个用户进程，都需要从第一个用户进程接入到内核中，保存用户进程的状态并运行第一个用户进程的内核线程。
2. 再从第一个用户进程的内核线程切换到第二个用户进程的内核线程。
3. 之后第二个用户进程的内核线程暂停自己，并恢复第二个用户进程的用户寄存器。
4. 最后返回到第二个用户进程继续执行。

### More Details

当然实际的线程切换流程会复杂的多，主要是上述描述的第2步。假设我们有进程P1正在运行，进程P2是RUNABLE当前并不在运行。假设在xv6中我们有2个CPU核，这意味着在硬件层面我们有CPU0和CPU1。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MPg0UEY8_oK6jPwEU3p%2F-MPipIWU2WKegXEwqJn9%2Fimage.png?alt=media&token=be681ec4-2808-4cfa-a96f-6cba66d4c260">

我们从一个正在运行的用户空间进程切换到另一个RUNABLE但是还没有运行的用户空间进程的更完整的故事是：

1. 首先一个定时器中断强迫CPU从用户空间进程切换到内核，trampoline代码将用户寄存器保存于用户进程对应的trapframe对象中
2. 之后在内核中运行usertrap，来实际执行相应的中断处理程序。这时，CPU正在运行进程P1的内核线程和内核栈，执行内核中普通的C代码
3. 假设进程P1对应的内核线程决定它想出让CPU，最后它会调用swtch函数（switch 是C 语言关键字，因此这个函数命名为swtch 来避免冲突），这是整个线程切换的核心函数之一
4. swtch函数会保存用户进程P1对应内核线程的寄存器至context对象。所以目前为止有两类寄存器：**用户寄存器存在trapframe中，内核线程的寄存器存在context中**。

>swtch对线程没有直接的了解；它只是保存和恢复寄存器集，称为上下文（contexts）。当某个进程要放弃CPU时，**该进程的内核线程调用swtch来保存自己的上下文并返回到调度程序的上下文**。每个上下文都包含在一个struct context中，这个结构体本身包含在一个进程的struct proc或一个CPU的struct cpu中。Swtch接受两个参数：struct context *old和struct context *new。它将当前寄存器保存在old中，从new中加载寄存器，然后返回。

> kernel/proc.h

```c
// Saved registers for kernel context switches.
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

<img src="http://xv6.dgs.zone/tranlate_books/book-riscv-rev1/images/c7/p1.png">

但是，实际上swtch函数并不是直接从一个内核线程切换到另一个内核线程。**xv6中，一个CPU上运行的内核线程可以直接切换到的是这个CPU对应的调度器线程（比起小节开始那个课程中手绘的图，上图更直观一些）**。所以如果我们运行在CPU0，swtch函数会恢复之前为CPU0的调度器线程保存的寄存器和stack pointer，之后就在调度器线程的context下执行schedulder函数。在schedulder函数中会做一些清理工作，例如将进程P1设置成RUNABLE状态。之后再通过进程表单找到下一个RUNABLE进程。假设找到的下一个进程是P2（虽然也有可能找到的还是P1），schedulder函数会再次调用swtch函数，完成下面步骤：

1. 先保存自己的寄存器到调度器线程的context对象
2. 找到进程P2之前保存的context，恢复其中的寄存器
3. 因为进程P2在进入RUNABLE状态之前，如刚刚介绍的进程P1一样，必然也调用了swtch函数。所以之前的swtch函数会被恢复，并返回到进程P2所在的系统调用或者中断处理程序中
4. 当内核程序执行完成之后，trapframe中的用户寄存器会被恢复
5. 最后用户进程P2就恢复运行了

> 每一个CPU都有一个完全不同的调度器线程，调度器线程也是一种内核线程，它也有自己的context对象。任何运行在CPU上的进程，当它决定出让CPU，它都会切换到CPU对应的调度器线程，并由调度器线程切换到下一个进程。

> 关于context：每一个内核线程都有一个context对象，但是内核线程实际上有两类。
>
> 每一个用户进程有一个对应的内核线程，它的context对象保存在用户进程对应的proc结构体中。
>
> 每一个调度器线程，它也有自己的context对象，但是它却没有对应的进程和proc结构体，所以调度器线程的context对象保存在cpu结构体中。在内核中，有一个cpu结构体的数组，每个cpu结构体对应一个CPU核，每个结构体中都有一个context字段。

每一个CPU核在一个时间只会做一件事情，即每个CPU核在一个时间只会运行一个线程，它要么是运行用户进程的线程，要么是运行内核线程，要么是运行这个CPU核对应的调度器线程。线程的切换创造了多个线程同时运行在一个CPU上的假象。线程永远不会运行在多个CPU核上，线程要么运行在一个CPU核上，要么就没有运行。

## Code

### Code: mycpu and myproc

xv6通常需要指向当前进程的proc结构体的指针。在单处理器系统上，可以有一个指向当前proc的全局变量。但这不能用于多核系统，因为每个核执行的进程不同。解决这个问题的方法是基于**每个核都有自己的寄存器集，从而使用其中一个寄存器来帮助查找每个核心的信息**。

RISC-V为每个处理器核心分配了一个唯一的编号，称为hartid。在多核处理器中，每个处理器核心都有自己的hartid，这样操作系统可以通过hartid来区分不同的处理器核心，并对它们进行管理。在xv6操作系统中，为了方便管理多核处理器，它会将每个处理器核心的hartid存储在该核心的tp寄存器中。这样，当需要对某个处理器核心进行操作时，操作系统可以通过tp寄存器来索引该核心对应的CPU结构体数组中的元素，以找到正确的处理器核心。

为了确保每个处理器核心的tp寄存器始终保存着该核心的hartid，xv6采取了一些措施。首先，在处理器启动时，mstart会设置tp寄存器，此时处理器还处于机器模式。然后，在用户进程执行时，usertrapret会保存tp寄存器的值，并将其存储在trampoline page中。最后，在从用户空间进入内核时，uservec会恢复保存的tp寄存器的值。

需要注意的是，由于用户进程可能会修改tp寄存器的值，因此编译器会保证永远不会使用tp寄存器。如果RISC-V允许xv6直接读取当前hartid，那么操作系统就不需要这些复杂的措施了。但是，由于RISC-V只允许在机器模式下直接读取hartid，而在管理模式下不允许，因此xv6不得不采取这些额外的措施来确保tp寄存器的正确性。

cpuid和mycpu的返回值很脆弱：如果定时器中断并导致线程让步（yield），然后移动到另一个CPU，以前返回的值将不再正确。为了避免这个问题，**xv6要求调用者禁用中断，并且只有在使用完返回的struct cpu后才重新启用**。

> kernel/proc.c

```c
// ***********kernel/param.h*************
// #define NPROC        64  // maximum number of processes
// #define NCPU          8  // maximum number of CPUs

struct cpu cpus[NCPU];

struct proc proc[NPROC];

// ***********kernel/proc.h*************
// Per-CPU state.
struct cpu {
  struct proc *proc;          // The process running on this cpu, or null.
  struct context context;     // swtch() here to enter scheduler().
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};	

// ***********kernel/riscv.h*************
// read and write tp, the thread pointer, which holds
// this core's hartid (core number), the index into cpus[].
static inline uint64
r_tp()
{
  uint64 x;
  asm volatile("mv %0, tp" : "=r" (x) );
  return x;
}

// Must be called with interrupts disabled,
// to prevent race with process being moved
// to a different CPU.
int
cpuid()
{
  int id = r_tp();
  return id;
}

// Return this CPU's cpu struct.
// Interrupts must be disabled.
struct cpu*
mycpu(void) {
  int id = cpuid();
  struct cpu *c = &cpus[id];
  return c;
}
```

函数myproc返回当前CPU上运行进程struct proc的指针。myproc禁用中断，调用mycpu，从struct cpu中取出当前进程指针（c->proc），然后启用中断。即使启用中断，myproc的返回值也可以安全使用：如果计时器中断将调用进程移动到另一个CPU，其struct proc指针不会改变。

> kernel/proc.c

```c
// Return the current struct proc *, or zero if none.
struct proc*
myproc(void) {
  push_off();
  struct cpu *c = mycpu();
  struct proc *p = c->proc;
  pop_off();
  return p;
}
```

 

### Example Program of Thread Switching

**这一部分直接看[视频中Robert举的例子(40:59开始)](https://www.bilibili.com/video/BV19k4y1C7kA/?p=10&vd_source=1eef42fa21ab7c4e759ac52299a8dfb1)，会一行行的代码调试，对前面的进程/线程切换可以有更好的理解**。示例程序如下，这个程序中会创建两个进程，两个进程会一直运行。代码首先通过fork创建了一个子进程，然后两个进程都会进入一个死循环，并每隔一段时间生成一个输出表明程序还在运行。但是它们都不会很频繁的打印输出（注，每隔1000000次循环才打印一个输出），并且它们也不会主动出让CPU（注，因为每个进程都执行的是没有sleep的死循环），也就是有了两个运算密集型进程。并且由于设定启动的XV6只有一个CPU核，它们都运行在同一个CPU上。因此为了让这两个进程都能运行，通过计时器中断让两个进程之间能相互切换。

```c
//
// spin.c
//

#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char * argv[]){
    int pid;
    char c;

    pid = fork();
    if(pid == 0){
        c = '/';
    } else{
        printf("parent pid is %d, child is %d\n", getpid(), pid);
        c = '\\';
    }

    for(int i = 0;;i++){
        if((i % 1000000) == 0)
            write(2, &c, 1);
    }

    exit(0);
}
```

> kernel/proc.h

```c
// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

> 可以看到进程结构体中按照是否拥有锁才能访问将元素划分为了两部分：
>
> - state、parent、chan、killed、xstate和pid：这些字段表示进程的状态、父进程、等待的chan、是否被杀死、退出状态和进程ID。这些字段都是在进程切换、等待、退出等操作中使用的，因此必须先获取p->lock才能访问它们。
> - kstack、sz、pagetable、trapframe、context、ofile、cwd和name：这些字段表示进程的私有数据，只有进程本身可以访问。因此，在访问这些字段时，不需要获取p->lock。（关于这里我的理解是用户进程所对应的内核线程也是进程本身）

> kernel/trap.c

```c
// check if it's an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
int
devintr()
{
  // 读取scause寄存器的值，得到最近一次发生的异常或者中断的原因。
  uint64 scause = r_scause();

  // 首先检查 `scause` 寄存器的最高位是否为 1，如果是，说明当前是在内核态下发生了异常。
  // 接着，它检查 `scause` 寄存器的最低 8 位是否等于 9，如果是，说明当前是一个外部中断。
  if((scause & 0x8000000000000000L) &&
     (scause & 0xff) == 9){
    // this is a supervisor external interrupt, via PLIC.（通过PLC确定哪个设备发生中断）

    // irq indicates which device interrupted.（获取当前发生中断所对应的设备号）
    int irq = plic_claim();
	
    // 串口设备发生了中断
    if(irq == UART0_IRQ){
      uartintr();
    } else if(irq == VIRTIO0_IRQ){
      // 虚拟磁盘设备发生了中断
      virtio_disk_intr();
    } else if(irq){
      // 意外中断
      printf("unexpected interrupt irq=%d\n", irq);
    }

    // the PLIC allows each device to raise at most one
    // interrupt at a time; tell the PLIC the device is
    // now allowed to interrupt again.
    if(irq)
      plic_complete(irq);

    return 1;
  } else if(scause == 0x8000000000000001L){
    // 软件中断（定时器中断）
    // software interrupt from a machine-mode timer interrupt,
    // forwarded by timervec in kernelvec.S.

    if(cpuid() == 0){
      // 更新系统时间，并且唤醒等待在时钟队列上的进程。
      clockintr();
    }
    
    // acknowledge the software interrupt by clearing
    // the SSIP bit in sip.
    w_sip(r_sip() & ~2);

    return 2;
  } else {
    return 0;
  }
}
```

> kernel/trap.c

```c
//
// handle an interrupt, exception, or system call from user space.
// called from trampoline.S
//
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
   
  // 下面就是trap讲的内容了
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt.（处理定时器中断）
  if(which_dev == 2)
    yield();

  // 返回用户态
  usertrapret(); 
}
```

> kernel/proc.c

```c
// Give up the CPU for one scheduling round.
void
yield(void)
{
  struct proc *p = myproc();
  acquire(&p->lock);
  p->state = RUNNABLE;
  sched();
  release(&p->lock);
}
```

yield的代码比较简单易懂：获得进程的锁、讲进程的状态由RUNNING修改为RUNNABLE、调用sched函数，释放进程的锁。

> kernel/proc.c

```c
// Switch to scheduler.  Must hold only p->lock
// and have changed proc->state. Saves and restores
// intena because intena is a property of this
// kernel thread, not this CPU. It should
// be proc->intena and proc->noff, but that would
// break in the few places where a lock is held but
// there's no process.
void
sched(void)
{
  int intena;
  struct proc *p = myproc();

  if(!holding(&p->lock))
    panic("sched p->lock");
  if(mycpu()->noff != 1)
    panic("sched locks");
  if(p->state == RUNNING)
    panic("sched running");
  if(intr_get())
    panic("sched interruptible");

  intena = mycpu()->intena;
  // 关键：保存上下文至p->context中，并加载mycpu()->context至寄存器中
  swtch(&p->context, &mycpu()->context);
  mycpu()->intena = intena;
}
```

> kernel/swtch.s

**线程切换的核心**

采用汇编的原因：C语言中很难与寄存器交互。可以肯定的是C语言中没有方法能更改sp、ra寄存器。所以在普通的C语言中很难完成寄存器的存储和加载，唯一的方法就是在C中嵌套汇编语言。所以我们也可以在C函数中内嵌swtch中的指令，但是这跟我们直接定义一个汇编函数是一样的。或者说swtch函数中的操作是在C语言的层级之下，所以并不能使用C语言。

swtch将mycpu()->context至寄存器中，也就是调度器线程的context，查看其寄存器，最有趣的就是ra（Return Address）寄存器，因为ra寄存器保存的是当前函数的返回地址，所以调度器线程中的代码会返回到ra寄存器中的地址。通过查看kernel.asm，我们可以知道这个地址的内容是什么，正好就是scheduler函数。

```c
# Context switch
#
#   void swtch(struct context *old, struct context *new);
# 
# Save current registers in old. Load from new.	

// a0，a1分别对应第一个和第二个参数
// sd ra, 0(a0)，将寄存器ra的值存储到内存地址a0+0中
// ld ra, 0(a1)，将存储在地址a1+0处的数据加载到寄存器ra 

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

> kernel/proc.c

CPU的进程调度器scheduler，swtch(&c->context, &p->context)这一句和上面一样的道理，会回到p->context的ra寄存器的内容，即sched函数。这完全在意料之中，因为可以预期的是，将要切换到的进程之前是被定时器中断通过sched函数挂起的，并且之前在sched函数中又调用了swtch函数。

```c
// Per-CPU process scheduler.
// Each CPU calls scheduler() after setting itself up.
// Scheduler never returns.  It loops, doing:
//  - choose a process to run.
//  - swtch to start running that process.
//  - eventually that process transfers control
//    via swtch back to the scheduler.
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  
  c->proc = 0;
  for(;;){
    // Avoid deadlock by ensuring that devices can interrupt.
    intr_on();
    
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        found = 1;
      }
      release(&p->lock);
    }
#if !defined (LAB_FS)
    if(found == 0) {
      intr_on();
      asm volatile("wfi");
    }
#else
    ;
#endif
  }
}
```



### Lock

从调度的角度来说，进程的锁（p->lock）完成了两件事情。

- 首先，出让CPU涉及到很多步骤，我们**需要将进程的状态从RUNNING改成RUNABLE，我们需要将进程的寄存器保存在context对象中，并且我们还需要停止使用当前进程的栈**。所以这里至少有三个步骤，而这三个步骤需要花费一些时间。**所以锁的第一个工作就是在这三个步骤完成之前，阻止任何一个其他核的调度器线程看到当前进程（有可能其他核的调度器线程也在遍历进程数组）。锁这里确保了三个步骤的原子性**，从CPU核的角度来说，三个步骤要么全发生，要么全不发生。


- 第二，当我们开始要运行一个进程时，p->lock也有类似的保护功能。当我们要运行一个进程时，我们需要将进程的状态设置为RUNNING，我们需要将进程的context移到RISC-V的寄存器中。但是，如果在这个过程中，发生了中断，从中断的角度来说进程将会处于一个奇怪的状态。比如说进程的状态是RUNNING，但是又还没有将所有的寄存器从context对象拷贝到RISC-V寄存器中。所以，如果这时候有了一个定时器中断将会是个灾难，因为我们可能在寄存器完全恢复之前，从这个进程中切换走。而从这个进程切换走的过程中，将会保存不完整的RISC-V寄存器到进程的context对象中。**所以我们希望启动一个进程的过程也具有原子性。在这种情况下，切换到一个进程的过程中，也需要获取进程的锁以确保其他的CPU核不能看到这个进程。同时在切换到进程的过程中，还需要关闭中断（aquire会关闭中断），这样可以避免定时器中断看到还在切换过程中的进程**。


**此外，xv6不允许进程在执行switch函数的过程中，持有任何其他的锁**。所以，进程在调用switch函数的过程中，必须要持有p->lock（也就是进程对应的proc结构体中的锁），但是同时又不能持有任何其他的锁。构建一个不满足这个限制条件的场景：

我们有进程P1，P1的内核线程持有了p->lock以外的其他锁，这些锁可能是在使用磁盘，UART，console过程中持有的。之后内核线程在持有锁的时候，通过调用switch/yield/sched函数出让CPU，这会导致进程P1持有了锁，但是进程P1又不在运行。假设我们在一个只有一个CPU核的机器上，进程P1调用了switch函数将CPU控制转给了调度器线程，调度器线程发现还有一个进程P2的内核线程正在等待被运行，所以调度器线程会切换到运行进程P2。假设P2也想使用磁盘，UART或者console，它会对P1持有的锁调用acquire，这是对于同一个锁的第二个acquire调用。当然这个锁现在已经被P1持有了，所以这里的acquire并不能获取锁。假设这里是spinlock，那么进程P2会在一个循环里不停的“旋转”并等待锁被释放。但是很明显进程P2的acquire不会返回，所以即使进程P2稍后愿意出让CPU，P2也没机会这么做。之所以没机会是因为P2对于锁的acquire调用在直到锁释放之前都不会返回，而唯一锁能被释放的方式就是进程P1恢复执行并在稍后release锁，但是这一步又还没有发生，因为进程P1通过调用switch函数切换到了P2，而P2又在不停的“旋转”并等待锁被释放。这是一种死锁，它会导致系统停止运行。
