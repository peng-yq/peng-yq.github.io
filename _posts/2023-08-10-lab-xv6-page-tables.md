---
layout: post
title: "【MIT 6.S081】Lab 3: page tables"
subtitle: "page tables"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - xv6
---

Before you start coding, read Chapter 3 of the [xv6 book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf), and related files:

- `kern/memlayout.h`, which captures the layout of memory.
- `kern/vm.c`, which contains most virtual memory (VM) code.
- `kernel/kalloc.c`, which contains code for allocating and freeing physical memory.

It may also help to consult the [RISC-V privileged architecture manual](https://github.com/riscv/riscv-isa-manual/releases/download/Ratified-IMFDQC-and-Priv-v1.11/riscv-privileged-20190608.pdf).

## Speed up system calls (easy)

Some operating systems (e.g., Linux) speed up certain system calls by sharing data in a read-only region between userspace and the kernel. This eliminates the need for kernel crossings when performing these system calls. To help you learn how to insert mappings into a page table, your first task is to implement this optimization for the `getpid()` system call in xv6.

这个实验的目的是为了理解操作系统为了加速某些系统调用，通过在用户态和内核态之间共享一个可读区域。

When each process is created, map one read-only page at USYSCALL (a VA defined in `memlayout.h`). At the start of this page, store a `struct usyscall` (also defined in `memlayout.h`), and initialize it to store the PID of the current process. For this lab, `ugetpid()` has been provided on the userspace side and will automatically use the USYSCALL mapping. You will receive full credit for this part of the lab if the `ugetpid` test case passes when running `pgtbltest`.

当每个进程创建时，都会映射一个只读的页USYSCALL（虚拟地址），这个页的初始会存储一个usyscall的字段，用于存储当前进程的pid。

```c
#define USYSCALL (PHYSTOP - PGSIZE)

struct usyscall{
    int pid;
};
```

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
  // 进程的结构体中需要加上usyscall字段
  struct usyscall *usyscall;   // data page for usyscall
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

Some hints:

- You can perform the mapping in `proc_pagetable()` in `kernel/proc.c`.

需要在proc_pagetable()这个函数中对USYSCALL进行映射。

```c
// Create a user page table for a given process,
// with no user memory, but with trampoline pages.
pagetable_t
proc_pagetable(struct proc *p)
{
  pagetable_t pagetable;

  // An empty page table.
  pagetable = uvmcreate();
  if(pagetable == 0)
    return 0;

  // map the trampoline code (for system call return)
  // at the highest user virtual address.
  // only the supervisor uses it, on the way
  // to/from user space, so not PTE_U.
  if(mappages(pagetable, TRAMPOLINE, PGSIZE,
              (uint64)trampoline, PTE_R | PTE_X) < 0){
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the trapframe just below TRAMPOLINE, for trampoline.S.
  if(mappages(pagetable, TRAPFRAME, PGSIZE,
              (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  // map the usyscall just below TRAMPOFRAME, for trampoline.S.
  // 这个页需要设置PTE_U为，使得用户态可以访问
  if(mappages(pagetable, USYSCALL, PGSIZE,
              (uint64)(p->usyscall), PTE_R | PTE_U) < 0){
    uvmunmap(pagetable, TRAPFRAME, 1, 0);
    uvmunmap(pagetable, TRAMPOLINE, 1, 0);
    uvmfree(pagetable, 0);
    return 0;
  }

  return pagetable;
}
```

- Choose permission bits that allow userspace to only read the page.
- You may find that `mappages()` is a useful utility.
- Don't forget to allocate and initialize the page in `allocproc()`.

创建进程时，需要在进程表中分配未使用的进程，并给相关的元素（trapframe、usyscall）分配物理页。

```c
// Look in the process table for an UNUSED proc.
// If found, initialize state required to run in the kernel,
// and return with p->lock held.
// If there are no free procs, or a memory allocation fails, return 0.
static struct proc*
allocproc(void)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == UNUSED) {
      goto found;
    } else {
      release(&p->lock);
    }
  }
  return 0;

found:
  p->pid = allocpid();

  // Allocate a trapframe page.
  if((p->trapframe = (struct trapframe *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // Allocate a usyscall page.
  if((p->usyscall = (struct usyscall *)kalloc()) == 0){
    release(&p->lock);
    return 0;
  }

  // An empty user page table.
  p->pagetable = proc_pagetable(p);
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }

  // Set up new context to start executing at forkret,
  // which returns to user space.
  memset(&p->context, 0, sizeof(p->context));
  p->context.ra = (uint64)forkret;
  p->context.sp = p->kstack + PGSIZE;

  return p;
}
```

- Make sure to free the page in `freeproc()`.

同样的，在释放进程的时候，也要释放前面分配的物理页。

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
  if(p->usyscall)
    proc_freepagetable(p->usyscall, p->sz);
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

## Print a page table (easy)

To help you learn about RISC-V page tables, and perhaps to aid future debugging, your first task is to write a function that prints the contents of a page table.

Define a function called `vmprint()`. It should take a `pagetable_t` argument, and print that pagetable in the format described below. Insert `if(p->pid==1) vmprint(p->pagetable)` in exec.c just before the `return argc`, to print the first process's page table. You receive full credit for this assignment if you pass the `pte printout` test of `make grade`.

这里按照提示在exec.c中的`return agrc`代码前加上`if(p->pid==1) vmprint(p->pagetable)`，即当进程号为1时，打印其pte。

Now when you start xv6 it should print output like this, describing the page table of the first process at the point when it has just finished `exec()`ing `init`:

```
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
.. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
.. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
.. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
.. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
.. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
.. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
.. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000  
```

The first line displays the argument to `vmprint`. After that there is a line for each PTE, including PTEs that refer to page-table pages deeper in the tree. Each PTE line is indented by a number of `" .."` that indicates its depth in the tree. Each PTE line shows the PTE index in its page-table page, the pte bits, and the physical address extracted from the PTE. Don't print PTEs that are not valid. In the above example, the top-level page-table page has mappings for entries 0 and 255. The next level down for entry 0 has only index 0 mapped, and the bottom-level for that index 0 has entries 0, 1, and 2 mapped.

Your code might emit different physical addresses than those shown above. The number of entries and the virtual addresses should be the same.

这里也说了显示的物理地址可能不一样，但虚拟地址和pte number是一样的，这也显示了虚拟地址的优势，开发者只需关注虚拟地址即可，而具体的物理地址则交给mmu。

Some hints:

- You can put `vmprint()` in `kernel/vm.c`.

所有与页表相关的函数都在vm.c中定义，因此我们也需要把vmprint的定义放到vm.c中。

- Use the macros at the end of the file kernel/riscv.h.

叫我们使用pagetable_t，一个64位的物理地址。

- The function `freewalk` may be inspirational.

参考freewalk函数的实现，用于释放页表页，还是比较好理解的，使用递归的思想。

```c
// Recursively free page-table pages.
// All leaf mappings must already have been removed.
void
freewalk(pagetable_t pagetable)
{
  // there are 2^9 = 512 PTEs in a page table.
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V) && (pte & (PTE_R|PTE_W|PTE_X)) == 0){
      // (pte & (PTE_R|PTE_W|PTE_X)) == 0非叶子节点这三位必有一个是1
      // this PTE points to a lower-level page table.
      uint64 child = PTE2PA(pte);
      freewalk((pagetable_t)child);
      pagetable[i] = 0;
    } else if(pte & PTE_V){
      panic("freewalk: leaf");
    }
  }
  kfree((void*)pagetable);
}
```

- Define the prototype for `vmprint` in kernel/defs.h so that you can call it from exec.c.

需要在defs.h中声明vmprint函数。

- Use `%p` in your printf calls to print out full 64-bit hex PTEs and addresses as shown in the example.

vmprint实现：

```c
void
vmprint(pagetable_t pagetable, uint64 depth){
  // page table 0x0000000087f6e000
  // ..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
  // .. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
  // .. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
  // .. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
  // .. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
  // ..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
  // .. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
  // .. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
  // .. .. ..511: pte 0x0000000020001c0b pa 0x0000000080007000
  char* string[] = {
    [0] = "..",
    [1] = "....",
    [2] = "......"
  };
  char* buf = string[depth];
  if(depth == 0)
    printf("page table %p\n", pagetable);
  if(depth > 2)
    return;
  for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if((pte & PTE_V)){
      printf("%s%d: pte %p pa %p\n", buf, i, pte, PTE2PA(pte));
      uint64 child = PTE2PA(pte);
      vmprint((pagetable_t)child, depth+1);
    }
  }
}
```



