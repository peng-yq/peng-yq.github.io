---
layout: post
title: "MIT 6.S081—调度（二）"
subtitle: "scheduling Part Two"
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

这部分会讨论coordination，xv6通过Sleep&Wakeup实现了coordination。

## Coordination

当你在写一个线程的代码时，有些场景需要等待一些特定的事件，或者不同的线程之间需要交互。

- 假设我们有一个Pipe，并且我正在从Pipe中读数据。但是Pipe当前又没有数据，所以我需要等待一个Pipe非空的事件。
- 类似的，假设我在读取磁盘，我会告诉磁盘控制器请读取磁盘上的特定块。这或许要花费较长的时间，尤其当磁碟需要旋转时（通常是毫秒级别），磁盘才能完成读取。而执行读磁盘的进程需要等待读磁盘结束的事件。
- 类似的，一个Unix进程可以调用wait函数。这个会使得调用进程等待任何一个子进程退出。所以这里父进程有意的在等待另一个进程产生的事件。

以上就是进程需要等待特定事件的一些例子。特定事件可能来自于I/O，也可能来自于另一个进程，并且它描述了某件事情已经发生。**Coordination是帮助我们解决这些问题并帮助我们实现这些需求的工具。Coordination是非常基础的工具，就像锁一样，在实现线程代码时它会一直出现**。

我们怎么能让进程或者线程等待一些特定的事件呢？**一种非常直观的方法是通过循环实现busy-wait，例如我们想从一个Pipe读取数据，我们就写一个循环一直等待Pipe的buffer不为空**。这个循环会一直运行直到其他的线程向Pipe的buffer写了数据。之后循环会结束，我们就可以从Pipe中读取数据并返回。实际中会有这样的代码。**如果你知道你要等待的事件极有可能在0.1微秒内发生，通过循环等待或许是最好的实现方式。通常来说在操作设备硬件的代码中会采用这样的等待方式，如果你要求一个硬件完成一个任务，并且你知道硬件总是能非常快的完成任务，这时通过一个类似的循环等待或许是最正确的方式**。

但是事件可能需要数个毫秒甚至你都不知道事件要多久才能发生，或许要10分钟其他的进程才能向Pipe写入数据，那么我们就不想在这一直循环并且浪费本可以用来完成其他任务的CPU时间。这时我们想要通过类似switch函数调用的方式出让CPU，并在我们关心的事件发生时重新获取CPU。**Coordination就是有关出让CPU，直到等待的事件发生再恢复执行**。人们发明了很多不同的Coordination的实现方式，但是与许多Unix风格操作系统一样，xv6使用的是Sleep&Wakeup这种方式。

## Sleep & Wake up

在教学视频中， Robert重写了UART的驱动代码，xv6通过下面的驱动代码从console中读写字符。

```c
static int tx_done; // has the UART finished sending?
static int tx_chan; // &tx_chan is the "wait channel"

// transmit buf[].

void uartwrite(char buf[], int n){
    acquire(&uart_tx_lock);
    
    int i = 0;
    while(i < n){
        while(tx_done == 0){
            // UART is busy sending a character
            // wait for it to interrupt;
            sleep(&tx_chan, &uart_tx_lock);
        }
        WriteReg(THR, buf[i]);
        i +=1;
        tx_done = 0;
    }
    release(&uart_tx_lock);
}
```

首先是uartwrite函数。当shell需要输出时会调用write系统调用最终走到uartwrite函数中，这个函数会在循环中将buf中的字符一个一个的向UART硬件写入。这是一种经典的设备驱动实现风格，可以在很多设备驱动中看到类似的代码。UART硬件一次只能接受一个字符的传输，而通常来说会有很多字符需要写到UART硬件。你可以向UART硬件写入一个字符，并等待UART硬件说：好的我完成了传输上一个字符并且准备好了传输下一个字符，之后驱动程序才可以写入下一个字符。**因为这里的硬件可能会非常慢，或许每秒只能传输1000个字符，所以我们在两个字符之间的等待时间可能会很长。而1毫秒在现在计算机上是一个非常非常长的时间，它可能包含了数百万条指令时间**。所以我们不想通过循环来等待UART完成字符传输，我们想通过一个更好的方式来等待。

如大多数操作系统一样，xv6也的确存在更好的等待方式。UART硬件会在完成传输一个字符后，触发一个中断。所以UART驱动中除了uartwrite函数外，还有名为uartintr的中断处理程序。这个中断处理程序会在UART硬件触发中断时由trap.c代码调用。

```c
// handle a uart interrupt, raised because input has
// arrived, or the uart is ready for more output, or
// both, called from trap.c

void uartintr(void){
    acquire(&uart_tx_lock);
    if(ReadReg(LSR) & LSR_TX_IDLE){
        // UART finished transmitting; wake up any sending thread.
        tx_done = 1;
        wakeup(&tx_chan);
    }
    release(&uart_tx_lock);
    
    // read and process incoming characters.
    while(1){
        int c = uartgetc();
        if(c == -1)
            break;
    }
}
```

中断处理程序会在最开始读取UART对应的memory mapped register，并检查其中表明传输完成的相应的标志位，也就是LSR_TX_IDLE标志位。如果这个标志位为1，代码会将tx_done设置为1。**并调用wakeup函数，这个函数会使得uartwrite中的sleep函数恢复执行（while(tx_done == 0)不满足条件），并尝试发送一个新的字符**。所以这里的机制是，如果一个线程需要等待某些事件，比如说等待UART硬件愿意接收一个新的字符，线程调用sleep函数并等待一个特定的条件。当特定的条件满足时，代码会调用wakeup函数。这里的**sleep函数和wakeup函数是成对出现的，并通过某种方式链接到一起。也就是说，如果我们调用wakeup函数，我们只想唤醒正在等待刚刚发生的特定事件的线程。所以，sleep函数和wakeup函数都带有一个叫做sleep channel的参数。我们在调用wakeup的时候，需要传入与调用sleep函数相同的sleep channel**。不过sleep和wakeup函数只是接收表示了sleep channel的64bit数值，它们并不关心这个数值代表什么。当我们调用sleep函数时，我们通过一个sleep channel表明我们等待的特定事件，当调用wakeup时我们希望能传入相同的数值来表明想唤醒哪个线程。

sleep&wakeup的一个优点是它们可以很灵活，它们不关心代码正在执行什么操作，你不用告诉sleep函数你在等待什么事件，你也不用告诉wakeup函数发生了什么事件，你只需要匹配好64bit的sleep channel就行。不过，对于sleep函数，我们需要将一个锁作为第二个参数传入。

> 需要注意的是，虽然示例程序中传输每个字符都有一个中断，UART实际上支持一次传输4或者16个字符，所以一个更有效的驱动会在每一次循环都传输16个字符给UART，并且中断也是每16个字符触发一次。更高速的设备，例如以太网卡通常会更多个字节触发一次中断。

>课堂中一个不错的提问：当从sleep函数中唤醒时，不是已经知道是来自UART的中断处理程序调用wakeup的结果吗？这样的话tx_done有些多余。
>
>Robert教授：我想你的问题也可以描述为：为什么需要通过一个循环while(tx_done == 0)来调用sleep函数？这个问题的答案适用于一个更通用的场景：**实际中不太可能将sleep和wakeup精确匹配（关键）**。并不是说sleep函数返回了，你等待的事件就一定会发生。举个例子，假设我们有两个进程同时想写UART，它们都在uartwrite函数中。可能发生这种场景，当一个进程写完一个字符之后，会进入SLEEPING状态并释放锁，而另一个进程可以在这时进入到循环并等待UART空闲下来。之后两个进程都进入到SLEEPING状态，当发生中断时UART可以再次接收一个字符，两个进程都会被唤醒，但是只有一个进程应该写入字符，所以我们才需要在sleep外面包一层while循环。实际上，你可以在XV6中的每一个sleep函数调用都被一个while循环包着。因为事实是，你或许被唤醒了，但是其他人将你等待的事件拿走了，所以你还得继续sleep。这种现象还挺普遍的。

## Lost wakeup

### Why Sleep needs Lock

我们知道wake函数的参数不需要lock，而sleep函数的参数则需要lock。假设sleep函数的参数不需要lock，只是接收任意的sleep channel作为唯一的参数，会有什么样的结果？它其实不能正常工作，我们称这个sleep实现为broken_sleep。相较于sleep，broken_sleep唯一的不同是没有获取进程的锁。很明显，uartwrite和uartintr两个函数需要使用锁来协调工作：

- 第一个原因是两个函数共享的done标志位，**任何时候我们有了共享的数据，我们需要为这个数据加上锁**。
- 另一个原因是两个函数都需要访问UART硬件，通常来说让两个线程并发的访问memory mapped register是错误的行为。

所以我们需要在两个函数中加锁来避免对于done标志位和硬件的竞争访问。现在的问题是，我们该在哪个位置加锁？在中断处理程序中较为简单，我们在最开始加锁，在最后解锁。难的是如何在uartwrite函数中加锁。一种可能是，每次发送一个字符的过程中持有锁，所以在每一次遍历buffer的起始和结束位置加锁和解锁。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRDCIRwsFLpdVshZNe4%2F-MRDJ0mD2eAZD1RMEEoy%2Fimage.png?alt=media&token=882ca03d-0f90-4bcd-bbda-69d9f7239702">

但这样肯定不能工作？一个原因是，我们能从while not done的循环退出的唯一可能是中断处理程序将done设置为1。但是**如果我们为整个代码段都加锁的话，中断处理程序就不能获取锁了，中断程序会不停“自旋”并等待锁释放**。而锁被uartwrite持有，在done设置为1之前不会释放。而done只有在中断处理程序获取锁之后才可能设置为1。

另一种实现可能是，在传输字符的最开始获取锁，因为我们需要保护共享变量done，但是在调用sleep函数之前释放锁。这样中断处理程序就有可能运行并且设置done标志位为1。之后在sleep函数返回时，再次获取锁。修改后的代码如下：

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRDCIRwsFLpdVshZNe4%2F-MRDOAEDM4aLVRx_Rm0q%2Fimage.png?alt=media&token=4d1793a9-64e0-4a51-9411-52030003d9be">

编译代码会出现下面的问题：

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRDORY2vyvR4-n_E44q%2F-MRDPpCWmgERivtXWy1q%2Fimage.png?alt=media&token=117ac381-9a79-4797-ac75-7ee001d215db">

xv6正常启动的时候会打“init starting”，但这里看来输出了一些字符之后就停住了。在前面的代码中，sleep之前释放了锁，但是在释放锁和broken_sleep之间可能会发生中断。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRDORY2vyvR4-n_E44q%2F-MRDRqIMwnqW32Rsd973%2Fimage.png?alt=media&token=cc483651-a57e-42c3-bbe3-3c692843dbef">

一旦释放了锁，当前CPU的中断会被重新打开。因为这是一个多核机器，所以中断可能发生在任意一个CPU核。在上面代码标记的位置，其他CPU核上正在执行UART的中断处理程序，并且正在acquire函数中等待当前锁释放。**所以一旦锁被释放了，另一个CPU核就会获取锁，并发现UART硬件完成了发送上一个字符，之后会设置tx_done为1，最后再调用wakeup函数，并传入tx_chan。但现在uartwrite线程还在执行并位于release和broken_sleep之间，也就是uartwrite线程还没有进入SLEEPING状态，所以中断处理程序中的wakeup并没有唤醒任何进程，因为还没有任何进程在tx_chan上睡眠。之后uartwrite线程会继续运行，调用broken_sleep，将进程状态设置为SLEEPING，保存sleep channel。但是中断已经发生了，wakeup也已经被调用了。所以这次的broken_sleep，没有人会唤醒它，因为wakeup已经发生过了。这就是lost wakeup问题**。

### How to Avoid Lost Wakeup

首先uart_tx_lock是必须要释放的，因为中断需要获取这个锁，但是我们又不能在释放锁和sleep之间留有窗口（这样会导致lost wakeup）。为了实现这个目的，我们需要将sleep函数设计的稍微复杂点。**这里的解决方法是，即使sleep函数不需要知道你在等待什么事件，它还是需要你知道你在等待什么数据，并且传入一个用来保护你在等待数据的锁**。在上面的例子中，sleep函数执行的特定条件是tx_done等于1。虽然sleep不需要知道tx_done，但是它需要知道保护这个条件的锁，也就是这里的uart_tx_lock。在调用sleep的时候，锁还被当前线程持有，之后这个锁被传递给了sleep。

> 说了那么多，似乎还是很懵逼。嗯！**Talk is cheap, Show me the code**!

首先我们来看一下proc.c中的wakeup函数：查看整个进程表单，对于每个进程首先加锁，这点很重要。之后查看进程的状态，如果进程当前是SLEEPING并且进程的channel与wakeup传入的channel相同，将进程的状态设置为RUNNABLE。最后再释放进程的锁。

> kernel/proc.c

```c
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
查看sleep函数：首先让我们回忆下，uartwrite在最开始获取了sleep的condition lock（uart_tx_lock），并且一直持有condition lock直到调用sleep函数。此时wakeup不能做任何事情，准确来说wakeup现在甚至都不能被调用，因为uartintr在调用wakeup前需要先获取condition lock，而此时condition lock一直被uartwrite所持有。因此，sleep函数中第一件事情就是释放这个锁，这样wakeup就能被调用了。当然在释放锁之后，我们会担心在这个时间点相应的wakeup会被调用并尝试唤醒当前进程，而当前进程还没有进入到SLEEPING状态。解决办法是在release锁之前，sleep会获取即将进入SLEEPING状态的进程的锁。这样wakeup在调用时会在访问p->state前获取进程的锁，这里由于sleep持有了进程的锁，所以wakeup不能在这个时间点被调用。

在持有进程锁的时候，将进程的状态设置为SLEEPING并记录sleep channel，之后再调用sched函数，这个函数中会再调用switch函数，此时sleep函数中仍然持有了进程的锁，wakeup仍然不能做任何事情。**当我们从当前线程切换走时，调度器线程中会释放前一个进程的锁。所以在调度器线程释放进程锁之后，wakeup才能终于获取进程的锁，发现它正在SLEEPING状态，并唤醒它**。此时对应进程的状态才被设置为RUNNABLE，然后最终又回到[Part One](https://peng-yq.github.io/2023/06/01/scheduling/)中调度器的工作，然后恢复进程的运行。

> kernel/sleep.c

```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
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
```

总结一下，为了解决lost wakeup问题，对sleep和wakeup定义了一些规则确保的，这些规则包括了：

- 调用sleep时需要持有condition lock，这样sleep函数才能知道相应的锁。
- sleep函数只有在获取到进程的锁p->lock之后，才能释放condition lock。
- wakeup需要同时持有两个锁才能查看进程。

这样的话，我们就不会再丢失任何一个wakeup，也就是说我们修复了lost wakeup的问题。

## Sleep & Wakeup in Pipe

前面对UART的举例中，sleep等待的场景是发生了中断并且硬件准备好了传输下一个字符。在一些其他场景，内核代码会调用sleep函数并等待其他的线程完成某些事情，例如pipe.c中的piperead函数。

> kernel/pipe.c

```c
int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(pr->killed){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```

当read系统调用最终调用到piperead函数时，pi->lock会用来保护pipe，这就是sleep函数对应的condition lock。piperead需要等待的condition是pipe中有数据，也就是pi->nwrite大于pi->nread（写入pipe的字节数大于被读取的字节数）。如果这个condition不满足，那么piperead会调用sleep函数，并等待condition发生。同时piperead会将condition lock也就是pi->lock作为参数传递给sleep函数，以确保不会发生lost wakeup。

详细解释一下上面描述的可能出现的lost wakeup：在piperead函数检查发现没有字节可以读取，到piperead函数调用sleep函数之间，另一个CPU调用了pipewrite函数。因为这样的话，另一个CPU会向pipe写入数据并在piperead进程进入SLEEPING之前调用wakeup，进而产生一次lost wakeup。

> kernel/pipe.c

```c
int
pipewrite(struct pipe *pi, uint64 addr, int n)
{
  int i;
  char ch;
  struct proc *pr = myproc();

  acquire(&pi->lock);
  for(i = 0; i < n; i++){
    while(pi->nwrite == pi->nread + PIPESIZE){  //DOC: pipewrite-full
      if(pi->readopen == 0 || pr->killed){
        release(&pi->lock);
        return -1;
      }
      wakeup(&pi->nread);
      sleep(&pi->nwrite, &pi->lock);
    }
    if(copyin(pr->pagetable, &ch, addr + i, 1) == -1)
      break;
    pi->data[pi->nwrite++ % PIPESIZE] = ch;
  }
  wakeup(&pi->nread);
  release(&pi->lock);
  return i;
}
```

在pipe的代码中，pipewrite和piperead都将sleep包装在一个while循环中。

- piperead中的循环等待pipe的缓存为非空

- pipewrite中的循环等待的是pipe的缓存未满 


之所以要将sleep包装在一个循环中，是因为可能有多个进程在读取同一个pipe。**如果一个进程向pipe中写入了一个字节，这个进程会调用wakeup进而同时唤醒所有在读取同一个pipe的进程。但是因为pipe中只有一个字节并且总是有一个进程能够先被唤醒，那怎么处理呢？sleep函数中最后一件事情就是重新获取condition lock，这样第一个被唤醒的线程会持有condition lock，而其他的线程会在锁的acquire函数中等待**。被唤醒的进程会从sleep函数中返回，之后通过检查可以发现pi->nwrite比pi->nread大1，所以进程可以从piperead的循环中退出，并读取一个字节，之后pipe缓存中就没有数据了。之后piperead函数释放锁并返回。接下来，第二个被唤醒的线程，它的sleep函数可以获取condition lock并返回，但是通过检查发现pi->nwrite等于pi->nread（因为唯一的字节已经被前一个进程读走了），所以这个线程以及其他所有的等待线程都会重新进入sleep函数。**所以这里也可以看出，几乎所有对于sleep的调用都需要包装在一个循环中，这样从sleep中返回的时候才能够重新检查condition是否还符合**。

除了sleep&wakeup之外，还有一些其他的更高级的Coordination实现方式，但sleep和wakeup更通用。

## Exit

每个进程最终都需要退出，我们需要清除进程的状态，释放栈。**在xv6中，一个进程如果退出的话，我们需要释放用户内存，释放page table，释放trapframe对象，将进程在进程表单中标为REUSABLE**。xv6有两个函数与关闭线程进程相关，第一个是exit，第二个是kill。

> kernel/proc.c

**一些解释：**

> 当子进程退出时，其PCB(Process Control Block)并不会立即被删除，而是留在系统中等待父进程调用wait或waitpid来获取子进程的退出状态。此时子进程就成为了一个僵尸进程(Zombie Process)。这是因为在Linux系统中，进程的退出状态是由父进程来获取的。当父进程调用wait或waitpid时，内核会将子进程的PCB从系统中删除，并将子进程的退出状态传递给父进程。因此，如果父进程没有调用wait或waitpid来获取子进程的退出状态，子进程的PCB就会一直留在系统中，成为僵尸进程。
>
> PCB是操作系统中用于描述进程状态和进程控制信息的数据结构。每个进程都有一个对应的PCB，用于记录进程的状态、进程所需的资源、进程调度信息等。也就是后面freeproc需要释放的资源。
>
> PCB通常包括以下信息：
>
> 1. 进程标识符(Process ID)
> 2. 进程状态(Process State)：就绪、运行、阻塞等
> 3. 程序计数器(Program Counter)：指向下一条要执行的指令
> 4. 寄存器(Register)：保存进程执行过程中的寄存器值
> 5. 进程优先级(Process Priority)
> 6. 进程所需的资源信息(Resource Information)：如打开的文件、分配的内存等
> 7. 进程调度信息(Scheduling Information)：如进程的调度队列、进程等待的事件等
>

```c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  // 关闭进程打开的所有文件，并释放当前进程的当前工作目录（cwd），细节暂时忽略
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  // we might re-parent a child to init. we can't be precise about
  // waking up init, since we can't acquire its lock once we've
  // acquired any other proc lock. so wake up init whether that's
  // necessary or not. init may miss this wakeup, but that seems
  // harmless.
  // 唤醒init进程，以便让它接管当前进程的子进程，即调用wakeup1()函数唤醒init进程
  acquire(&initproc->lock);
  wakeup1(initproc);
  release(&initproc->lock);

  // grab a copy of p->parent, to ensure that we unlock the same
  // parent we locked. in case our parent gives us away to init while
  // we're waiting for the parent lock. we may then race with an
  // exiting parent, but the result will be a harmless spurious wakeup
  // to a dead or wrong process; proc structs are never re-allocated
  // as anything else.
  // 函数获取当前进程的父进程指针original_parent（避免父进程通过reparent把这个进程的父进程设置为了initproc），并尝试获取该父进程的锁。
  // 此时，如果父进程正在退出，那么当前进程会等待父进程退出后再继续执行。
  acquire(&p->lock);
  struct proc *original_parent = p->parent;
  release(&p->lock);
  
  // we need the parent's lock in order to wake it up from wait().
  // the parent-then-child rule says we have to lock it first.
  acquire(&original_parent->lock);

  acquire(&p->lock);

  // Give any children to init.
  // 将当前进程的子进程交给init进程接管
  reparent(p);

  // Parent might be sleeping in wait().
  // 唤醒父进程
  wakeup1(original_parent);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&original_parent->lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```

整体来看，exit会释放进程的内存和page table，关闭已经打开的文件，同时父进程会从wait系统调用中唤醒。

- 首先exit函数关闭了所有已打开的文件。
- 接下来是类似的处理，进程有一个对于当前目录的记录，这个记录会随着你执行cd指令而改变。在exit过程中也需要将对这个目录的引用释放给文件系统。
- 如果一个进程要退出，但是它又有自己的子进程，接下来需要设置这些子进程的父进程为init进程。
- 之后，我们需要通过调用wakeup函数唤醒当前进程的父进程，当前进程的父进程或许正在等待当前进程退出。
- 接下来，进程的状态被设置为ZOMBIE。现在进程还没有完全释放它的资源，所以它还不能被重用。所谓的进程重用是指，我们期望在最后，进程的所有状态都可以被一些其他无关的fork系统调用复用，但是目前我们还没有到那一步。
- 现在我们还没有结束，因为我们还没有释放进程资源。我们在还没有完全释放所有资源的时候，通过调用sched函数进入到调度器线程。

到目前位置，进程的状态是ZOMBIE，并且进程不会再运行，因为调度器只会运行RUNNABLE进程。同时进程资源也并没有完全释放，如果释放了进程的状态应该是UNUSED。但是可以肯定的是进程不会再运行了，因为它的状态是ZOMBIE。所以调度器线程会决定运行其他的进程。

## Wait

如果一个进程exit了，并且它的父进程调用了wait系统调用，父进程的wait会返回，wait函数的返回表明当前进程的一个子进程退出了。

> kernel/proc.c

```c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *np;
  int havekids, pid;
  struct proc *p = myproc();

  // hold p->lock for the whole time to avoid lost
  // wakeups from a child's exit().
  acquire(&p->lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(np = proc; np < &proc[NPROC]; np++){
      // 在遍历的过程中只会有一个子进程是僵尸进程，因为只要exit了，就会被wait处理
      // 即使父进程可能出现多个子进程是僵尸进程，但同一时间只有一个
      // this code uses np->parent without holding np->lock.
      // acquiring the lock first would cause a deadlock,
      // since np might be an ancestor, and we already hold p->lock.
      // 遍历过程中会遍历到自己，而前面已经acquire了当前进程的锁了，因此这里没有获取np的锁
      if(np->parent == p){
        // np->parent can't change between the check and the acquire()
        // because only the parent changes it, and we're the parent.
        acquire(&np->lock);
        havekids = 1;
        if(np->state == ZOMBIE){
          // Found one.
          pid = np->pid;
          // 将其退出状态复制到用户空间指针addr所指向的结构体中
          // 注意这里copyout出错的情况下才会直接返回-1
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,
                                  sizeof(np->xstate)) < 0) {
            release(&np->lock);
            release(&p->lock);
            return -1;
          }
          freeproc(np);
          release(&np->lock);
          release(&p->lock);
          return pid;
        }
        release(&np->lock);
      }
    }

    // No point waiting if we don't have any children.
    // 没有子进程或进程被杀死
    if(!havekids || p->killed){
      release(&p->lock);
      return -1;
    }
    
    // Wait for a child to exit.
    sleep(p, &p->lock);  //DOC: wait-sleep
  }
}
```

> kernel/proc.c

```c
// free a proc structure and the data hanging from it,
// including user pages.
// p->lock must be held.
static void
freeproc(struct proc *p)
{
  if(p->trapframe)
    kfree((void*)p->trapframe);
  p->trapframe = 0;
  if(p->pagetable)
    proc_freepagetable(p->pagetable, p->sz);
  p->pagetable = 0;
  p->sz = 0;
  p->pid = 0;
  p->parent = 0;
  p->name[0] = 0;
  p->chan = 0;
  p->killed = 0;
  p->xstate = 0;
  p->state = UNUSED;
}
```

freeproc是关闭一个进程的最后一些步骤，**如果由正在退出的进程自己在exit函数中执行这些步骤，将会非常奇怪（确实是很奇怪，尤其自己释放trapframe和页表）**。但是freeproc没有对内核栈进行释放，是因为内核栈有guard page进行保护避免被栈溢出攻击破坏，没有必要再释放一次内核栈。不管怎样，当进程还在exit函数中运行时，任何这些资源在exit函数中释放都会很难受，所以这些资源都是由父进程释放的。

wait不仅是为了父进程方便的知道子进程退出，wait实际上也是进程退出的一个重要组成部分。**在Unix中，对于每一个退出的进程，都需要有一个对应的wait系统调用，这就是为什么当一个进程退出时，它的子进程需要变成init进程的子进程。init进程的工作就是在一个循环中不停调用wait，因为每个进程都需要对应一个wait，这样它的父进程才能调用freeproc函数，并清理进程的资源**。

当父进程完成了清理进程的所有资源，子进程的状态会被设置成UNUSED。之后，fork系统调用才能重用进程在进程表单的位置。

## Kill

Unix中的一个进程可以将另一个进程的ID传递给kill系统调用，并让另一个进程停止运行。

> kernel/proc.c

```c
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```

通过代码可以看到，kill调用基本不做任何事情（其他的Unix系统也是这样的），并不是真正的停止进程的运行。kill先扫描进程表单，找到目标进程，然后只是将进程的proc结构体中killed标志位设置为1。**如果进程正在SLEEPING状态，将其设置为RUNNABLE，以便它能够被wakeup唤醒返回并被回收**。

而目标进程运行到内核代码中能安全停止运行的位置时，会检查自己的killed标志位，如果设置为1，目标进程会自愿的执行exit系统调用。在执行系统调用之前，如果进程已经被kill了，进程会自己调用exit。在这个内核代码位置，代码并没有持有任何锁，也不在执行任何操作的过程中，所以进程通过exit退出是完全安全的。类似的，在usertrap函数的最后，也有类似的代码。在执行完系统调用之后，进程会再次检查自己是否已经被kill了。即使进程是被中断打断，这里的检查也会被执行。例如当一个定时器中断打断了进程的运行，我们可以通过检查发现进程是killed状态，之后进程会调用exit退出。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MRKwJwSQOULTxQNadvq%2F-MRNDod-w1OvPTWwuj9m%2Fimage.png?alt=media&token=3b9f19fd-394f-453c-a90d-e435df41776f">

所以kill系统调用并不是真正的立即停止进程的运行，它更像是这样：如果进程在用户空间，那么下一次它执行系统调用它就会退出，又或者目标进程正在执行用户代码，当时下一次定时器中断或者其他中断触发了，进程才会退出。所以**从一个进程调用kill，到另一个进程真正退出，中间可能有很明显的延时**。

这里有个很直观问题：如果进程不在用户空间执行，而是正在执行系统调用的过程中，然后它被kill了，我们需要做什么特别的操作吗？之所以会提出这个问题，是因为进程可能正在从console读取即将输入的字符，而你可能要明天才会输入一个字符，所以当你kill一个进程时，最好进程不是等到明天才退出。出于这个原因，**在xv6中，如果进程在SLEEPING状态时被kill了，进程会实际的退出**。

> kernel/pipe.c

```c
int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  //DOC: pipe-empty
    if(pr->killed){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); //DOC: piperead-sleep
  }
  for(i = 0; i < n; i++){  //DOC: piperead-copy
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  //DOC: piperead-wakeup
  release(&pi->lock);
  return i;
}
```

如果一个进程正在sleep状态等待从pipe中读取数据，然后它被kill了。kill函数会将其设置为RUNNABLE，之后进程会从sleep中返回，返回到循环的最开始。pipe中大概率还是没有数据，之后在piperead中，会判断进程是否被kill了（if(pr->killed)）。如果进程被kill了，那么接下来piperead会返回-1，并且返回到usertrap函数的syscall位置。之后在usertrap函数中会检查p->killed，并调用exit。所以对于SLEEPING状态的进程，如果它被kill了，它会被直接唤醒，包装了sleep的循环会检查进程的killed标志位，最后再调用exit。

同时还有一些情况，如果进程在SLEEPING状态中被kill了并不能直接退出。例如，一个进程正在更新一个文件系统并创建一个文件的过程中，进程不适宜在这个时间点退出，因为我们想要完成文件系统的操作，之后进程才能退出。

## Real World

关于kill：**xv6这是个教学用的操作系统，任何与权限相关的内容在XV6中都不存在**。在Linux或者真正的操作系统中，每个进程都有一个user id或多或少的对应了执行进程的用户，一些系统调用使用进程的user id来检查进程允许做的操作。所以在Linux中会有额外的检查，调用kill的进程必须与被kill的进程有相同的user id，否则的话，kill操作不被允许。所以，在一个分时复用的计算机上，我们会有多个用户，我们不会想要用户kill其他人的进程，这样一套机制可以防止用户误删别人的进程。

**对整个调度的总结，总体而言为了教学，xv6很简单，省去了很多东西，而现实世界复杂的多**。

xv6调度器实现了一个简单的调度策略：它依次运行每个进程。这一策略被称为轮询调度（round robin）。真实的操作系统实施更复杂的策略，例如，允许进程具有优先级。其思想是调度器将优先选择可运行的高优先级进程，而不是可运行的低优先级进程。这些策略可能变得很复杂，因为常常存在相互竞争的目标：例如，操作系统可能希望保证公平性和高吞吐量。此外，复杂的策略可能会导致意外的交互，例如优先级反转（priority inversion）和航队（convoys）。当低优先级进程和高优先级进程共享一个锁时，可能会发生优先级反转，当低优先级进程持有该锁时，可能会阻止高优先级进程前进。当许多高优先级进程正在等待一个获得共享锁的低优先级进程时，可能会形成一个长的等待进程航队；一旦航队形成，它可以持续很长时间。为了避免此类问题，在复杂的调度器中需要额外的机制。

在wakeup中扫描整个进程列表以查找具有匹配chan的进程效率低下。一个更好的解决方案是用一个数据结构替换sleep和wakeup中的chan，该数据结构包含在该结构上休眠的进程列表，例如Linux的等待队列。sleep和wakeup将该结构称为集结点（rendezvous point）或Rendez。许多线程库引用与条件变量相同的结构；在这种情况下，sleep和wakeup操作称为wait和signal。所有这些机制都有一个共同的特点：睡眠条件受到某种在睡眠过程中原子级释放的锁的保护。

wakeup的实现会唤醒在特定通道上等待的所有进程，可能有许多进程在等待该特定通道。操作系统将安排所有这些进程，它们将竞相检查睡眠条件。进程的这种行为有时被称为惊群效应（thundering herd），最好避免。大多数条件变量都有两个用于唤醒的原语：signal用于唤醒一个进程；broadcast用于唤醒所有等待进程。

信号量（Semaphores）通常用于同步。计数count通常对应于管道缓冲区中可用的字节数或进程具有的僵尸子进程数。使用显式计数作为抽象的一部分可以避免“丢失唤醒”问题：使用显式计数记录已经发生wakeup的次数。计数还避免了虚假唤醒和惊群效应问题。

终止进程并清理它们在xv6中引入了很多复杂性。在大多数操作系统中甚至更复杂，因为，例如，受害者进程可能在内核深处休眠，而展开其栈空间需要非常仔细的编程。许多操作系统使用显式异常处理机制（如longjmp）来展开栈。此外，还有其他事件可能导致睡眠进程被唤醒，即使它等待的事件尚未发生。例如，当一个Unix进程处于休眠状态时，另一个进程可能会向它发送一个signal。在这种情况下，进程将从中断的系统调用返回，返回值为-1，错误代码设置为EINTR。应用程序可以检查这些值并决定执行什么操作。Xv6不支持信号，因此不会出现这种复杂性。

xv6对kill的支持并不完全令人满意：有一些sleep循环可能应该检查p->killed。一个相关的问题是，即使对于检查p->killed的sleep循环，sleep和kill之间也存在竞争；后者可能会设置p->killed，并试图在受害者的循环检查p->killed之后但在调用sleep之前尝试唤醒受害者。如果出现此问题，受害者将不会注意到p->killed，直到其等待的条件发生。这可能比正常情况要晚一点（例如，当virtio驱动程序返回受害者正在等待的磁盘块时）或永远不会发生（例如，如果受害者正在等待来自控制台的输入，但用户没有键入任何输入）。

而如果像上面说的，在检查p->killed之后调用sleep之前唤醒受害者进程，那么接下来执行sleep就会导致进程无法进入内核，无法在usertrap中退出，而必须等待所需事件的发生再次唤醒

一个实际的操作系统将在固定时间内使用空闲列表找到自由的proc结构体，而不是allocproc中的线性时间搜索；xv6使用线性扫描是为了简单起见。

