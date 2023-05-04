---
layout: post
title: "MIT 6.S081—Lab 2: system calls"
subtitle: "system calls"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - xv6
---

Before you start coding, read Chapter 2 of the [xv6 book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev1.pdf), and Sections 4.3 and 4.4 of Chapter 4, and related source files:

- The user-space code for systems calls is in `user/user.h` and `user/usys.pl`.
- The kernel-space code is `kernel/syscall.h`, kernel/syscall.c.
- The process-related code is `kernel/proc.h` and `kernel/proc.c`.

## System call tracing (moderate)

In this assignment you will add a system call tracing feature that may help you when debugging later labs. You'll create a new `trace` system call that will control tracing. It should take one argument, an integer "mask", whose bits specify which system calls to trace. For example, to trace the fork system call, a program calls `trace(1 << SYS_fork)`, where `SYS_fork` is a syscall number from `kernel/syscall.h`. You have to modify the xv6 kernel to print out a line when each system call is about to return, if the system call's number is set in the mask. The line should contain the process id, the name of the system call and the return value; you don't need to print the system call arguments. The `trace` system call should enable tracing for the process that calls it and any children that it subsequently forks, but should not affect other processes.

**示例**

We provide a `trace` user-level program that runs another program with tracing enabled (see `user/trace.c`). When you're done, you should see output like this:

```shell
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
$
$ trace 2147483647 grep hello README
4: syscall trace -> 0
4: syscall exec -> 3
4: syscall open -> 3
4: syscall read -> 1023
4: syscall read -> 966
4: syscall read -> 70
4: syscall read -> 0
4: syscall close -> 0
$
$ grep hello README
$
$ trace 2 usertests forkforkfork
usertests starting
test forkforkfork: 407: syscall fork -> 408
408: syscall fork -> 409
409: syscall fork -> 410
410: syscall fork -> 411
409: syscall fork -> 412
410: syscall fork -> 413
409: syscall fork -> 414
411: syscall fork -> 415
...
$   
```

In the first example above, trace invokes grep tracing just the read system call. The 32 is `1<<SYS_read`. In the second example, trace runs grep while tracing all system calls; the 2147483647 has all 31 low bits set. In the third example, the program isn't traced, so no trace output is printed. In the fourth example, the fork system calls of all the descendants of the `forkforkfork` test in `usertests` are being traced. Your solution is correct if your program behaves as shown above (though the process IDs may be different).

Some hints:

- Add `$U/_trace` to UPROGS in Makefile
- Run make qemu and you will see that the compiler cannot compile `user/trace.c`, because the user-space stubs for the system call don't exist yet: add a prototype for the system call to `user/user.h`, a stub to `user/usys.pl`, and a syscall number to `kernel/syscall.h`. The Makefile invokes the perl script `user/usys.pl`, which produces `user/usys.S`, the actual system call stubs, which use the RISC-V `ecall` instruction to transition to the kernel. Once you fix the compilation issues, run trace 32 grep hello README; it will fail because you haven't implemented the system call in the kernel yet.

  > 前面两个hints就按照要求去相应文件添加就好了，使得系统能成功编译

```c
// user/trace.c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]){
  int i;
  char *nargv[MAXARG];

  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }

  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);
  exit(0);
}
```

代码比较简单，关键就是调用trace(mask)，然后把mask后面的命令行输入通过exec执行调用。

- Add a `sys_trace()` function in `kernel/sysproc.c` that implements the new system call by remembering its argument in a new variable in the `proc` structure (see `kernel/proc.h`). The functions to retrieve system call arguments from user space are in `kernel/syscall.c`, and you can see examples of their use in `kernel/sysproc.c`.

    > 这里首先需要在kernel/sysproc.c中定义trace系统调用，即sys_trace()函数，直接根据前面的示例有样学样即可。
    >
    > 然后还需要在kernel/proc.h中对进程结构体proc添加一个变量用于保存trace调用的参数mask。
    >
    > 最后一步也是很重要的，就是如何把我们输入的命令中的参数写入到相应的结构体变量中，这里需要看kernel/syscall.c、kernel/sysproc.c以及xv6 book 4.3和4.4节的内容，主要是trapframe和riscv中a0-a7寄存器的使用。

    ```c
    // kernel/sysproc.c
    uint64
    sys_trace(void)
    {
      if(argint(0, &(myproc()->mask)) < 0)
        return -1;
      return 0;
    }

    // argint(0, &(myproc()->mask))作用是将第0位的系统调用参数写入myproc()->mask（当前调用进程的mask变量）
    // argint函数通过argraw(n)来返回当前进程p->trapframe->an的值
    ```

- Modify `fork()` (see `kernel/proc.c`) to copy the trace mask from the parent to the child process.

	> 这个就很简单了，直接把父进程的mask变量赋给子进程的mask变量即可

- Modify the `syscall()` function in `kernel/syscall.c` to print the trace output. You will need to add an array of syscall names to index into.

	> 这里需要对syscall()进行修改，使得输出可以满足题目要求

    ```c
    // kernel/syscall.c
    static char *syscall_names[] = {
    [SYS_fork]    "fork",
    [SYS_exit]    "exit",
    [SYS_wait]    "wait",
    [SYS_pipe]    "pipe",
    [SYS_read]    "read",
    [SYS_kill]    "kill",
    [SYS_exec]    "exec",
    [SYS_fstat]   "fstat",
    [SYS_chdir]   "chdir",
    [SYS_dup]     "dup",
    [SYS_getpid]  "getpid",
    [SYS_sbrk]    "sbrk",
    [SYS_sleep]   "sleep",
    [SYS_uptime]  "uptime",
    [SYS_open]    "open",
    [SYS_write]   "write",
    [SYS_mknod]   "mknod",
    [SYS_unlink]  "unlink",
    [SYS_link]    "link",
    [SYS_mkdir]   "mkdir",
    [SYS_close]   "close",
    [SYS_trace]   "trace",
    };
  
    // 方括号 [] 则是用于指定初始化列表中每个元素对应的下标（索引）。以第一行为例：[SYS_fork] "fork" 表示将 "fork" 字符串分配给 syscall_names 数组中下标为 SYS_fork 的位置，也就是 syscall_names[SYS_fork] = "fork"。通过这种方式，可以快速地初始化一个字符型指针数组，并且可以实现在数组中按照自定义顺序排列元素的目的。
  
    void
    syscall(void)
    {
      int num;
      struct proc *p = myproc();
  
      num = p->trapframe->a7;
      if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
        p->trapframe->a0 = syscalls[num]();
        if(1<<num & p->mask)
          printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);
      } else {
        printf("%d %s: unknown sys call %d\n",
                p->pid, p->name, num);
        p->trapframe->a0 = -1;
      }
    }
  
    // 通过阅读xv6 book 4.3/4.4节，我们知道a7保存的是syscall的系统调用号，a0则保存系统调用执行后的返回值，syscall函数先对num进行检查，以及当前定义的syscalls数组中是否有该调用号的系统调用，如果满足条件，则说明系统调用成功，这里我们就需要修改代码进行输出了
    // 1<<num & p->mask，通过位运算巧妙的满足只要位置1的系统调用都输出
    ```

## Sysinfo (moderate)

In this assignment you will add a system call, `sysinfo`, that collects information about the running system. The system call takes one argument: a pointer to a `struct sysinfo` (see `kernel/sysinfo.h`). The kernel should fill out the fields of this struct: the `freemem` field should be set to the number of bytes of free memory, and the `nproc` field should be set to the number of processes whose `state` is not `UNUSED`. We provide a test program `sysinfotest`; you pass this assignment if it prints "sysinfotest: OK".

Some hints:

- Add `$U/_sysinfotest` to UPROGS in Makefile

- Run make qemu; `user/sysinfotest.c` will fail to compile. Add the system call sysinfo, following the same steps as in the previous assignment. To declare the prototype for sysinfo() `in user/user.h` you need predeclare the existence of `struct sysinfo`:

  ```c
  struct sysinfo;
  int sysinfo(struct sysinfo *); 
  ```

  > 步骤和trace实验中的一样，添加相应的系统调用即可

  Once you fix the compilation issues, run `sysinfotest`; it will fail because you haven't implemented the system call in the kernel yet.

  先来看下测试程序sysinfotest的构成：

  ```c
  // user/sysinfotest.c
  
  #include "kernel/types.h"
  #include "kernel/riscv.h"
  #include "kernel/sysinfo.h"
  #include "user/user.h"
  
  // 执行sysinfo系统调用
  
  void
  sinfo(struct sysinfo *info) {
    if (sysinfo(info) < 0) {
      printf("FAIL: sysinfo failed");
      exit(1);
    }
  }
  
  //
  // use sbrk() to count how many free physical memory pages there are.
  //
  int
  countfree()
  {
    // 记录当前地址到sz0，为最后释放分配的内存做准备
    uint64 sz0 = (uint64)sbrk(0);
    struct sysinfo info;
    int n = 0;
    
    // 使用sbrk()申请内存，直到达到最大限制
    while(1){
      if((uint64)sbrk(PGSIZE) == 0xffffffffffffffff){
        break;
      }
      n += PGSIZE;
    }
    sinfo(&info);
    // 测试此时sysinfo是否正确（即当前应该返回0，因为由于上面的申请，系统当前已无空闲内存）
    if (info.freemem != 0) {
      printf("FAIL: there is no free mem, but sysinfo.freemem=%d\n",
        info.freemem);
      exit(1);
    }
    // 来释放先前分配的所有内存块，并将堆大小恢复回初始状态。
    sbrk(-((uint64)sbrk(0) - sz0));
    // 返回前面成功分配的页面大小之和
    return n;
  }
  
  // 测试sysinfo调用是否正确获得当前空闲内存值
  
  void
  testmem() {
    struct sysinfo info;
    uint64 n = countfree();
    
    sinfo(&info);
  
    if (info.freemem!= n) {
      printf("FAIL: free mem %d (bytes) instead of %d\n", info.freemem, n);
      exit(1);
    }
    
    // 分配一个页
    if((uint64)sbrk(PGSIZE) == 0xffffffffffffffff){
      printf("sbrk failed");
      exit(1);
    }
  
    sinfo(&info);
      
    if (info.freemem != n-PGSIZE) {
      printf("FAIL: free mem %d (bytes) instead of %d\n", n-PGSIZE, info.freemem);
      exit(1);
    }
    
    // 释放一个页
    if((uint64)sbrk(-PGSIZE) == 0xffffffffffffffff){
      printf("sbrk failed");
      exit(1);
    }
  
    sinfo(&info);
      
    if (info.freemem != n) {
      printf("FAIL: free mem %d (bytes) instead of %d\n", n, info.freemem);
      exit(1);
    }
  }
  
  //  测试sysinfo系统调用是否成功
  
  void
  testcall() {
    struct sysinfo info;
    
    if (sysinfo(&info) < 0) {
      printf("FAIL: sysinfo failed\n");
      exit(1);
    }
  
    if (sysinfo((struct sysinfo *) 0xeaeb0b5b00002f5e) !=  0xffffffffffffffff) {
      printf("FAIL: sysinfo succeeded with bad argument\n");
      exit(1);
    /*
    它试图调用 sysinfo 函数并传递一个错误的指针作为参数。这个指针是一个远离有效内存区域的无效地址。0xeaeb0b5b00002f5e 是一个固定值，它代表了一个非法地址，这个地址显然远离当前系统所属的进程的有效内存区域。在 RISC-V 架构中，地址空间的范围可以是2^39个字节，并分为若干个区域，如用户空间、内核空间等。如果给 sysinfo 函数传递了一个无效的地址（类似于上述代码中的例子），则函数会试图访问一个不可操作的虚拟地址空间。当尝试访问这个无效地址时，硬件会接收到一个异常（exception），操作系统会相应地捕获该异常并在返回之前终止程序的执行。然后，在验证 sysinfo 函数返回值时，代码使用 0xffffffffffffffff 来标识未成功执行。
    */
    }
  }
  
  // 测试sysinfo是否正确获得当前处于UNUSED状态的进程数
  
  void testproc() {
    struct sysinfo info;
    uint64 nproc;
    int status;
    int pid;
    
    sinfo(&info);
    nproc = info.nproc;
  
    pid = fork();
    if(pid < 0){
      printf("sysinfotest: fork failed\n");
      exit(1);
    }
    // 子进程
    if(pid == 0){
      sinfo(&info);
      if(info.nproc != nproc+1) {
        printf("sysinfotest: FAIL nproc is %d instead of %d\n", info.nproc, nproc+1);
        exit(1);
      }
      exit(0);
    }
    // 等待子进程退出后的父进程
    wait(&status);
    sinfo(&info);
    if(info.nproc != nproc) {
        printf("sysinfotest: FAIL nproc is %d instead of %d\n", info.nproc, nproc);
        exit(1);
    }
  }
  
  /* 主函数很简单，分别测试sysinfo调用是否成功；测试sysinfo是否正确获得当前空闲内存大小；测试sysinfo是否正确获得当前处于UNUSED状态的进程数
     如果按照上述hints添加了相应的代码，现在执行sysinfotest则会输出：
     sysinfotest: start
     FAIL: sysinfo succeeded with bad argument
  */
  
  int
  main(int argc, char *argv[])
  {
    printf("sysinfotest: start\n");
    testcall();
    testmem();
    testproc();
    printf("sysinfotest: OK\n");
    exit(0);
  }
  ```

  很明显这里我们的工作是需要在调用sysinfo(info)的时候，将kernel中的struct sysinfo内容传递给info，即kernel space —> user space（也就是下一个hint）。

  > 不得不说大牛就是大牛，四大就是四大；上述测试程序写的是真的好，很值得学习。
  >
  > 关于sbrk和malloc：
  >
  > `sbrk()` 和 `malloc()` 的功能都是动态分配内存，但二者的本质不同。
  >
  > `sbrk()` 是一种系统调用，而不是库函数，可以扩展或收缩进程数据段（也就是堆），并返回新分配内存的地址。因此` sbrk()` 更底层，而且更加灵活，需要手动管理内存使用情况，通常用来实现更高级别的动态内存分配器比如 `malloc()` 或 `new`。
  >
  > 而 `malloc()` 是 C 标准库中的函数，在程序运行时自动调整堆，并返回指向所申请对象的指针。通常情况下，程序使用 `malloc()` 来动态地分配内存并实现程序员定义的数据结构。
  >
  > 总之，`sbrk()` 是内核提供的函数，`malloc()` 是程序设计语言库内部提供的函数。它们的区别在于应用场景和实际应用需求：如果只需简单地分配一些连续块内存，则使用 `sbrk()` 就足够了；如果需要分配复杂的数据结构或处理字符串等，则使用 `malloc()` 等库函数会更方便。

- sysinfo needs to copy a `struct sysinfo` back to user space; see `sys_fstat()` (`kernel/sysfile.c`) and `filestat()` (`kernel/file.c`) for examples of how to do that using `copyout()`.

  这里让我们学习sys_fstat() (kernel/sysfile.c)和filestat() (kernel/file.c)两个函数来学习copyout()函数的使用。

  ```c
  // kernel/sysfile.c
  
  uint64
  sys_fstat(void)
  {
    struct file *f;
    uint64 st; // user pointer to struct stat
  
    if(argfd(0, 0, &f) < 0 || argaddr(1, &st) < 0)
      return -1;
    return filestat(f, st);
  }
  
  // 这里对于实验有用的是，argaddr(1, &st)，里面又调用了*st = argraw(1)，即将p->trapframe->a1的值赋给st
  ```

  ```c
  // kernel/file.c
  
  // Get metadata about file f.
  // addr is a user virtual address, pointing to a struct stat.
  int
  filestat(struct file *f, uint64 addr)
  {
    struct proc *p = myproc();
    struct stat st;
    
    if(f->type == FD_INODE || f->type == FD_DEVICE){
      ilock(f->ip);
      stati(f->ip, &st);
      iunlock(f->ip);
      // 调用copyout()
      if(copyout(p->pagetable, addr, (char *)&st, sizeof(st)) < 0)
        return -1;
      return 0;
    }
    return -1;
  }
  ```

  ```c
  // kernel/vm.c
  
  // Copy from kernel to user.
  // Copy len bytes from src to virtual address dstva in a given page table.
  // Return 0 on success, -1 on error.
  int
  copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
  {
    ......
  }
  
  // copyout的具体代码我们现在可以不用关注（其实原理也就是内存复制操作，关于地址，关于指针）
  // 可以看到copyout函数是从kernel space复制东西给user space，需要四个参数，第一个是进程的页表，第二个是目的地址，第三个是源地址，第四个为复制的内容大小
  ```

- To collect the amount of free memory, add a function to `kernel/kalloc.c`

  ```c
  // kernel/kalloc.c
  
  uint64
  getfreemem(void)
  {
    struct run *r;
    uint64 freemem = 0;
  
    acquire(&kmem.lock);
    r = kmem.freelist;
    while(r){
      freemem += PGSIZE;
      r = r->next;
    }
    release(&kmem.lock);
    return freemem;
  }
  
  // xv6中通过freelist来管理空闲内存，r不为null则表示为空闲内存
  // 这里我们参考kalloc()来写就好，注意锁的加和释放
  // ！！！不要忘记在defs.h中声明
  ```

- To collect the number of processes, add a function to `kernel/proc.c`
  ```c
  // kernel/proc.c
  
  uint64
  getnproc(void)
  {
    struct proc *p;
    uint64 n = 0;
  
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state != UNUSED) 
        n++;
      release(&p->lock);
    }
    return n;
  }
  
  // 这个也很简单，在proc数组中遍历即可，参考allocproc()来写就好
  // ！！！不要忘记在defs.h中声明
  ```

最后来写sys_sysinfo函数。
```c
// kernel/sysproc.c

uint64
sys_sysinfo(void){
  struct sysinfo info;
  struct proc *p = myproc();
  uint64 addr;

  info.freemem = getfreemem();
  info.nproc = getnproc();
  
  // 需要注意的是这里是0，不是1；我们调用的时候sysinfo(info)只有一个参数，所以是0
  if(argaddr(0, &addr) < 0)
    return -1;
  if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
      return -1;
  return 0;
}
```







