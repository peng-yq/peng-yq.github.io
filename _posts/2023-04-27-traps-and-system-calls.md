---
layout: post
title: "xv6—trap指令和系统调用"
subtitle: "[xv6] Traps and system calls"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - xv6
---

## Introduction

>*此为*[MIT 6.S081](https://www.bilibili.com/video/BV19k4y1C7kA/?spm_id_from=333.1007.top_right_bar_window_custom_collection.content.click)*课程和*[xv6-Books](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf)*的学习笔记*

有三种事件会导致中央处理器搁置普通指令的执行，并强制将控制权转移到处理该事件的特殊代码上：

- **系统调用**：当用户程序执行`ecall`指令要求内核为其做些什么时
- **异常**：（用户或内核）指令做了一些非法的事情，**例如除以零或使用无效的虚拟地址 (page fault)**
- **设备中断**：一个设备，例如当磁盘硬件完成读或写请求时，向系统表明它需要被关注

xv6中使用**trap**作为上述情况的通用术语，trap对实现安全隔离和性能非常重要。通常我们系统trap的发生是“透明”的，正如进程切换一样：**trap发生时正在执行的任何代码都需要稍后恢复，并且不需要意识到发生了任何特殊的事情**。这对于中断尤其重要，中断代码通常难以预料。

让我们来思考一下，在发生trap后，内核应该做哪些操作：

- 首先，我们需要**保存32个用户寄存器**。因为很显然我们需要恢复用户应用程序的执行，尤其是当用户程序随机的被设备中断所打断时。我们希望内核能够响应中断，之后在用户程序完全无感知的情况下再恢复用户代码的执行。所以这意味着32个用户寄存器不能被内核弄乱，但是这些寄存器又要被内核代码所使用。所以在trap之前，你必须先在某处保存这32个用户寄存器。
- **程序计数器**也需要在某个地方保存，它几乎跟一个用户寄存器的地位是一样的，我们需要能够在用户程序运行中断的位置继续执行用户程序。
- 我们需要**将mode改成supervisor mode**，因为我们想要使用内核中的各种各样的特权指令。
- SATP寄存器现在正指向user page table，而user page table只包含了用户程序所需要的内存映射和一两个其他的映射，它并没有包含整个内核数据的内存映射。所以在运行内核代码之前，我们需要**将SATP指向kernel page table**。
- 我们需要将**堆栈寄存器指向位于内核的一个地址**，因为我们需要一个堆栈来调用内核的C函数。
- 一旦我们设置好了，并且所有的硬件状态都适合在内核中使用， 我们需要**跳入内核的C代码**。

**关于supervisor mode**

supervisor mode所获得的权限是有限的**。在supervisor mode，可以读写控制寄存器了：读写SATP寄存器，也就是page table的指针；STVEC，也就是处理trap的内核指令地址；SEPC，保存当发生trap时的程序计数器；SSCRATCH等等。另一件事情supervisor mode可以做的是，它可以使用PTE_U标志位为0的PTE**。当PTE_U标志位为1的时候，表明用户代码可以使用这个页表；如果这个标志位为0，则只有supervisor mode可以使用这个页表。我们接下来会看一下为什么这很重要。这两点就是supervisor mode可以做的事情，除此之外就不能再干别的事情了。

此外，supervisor mode中的代码并不能读写任意物理地址。在supervisor mode中，就像普通的用户代码一样，也需要通过page table来访问内存。**如果一个虚拟地址并不在当前由SATP指向的page table中，又或者SATP指向的page table中PTE_U=1，那么supervisor mode不能使用那个地址**。

虽然三种trap类型之间的共性表明内核可以用一个代码路径处理所有trap，但对于三种不同的情况：**来自用户空间的trap、来自内核空间的trap和定时器中断，分别使用单独的程序集向量和Ctrap处理程序更加方便**。

## RISC-V trap machinery

**每个RISC-V CPU都有一组控制寄存器，内核通过向这些寄存器写入内容来告诉CPU如何处理trap，内核可以读取这些寄存器来明确已经发生的trap**。RISC-V文档包含了完整的内容，riscv.h\(kernel/riscv.h)包含在xv6中使用到的内容的定义，以下是最重要的一些寄存器概述：

- `stvec`：内核在这里写入其trap处理程序的地址，RISC-V跳转到这里处理trap
- `sepc`：当发生trap时，RISC-V会在这里保存程序计数器`pc`（因为`pc`会被`stvec`覆盖）。`sret`（从trap返回）指令会将`sepc`复制到`pc`，内核可以写入`sepc`来控制`sret`的去向
- `scause`： RISC-V在这里放置一个描述trap原因的数字
- `sscratch`：内核在这里放置了一个值，这个值在trap处理程序一开始就会派上用场
- `sstatus`：其中的**SIE**位控制设备中断是否启用。如果内核清空**SIE**，RISC-V将推迟设备中断，直到内核重新设置**SIE**。**SPP**位指示trap是来自用户模式还是管理模式，并控制`sret`返回的模式

上述寄存器都用于在**管理模式下处理trap，在用户模式下不能读取或写入**。在**机器模式下处理trap有一组等效的控制寄存器，xv6仅在计时器中断的特殊情况下使用它们**。

**多核芯片上的每个CPU都有自己的这些寄存器集，并且在任何给定时间都可能有多个CPU在处理trap**。

当需要强制执行trap时，RISC-V硬件对所有trap类型（计时器中断除外）执行以下操作：

1. 如果trap是设备中断，并且状态**SIE**位被清空，则不执行以下任何操作。
2. 清除**SIE**以禁用中断。
3. 将`pc`复制到`sepc`。
4. 将当前模式（用户或管理）保存在状态的**SPP**位中。
5. 设置`scause`以反映产生trap的原因。
6. 将模式设置为管理模式。
7. 将`stvec`复制到`pc`。
8. 在新的`pc`上开始执行。

请注意，**CPU不会切换到内核页表，不会切换到内核栈，也不会保存除`pc`之外的任何寄存器**。内核软件必须执行这些任务，CPU在trap期间执行尽可能少量工作的一个原因是为软件提供灵活性，例如，一些操作系统在某些情况下不需要页表切换，这可以提高性能。

## Traps from user space

**来自用户代码的trap比来自内核的trap更具挑战性，因为satp指向不映射内核的用户页表，栈指针可能包含无效甚至恶意的值**。

当用户发起系统调用的时候会发生什么？以在shell中调用write系统调用为例。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKFsfImgYCtnwA1d2hO%2F-MKHxleUqYy-y0mrS48w%2Fimage.png?alt=media&token=ab7c66bc-cf61-4af4-90fd-1fefc96c7b5f">

从Shell的角度来说，这就是个Shell代码中的C函数调用，但是实际上，write通过执行ECALL指令来执行系统调用。ECALL指令会切换到具有supervisor mode的内核中：

- 内核中执行的第一个指令是一个由汇编语言写的函数，叫做uservec (kernel/trampoline.s)
- 之后，在uservec汇编函数中，代码执行跳转到了由C语言实现的函数usertrap (kernel/trap.c) 中
- 在usertrap这个C函数中，我们执行了一个叫做syscall (kernel/syscall.c) 的函数
- syscall会在一个函数表单中，根据传入的代表系统调用的数字进行查找，并在内核中执行具体实现了系统调用功能的函数。对于我们来说，这个函数就是sys_write
- sys_write会将要显示数据输出到console上，当它完成了之后，它会返回给syscall函数
- 因为我们现在相当于在ECALL之后中断了用户代码的执行，为了用户空间的代码恢复执行，需要做一系列的事情。在syscall函数中，会调用一个函数叫做usertrapret  (kernel/trap.c) ，这个函数完成了部分方便在C代码中实现的返回到用户空间的工作
- 除此之外，最终还有一些工作只能在汇编语言中完成。这部分工作通过汇编语言实现，并且存在于trampoline.s文件中的userret函数中
- 最终，在这个汇编函数中会调用机器指令返回到用户空间，并且恢复ECALL之后的用户程序的执行

> xv6 book中这一小节的描述看的让人比较混乱，建议看[教学视频](https://www.bilibili.com/video/BV19k4y1C7kA?p=5&vd_source=1eef42fa21ab7c4e759ac52299a8dfb1)，从代码和程序运行时去讲解trap from user space

##  Code: Calling system calls

第2章以initcode.S调用exec系统调用（user/initcode.S:11）结束，让我们看看用户调用是如何在内核中实现exec系统调用的。**用户代码将exec需要的参数放在寄存器a0和a1中，并将系统调用号放在a7中。系统调用号与syscalls数组中的条目相匹配，syscalls数组是一个函数指针表（kernel/syscall.c:108）。ecall指令陷入(trap)到内核中，执行uservec、usertrap和syscall**。

```c
// kernel/syscall.c

static uint64 (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

syscall函数从p->trapframe->a7检索系统调用号，并用它索引到syscalls中，对于第一次系统调用，a7中的内容是SYS_exec，导致了对系统调用接口函数sys_exec的调用。当系统调用接口函数返回时，syscall将其返回值记录在p->trapframe->a0中。这将导致原始用户空间对exec()的调用返回该值，因为RISC-V上的C调用约定将返回值放在a0中。系统调用通常返回负数表示错误，返回零或正数表示成功。如果系统调用号无效，syscall打印错误并返回-1。

## Code: System call arguments

**内核中的系统调用接口需要找到用户代码传递的参数，这些参数被放置在RISC-V C调用所约定的地方：寄存器**。内核trap代码将用户寄存器保存到当前进程的trap框架中，内核代码可以在那里找到它们。**函数artint、artaddr和artfd从trap框架中检索第n个系统调用参数并以整数、指针或文件描述符的形式保存。他们都调用argraw来检索相应的保存的用户寄存器**。

```c
// kernel/syscall.c

static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}

// Fetch the nth 32-bit system call argument.
int
argint(int n, int *ip)
{
  *ip = argraw(n);
  return 0;
}

// Retrieve an argument as a pointer.
// Doesn't check for legality, since
// copyin/copyout will do that.
int
argaddr(int n, uint64 *ip)
{
  *ip = argraw(n);
  return 0;
}

// Fetch the nth word-sized system call argument as a null-terminated string.
// Copies into buf, at most max.
// Returns string length if OK (including nul), -1 if error.
int
argstr(int n, char *buf, int max)
{
  uint64 addr;
  if(argaddr(n, &addr) < 0)
    return -1;
  return fetchstr(addr, buf, max);
}
```

有些系统调用传递指针作为参数，内核必须使用这些指针来读取或写入用户内存。例如：exec系统调用传递给内核一个指向用户空间中字符串参数的指针数组。这些指针带来了两个挑战：

- **首先，用户程序可能有缺陷或恶意，可能会传递给内核一个无效的指针，或者一个旨在欺骗内核访问内核内存而不是用户内存的指针**
- **其次，xv6内核页表映射与用户页表映射不同，因此内核不能使用普通指令从用户提供的地址加载或存储**

内核实现了安全地将数据传输到用户提供的地址和从用户提供的地址传输数据的功能，fetchstr是一个例子（kernel/syscall.c:25）。文件系统调用，如exec，使用fetchstr从用户空间检索字符串文件名参数。fetchstr调用copyinstr来完成这项困难的工作。

```c
// kernel/syscall.c

// Fetch the nul-terminated string at addr from the current process.
// Returns length of string, not including nul, or -1 for error.
int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  int err = copyinstr(p->pagetable, buf, addr, max);
  if(err < 0)
    return err;
  return strlen(buf);
}
```
copyinstr（kernel/vm.c:406）从用户页表页表中的虚拟地址srcva复制max字节到dst。它使用walkaddr（它又调用walk）在软件中遍历页表，以确定srcva的物理地址pa0。由于内核将所有物理RAM地址映射到同一个内核虚拟地址，copyinstr可以直接将字符串字节从pa0复制到dst。**walkaddr（kernel/vm.c:95）检查用户提供的虚拟地址是否为进程用户地址空间的一部分，因此程序不能欺骗内核读取其他内存**。一个类似的函数copyout，将数据从内核复制到用户提供的地址。

```c
// kernel/vm.c

// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  uint64 n, va0, pa0;
  int got_null = 0;

  while(got_null == 0 && max > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > max)
      n = max;

    char *p = (char *) (pa0 + (srcva - va0));
    while(n > 0){
      if(*p == '\0'){
        *dst = '\0';
        got_null = 1;
        break;
      } else {
        *dst = *p;
      }
      --n;
      --max;
      p++;
      dst++;
    }

    srcva = va0 + PGSIZE;
  }
  if(got_null){
    return 0;
  } else {
    return -1;
  }
}
```

## Traps from kernel space

xv6根据执行的是用户代码还是内核代码，对CPUtrap寄存器的配置有所不同。当在CPU上执行内核代码时，内核将stvec指向kernelvec(kernel/kernelvec.S:10)的汇编代码。由于xv6已经在内核中，kernelvec可以依赖于设置为内核页表的satp，以及指向有效内核栈的栈指针。kernelvec保存所有寄存器，以便被中断的代码最终可以不受干扰地恢复。

kernelvec将寄存器保存在被中断的内核线程的栈上，这是有意义的，因为寄存器值属于该线程。如果陷阱导致切换到不同的线程，那这一点就显得尤为重要——在这种情况下，陷阱将实际返回到新线程的栈上，将被中断线程保存的寄存器安全地保存在其栈上。

Kernelvec在保存寄存器后跳转到kerneltrap(kernel/trap.c:134)。kerneltrap为两种类型的陷阱做好了准备：设备中断和异常。它调用devintr(kernel/trap.c:177)来检查和处理前者。如果陷阱不是设备中断，则必定是一个异常，内核中的异常将是一个致命的错误；内核调用panic并停止执行。

如果由于计时器中断而调用了kerneltrap，并且一个进程的内核线程正在运行（而不是调度程序线程），kerneltrap会调用yield，给其他线程一个运行的机会。在某个时刻，其中一个线程会让步，让我们的线程和它的kerneltrap再次恢复。

当kerneltrap的工作完成后，它需要返回到任何被陷阱中断的代码。因为一个yield可能已经破坏了保存的sepc和在sstatus中保存的前一个状态模式，因此kerneltrap在启动时保存它们。它现在恢复这些控制寄存器并返回到kernelvec(kernel/kernelvec.S:48)。kernelvec从栈中弹出保存的寄存器并执行sret，将sepc复制到pc并恢复中断的内核代码。

值得思考的是，如果内核陷阱由于计时器中断而调用yield，陷阱返回是如何发生的。

当CPU从用户空间进入内核时，xv6将CPU的stvec设置为kernelvec；您可以在usertrap(kernel/trap.c:29)中看到这一点。内核执行时有一个时间窗口，但stvec设置为uservec，在该窗口中禁用设备中断至关重要。幸运的是，RISC-V总是在开始设置陷阱时禁用中断，xv6在设置stvec之前不会再次启用中断。

## Page-fault exceptions

> 下面笔记仅对相关功能或技术进行简单原理介绍，更多详细内容见[教学视频](https://www.bilibili.com/video/BV19k4y1C7kA?p=7)或[视频图文翻译](https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec08-page-faults-frans/8.3-zero-fill-on-demand)

xv6对page fault的响应相当简单： 如果用户空间中发生page fault，内核将终止故障进程。如果内核中发生page fault，则内核会崩溃。而现实世界真正的操作系统通常以更“有趣”的方式做出反应。

> 也就是说xv6对下面功能/技术都没有实现

**当CPU无法将虚拟地址转换为物理地址时，CPU会生成page fault异常**。当发生page fault时，内核需要什么样的信息才能够响应page fault：

- **出错的虚拟地址，或者是触发page fault的内存地址**。**当出现page fault的时候，xv6内核会打印出错的虚拟地址，并且这个地址会被保存在stval寄存器中**。所以，当一个用户应用程序触发了page fault，page fault会使用trap机制，将程序运行切换到内核，同时也会将出错的地址存放在stval寄存器中。

- **出错的原因类型**。我们或许想要对不同场景的page fault有不同的响应，不同的场景是指：

  - load指令触发的page fault

  - store指令触发的page fault

  - jump指令触发的page fault

  page fault的原因存在scause寄存器中，其中总共有3个类型的原因与page fault相关，分别是读、写和指令。ECALL进入到supervisor mode对应的是8，这是我们在上节课中应该看到的SCAUSE值。基本上来说，page fault和其他的异常使用与系统调用相同的trap机制来从用户空间切换到内核空间。如果是因为page fault触发的trap机制并且进入到内核空间，stval寄存器和scause寄存器都会有相应的值。
  
- **触发page fault的指令的地址（程序计数器值）**。这个地址存放在sepc（Supervisor Exception Program Counter）寄存器中，并同时会保存在trapframe->epc中。

### cow fork

例如，许多内核使用page fault来实现copy on write (COW) fork。

>COW Fork是指Copy-on-Write Fork，即复制时写入技术。COW Fork允许程序在启动新进程时，通过**共享父进程的内存来节省资源**。
>
>在COW Fork中，当父进程创建一个子进程时，**子进程会继承父进程的所有内存页表。在父进程或子进程中对内存进行修改时，会产生一个副本，而不是直接修改原始的内存页面**。这样，子进程就可以修改自己的副本，而不会影响到父进程或其他子进程的内存。
>
>COW Fork是一种非常有效的内存管理技术，它减少了内存使用，提高了程序的执行速度和效率。它也是进程间通信和虚拟内存管理的基石。

由page fault驱动的COW fork可以使父级和子级安全地共享物理内存，COW fork中的基本计划是让父子最初共享所有物理页面，但将它们映射为只读。因此，当子级或父级执行存储指令时，RISC-V CPU引发page fault异常。为了响应此异常，内核复制了包含错误地址的页面。它在子级的地址空间中映射一个权限为读/写的副本，在父级的地址空间中映射另一个权限为读/写的副本。更新页表后，内核会在导致故障的指令处恢复故障进程的执行。由于内核已经更新了相关的PTE以允许写入，所以错误指令现在将正确执行。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjvLvMe8CobFN8-afp%2Fimage.png?alt=media&token=fa092f13-afdf-4891-822d-7cab88f2872f">

COW策略对fork很有效，因为通常子进程会在fork之后立即调用exec，用新的地址空间替换其地址空间。在这种常见情况下，子级只会触发很少的页面错误，内核可以避免拷贝父进程内存完整的副本。此外，COW fork是透明的：无需对应用程序进行任何修改即可使其受益。

**发生COW时，实际是向一个只读的页面进行写操作，内核如何能分辨现在是一个copy-on-write fork的场景，而不是应用程序在向一个正常的只读地址写数据？**

内核必须要能够识别copy-on-write场景，几乎所有的page table硬件都支持了这一点。下图是一个常见的多级page table，对于PTE的标志位，最后两位RSW保留给supervisor software使用，supervisor softeware指的就是内核。内核可以随意使用这两个bit位。所以可以做的一件事情就是，将bit8标识为当前是一个copy-on-write page。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMk2SxJ_cq94IMH6I6W%2Fimage.png?alt=media&token=e0dad55b-5c74-4da0-a3b0-a89d462eda58">

关于COW还有个细节需要注意。目前在xv6中，除了trampoline page外，一个物理内存page只属于一个用户进程。trampoline page永远也不会释放，所以也不是什么大问题。但是对于这里的物理内存page，现在有多个用户进程或者说多个地址空间都指向了相同的物理内存page，举个例子，当父进程退出时我们需要更加的小心，因为我们要判断是否能立即释放相应的物理page。如果有子进程还在使用这些物理page，而内核又释放了这些物理page，我们将会出问题。那么现在释放内存page的依据是什么呢？

**我们需要对于每一个物理内存page的引用进行计数，当我们释放虚拟page时，我们将物理内存page的引用数减1，如果引用数等于0，那么我们就能释放物理内存page**。

### lazy allocation

除COW fork以外，页表和页面错误的结合还开发出了广泛有趣的可能性。另一个广泛使用的特性叫做惰性分配——lazy allocation，使用lazy allocation技术会在进程访问虚拟内存页时触发page fault，并通过动态分配物理内存来满足进程的内存需求

>传统的内存分配方式会在进程启动时为进程分配一大块连续的内存，这样会浪费大量的内存资源，因为很多内存并没有被使用到。**lazy allocation将内存分配推迟到实际需要使用内存时**。
>
>当进程需要内存时，系统会为进程分配内存页，这些页可能并不连续，但是操作系统会标记它们为该进程的虚拟内存页。**当进程访问虚拟内存页时，操作系统会在需要时将虚拟内存页映射到物理内存中的实际页，然后将实际的物理内存页标记为已使用**。这种方式可以避免浪费内存资源，因为只有真正需要使用内存时才会分配实际的物理内存。
>
>尽管lazy allocation可以使内存使用效率更高，但是它也需要更多的系统开销来维护虚拟内存和物理内存之间的映射关系。

### zero fill on demand

当你查看一个用户程序的地址空间时，存在.text区域，.data区域，同时还有一个.bss区域。当编译器在生成二进制文件时，编译器会填入这三个区域：.text区域是程序的指令，.data区域存放的是初始化了的全局变量，.bss包含了未被初始化或者初始化为0的全局变量。
在一个正常的操作系统中，如果执行exec，exec会申请地址空间，里面会存放text和data。因为bss里面保存了未被初始化的全局变量，这里或许有许多许多个page，但是所有的page内容都为0。通常可以调优的地方是，**我有如此多的内容全是0的page，在物理内存中，我只需要分配一个page，这个page的内容全是0。然后将所有虚拟地址空间的全0的page都map到这一个物理page上**，这样至少在程序启动的时候能节省大量的物理内存分配。

当然这里的mapping需要非常的小心，我们不能允许对于这个page执行写操作，因为所有的虚拟地址空间page都期望page的内容是全0，所以这里的PTE都是只读的。之后在某个时间点，应用程序尝试写bss中的一个page时，比如说需要更改一两个变量的值，我们会得到page fault。那么，对于这个特定场景中的page fault我们该做什么呢？**假设store指令发生在BSS最顶端的page中。我们想要做的是，在物理内存中申请一个新的内存page，将其内容设置为0，因为我们预期这个内存的内容为0。之后我们需要更新这个page的mapping关系，首先PTE要设置成可读可写，然后将其指向新的物理page**。这里相当于更新了PTE，之后我们可以重新执行指令。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MMidiGrWWP5Do6c9hwf%2F-MMjR5Fl20ATXIG70FFk%2Fimage.png?alt=media&token=6d3c6a13-7fa4-4ce6-b569-ceec31fa9694">

### virtual memory

利用page fault的另一个广泛使用的功能是virtual memory。**如果应用程序需要比可用物理RAM更多的内存，内核可以换出一些页面： 将它们写入存储设备 (如磁盘)，并将它们的PTE标记为无效。如果应用程序读取或写入被换出的页面，则CPU将触发page fault。然后内核可以检查故障地址。如果该地址属于磁盘上的页面，则内核分配物理内存页面，将该页面从磁盘读取到该内存，将PTE更新为有效并引用该内存，然后恢复应用程序**。为了给页面腾出空间，内核可能需要换出另一个页面。此功能不需要对应用程序进行更改，并且如果应用程序具有引用的地址 (即，它们在任何给定时间仅使用其内存的子集)，则该功能可以很好地工作。

###  memory mapped files

**memory mapped files的核心思想是，将完整或者部分文件加载到内存中，这样就可以通过内存地址相关的load或者store指令来操纵文件**。为了支持这个功能，一个现代的操作系统会提供一个叫做mmap的系统调用。这个系统调用会接收一个虚拟内存地址（VA），长度（len），protection，一些标志位，一个打开文件的文件描述符，和偏移量（offset）。这里的语义就是，从文件描述符对应的文件的偏移量的位置开始，映射长度为len的内容到虚拟内存地址VA，同时我们需要加上一些保护，比如只读或者读写。

假设文件内容是读写并且内核实现mmap的方式是eager方式（不过大部分系统都不会这么做），内核会从文件的offset位置开始，将数据拷贝到内存，设置好PTE指向物理内存的位置。之后应用程序就可以使用load或者store指令来修改内存中对应的文件内容。当完成操作之后，会有一个对应的unmap系统调用，参数是虚拟地址（VA），长度（len）。来表明应用程序已经完成了对文件的操作，在unmap时间点，我们需要将dirty block (PTE中dirty bit为1) 写回到文件中。

当然，在任何聪明的内存管理机制中，所有的这些都是以lazy的方式实现。**你不会立即将文件内容拷贝到内存中，而是先记录一下这个PTE属于这个文件描述符。相应的信息通常在VMA结构体中保存，VMA全称是Virtual Memory Area**。例如对于这里的文件f，会有一个VMA，在VMA中我们会记录文件描述符，偏移量等等，这些信息用来表示对应的内存虚拟地址的实际内容在哪，这样当我们得到一个位于VMA地址范围的page fault时，内核可以从磁盘中读数据，并加载到内存中。

