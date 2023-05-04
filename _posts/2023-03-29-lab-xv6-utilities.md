---
layout: post
title: "MIT 6.S081—Lab 1: Xv6 and Unix utilities"
subtitle: "Xv6 and Unix utilities"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - xv6
---

## sleep (easy)

Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.

Some hints:

- Before you start coding, read Chapter 1 of the [xv6 book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf).
- Look at some of the other programs in `user/` (e.g., `user/echo.c`, `user/grep.c`, and `user/rm.c`) to see how you can obtain the command-line arguments passed to a program.
- If the user forgets to pass an argument, sleep should print an error message.
- The command-line argument is passed as a string; you can convert it to an integer using `atoi` (see user/ulib.c).
- Use the system call `sleep`.
- See `kernel/sysproc.c` for the xv6 kernel code that implements the `sleep` system call (look for `sys_sleep`), `user/user.h` for the C definition of `sleep` callable from a user program, and `user/usys.S` for the assembler code that jumps from user code into the kernel for `sleep`.
- Make sure `main` calls `exit()` in order to exit your program.
- Add your `sleep` program to `UPROGS` in Makefile; once you've done that, `make qemu` will compile your program and you'll be able to run it from the xv6 shell.
- Look at Kernighan and Ritchie's book *The C programming language (second edition)* (K&R) to learn about C.

**示例**

Run the program from the xv6 shell:

```
$ make qemu
...
init: starting sh
$ sleep 10
(nothing happens for a little while)
$
```

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]){
    // 判断参数
    if(argc < 2){
        fprintf(2, "Usage: sleep <time>\n");
        exit(1);
    }
    // 使用atoi()函数将string转换为数字，并调用sleep syscall
    sleep(atoi(argv[1]));
    exit(0);
}
```

## pingpong (easy)

Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.

Some hints:

- Use `pipe` to create a pipe.
- Use `fork` to create a child.
- Use `read` to read from the pipe, and `write` to write to the pipe.
- Use `getpid` to find the process ID of the calling process.
- Add the program to `UPROGS` in Makefile.
- User programs on xv6 have a limited set of library functions available to them. You can see the list in `user/user.h`; the source (other than for system calls) is in `user/ulib.c`, `user/printf.c`, and `user/umalloc.c`.

使用两个管道进行父子进程通信，需要注意的是如果管道的写端没有close，那么管道中数据为空时对管道的读取将会阻塞。因此对于不需要的管道描述符，要尽可能早的关闭。

**示例**

Run the program from the xv6 shell and it should produce the following output:

```
$ make qemu
...
init: starting sh
$ pingpong
4: received ping
3: received pong
$
```

```c
#include "kernel/types.h"
#include "user/user.h"

int main(int argc, char *argv[]){
   // 创建两个管道
   int p_c[2], c_p[2];
   pipe(p_c);
   pipe(c_p);
   // 一字节数据
   char buff = 'p';
   int pid = fork();
   if(pid > 0){ // 父进程
        // 关闭不需要的管道端，即父进程的读端和子进程的写端
        close(p_c[0]);
        close(c_p[1]);
        // 父进程写入一字节的数据
        write(p_c[1], &buff, sizeof(char));
        // 父进程读入一字节的数据
        read(c_p[0], &buff, sizeof(char));
        printf("%d: received pong\n", getpid());
        // 关掉使用完毕的管道
        close(p_c[1]);
        close(p_c[0]);
        exit(0);
   }else if(pid == 0){ // 子进程
        // 关闭不需要的管道端，即父进程的写端和子进程的读端
        close(p_c[1]);
        close(c_p[0]);
        // 子进程读入一字节的数据
        read(p_c[0], &buff, sizeof(char));
        printf("%d: received ping\n", getpid());
        // 子进程写入一字节的数据
        write(c_p[1], &buff, sizeof(char));
        // 关闭使用完毕的管道
        close(p_c[0]);
        close(c_p[1]);
        exit(0);
   }else{
     close(p_c[0]);
     close(p_c[1]);
     close(c_p[0]);
     close(c_p[1]);
     exit(1);
   }
}
```

## primes (moderate)/(hard)

**关于并发编程（多进程版本递归）**

Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.

**素数生成**

> "As another example, which Hoare credits to Doug McIlroy, consider the generation of all primes less than a thousand. The sieve of Eratosthenes can be simulated by a pipeline of processes executing the following pseudocode:"

```
p = get a number from left neighbor
print p
loop:
    n = get a number from left neighbor
    if (p does not divide n)
        send n to right neighbor
```

> "A generating process can feed the numbers 2, 3, 4, ..., 1000 into the left end of the pipeline: the first process in the line eliminates the multiples of 2, the second eliminates the multiples of 3, the third eliminates the multiples of 5, and so on:"

<img src="https://swtch.com/~rsc/thread/sieve.gif">

Your goal is to use `pipe` and `fork` to set up the pipeline. The first process feeds the numbers 2 through 35 into the pipeline. For each prime number, you will arrange to create one process that reads from its left neighbor over a pipe and writes to its right neighbor over another pipe. Since xv6 has limited number of file descriptors and processes, the first process can stop at 35.

Some hints:

- Be careful to close file descriptors that a process doesn't need, because otherwise your program will run xv6 out of resources before the first process reaches 35.
- Once the first process reaches 35, it should wait until the entire pipeline terminates, including all children, grandchildren, &c. Thus the main primes process should only exit after all the output has been printed, and after all the other primes processes have exited.
- Hint: `read` returns zero when the write-side of a pipe is closed.
- It's simplest to directly write 32-bit (4-byte) `int`s to the pipes, rather than using formatted ASCII I/O.
- You should create the processes in the pipeline only as they are needed.
- Add the program to `UPROGS` in Makefile.

**示例**

Your solution is correct if it implements a pipe-based sieve and produces the following output:

```
$ make qemu
...
init: starting sh
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
$
```

```c
#include "kernel/types.h"
#include "user/user.h"

void pipline(int rfd){
    int p, n;
    // 数据读取完毕，关闭文件描述符，并退出程序（最后一个prime）
    if(read(rfd, &p, sizeof(int)) == 0){
        close(rfd);
        exit(0);
    };
    // 打印第一个数
    printf("prime %d\n", p);
    // 创建连接下一个进程的管道
    int ppl[2];
    pipe(ppl);
    if(fork() == 0){ // 子进程
        close(ppl[1]);
        pipline(ppl[0]); // 继续递归调用
    } else{// 父进程
        close(ppl[0]);
        for(;;){
            // 上个进程输入的数据读完即结束循环
            if(read(rfd, &n, sizeof(int)) == 0){
                close(rfd);
                break;
            }
            // 过滤非质数
            if(n % p != 0){
                write(ppl[1], &n, sizeof(int));
            }
        }
        close(ppl[1]);
        wait((int *)0);
        exit(0);
    }
}

int main(int argc, char* argv[]){
    // 创建一个管道
    int p[2];
    pipe(p);

    if(fork() > 0){  // 主进程
        // 关闭管道的读端
        close(p[0]);
        // 向第一个进程写入2-35，一次写入一个数
        for(int i = 2; i <= 35; i++){
            write(p[1], &i, sizeof(int));
        }
        // 关闭写完数据的写端
        close(p[1]);
        wait((int *) 0); // 等待子进程完成
        exit(0);
    } else{ // 子进程
        // 关闭写端 
        close(p[1]);
        pipline(p[0]);
        exit(0);
    }
}
```

## find (moderate)

Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`

Some hints:

- Look at user/ls.c to see how to read directories.
- Use recursion to allow find to descend into sub-directories.
- Don't recurse into "." and "..".
- Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.
- You'll need to use C strings. Have a look at K&R (the C book), for example Section 5.5.
- Note that == does not compare strings like in Python. Use strcmp() instead.
- Add the program to `UPROGS` in Makefile.

**示例**

Your solution is correct if produces the following output (when the file system contains the files `b` and `a/b`):

```
$ make qemu
...
init: starting sh
$ echo > b
$ mkdir a
$ echo > a/b
$ find . b
./b
./a/b
$ 
```

**阅读源码的一些笔记**

**dirsiz**

`dirsiz` 是一个用于计算目录大小的变量，它位于 XV6 操作系统中的 `fs.h` 头文件中。

在 UNIX 和类 UNIX 操作系统中，目录是一种特殊的文件，包含了指向文件和子目录的链接信息。每个链接信息都以一个叫做 `dirent` 的数据结构表示，其中包含了链接信息的长度、目标文件的 inode 编号以及目标文件的名称等字段。

因此，当我们获取目录的总大小时，实际上是在计算目录文件中所有 `dirent` 结构体的大小之和。这也是 `dirsiz` 变量的作用：在遍历目录文件内容时，通过累加每个 `dirent` 结构体的大小来计算目录大小。

需要注意的是，由于目录文件通常比较小，所以它们的大小单位通常是字节（byte），而非千字节（Kb）或兆字节（Mb）。

如果目录中的文件名长度总和小于或等于 `DIRSIZ`（通常定义为 14 或 255 字节），则可以通过向结构体添加填充来使目录占用固定的字节数。但是，如果文件名的长度大于 `DIRSIZ`（例如，当使用了超长的文件名时）目录项将无法采用固定的大小来存储它们，而是通过分配其他目录项调整大小。

**inode**

在文件系统中，每个文件和目录都有一个唯一的 `inode` (index node) 编号，它是 Unix/Linux 文件系统实现的关键。在内核中，每一个打开的文件或目录都对应一个 `inode` 结构体，记录了文件或目录的元数据信息，如权限、时间戳、所有者、组别等等。

`inode` 中还包含了指向表示该文件所在数据块的磁盘地址的指针，这些指针定义了文件的物理位置。

每个文件和目录名字存储在目录文件中，而这些名字仅用于标识以及与它们所表示的 `inode` 相关联。通常，文件如果被删除，则其相关的 `inode` 被释放，可以被重用；而实际的文件内容会在文件系统处理某个优化和空间回收时才会被回收。

通过使用 `ls -i` 命令，可以显示一个目录下每个文件和子目录的 `inode` 编号。同时，你还可以使用 `stat` 命令来查看一个文件的 `inode` 信息。

> 这题的关键在于读懂ls.c代码

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

// 该函数输入一个字符串表示路径，并返回最后一个斜杠 (‘/’) 后面的部分，在此之前如果字符长度超过了 DIRSIZ，则会省略并返回原始路径中的后缀；path是输入的路径字符串，形式为 dirname/filename 系统格式。
char*
fmtname(char *path)
{
  static char buf[DIRSIZ+1];  // 用于存储返回结果的静态字符数组，该数组的大小为 DIRSIZ 加上一个字节用于存放字符串结尾的空字符
  char *p;

  // 找到最后一个 '/' 后面的第一个字符。这些字符就是文件名或目录名
  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;

  // Return blank-padded name.
  // 如果文件名或目录名长度大于等于 DIRSIZ，则直接返回原始名称
  if(strlen(p) >= DIRSIZ)
    return p;
  // 先通过 memmove() 将 p 指针所指向的字符串复制到 buf 缓冲区中。紧接着，使用 memset() 函数将缓冲区剩余部分填充为空格，以便在显示长文件名时使每一行的输出长度相等
  memmove(buf, p, strlen(p));
  memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));
  return buf;
}

// 该函数输入一个路径字符串，打印出该路径下的所有文件和子目录的名称、类型、inode 号及大小信息
void
ls(char *path)
{
  char buf[512], *p;
  int fd;
  struct dirent de; // 用于保存从目录中读取到的每个文件或目录信息
  struct stat st;   // 用于读取指定文件或目录的元数据信息

  if((fd = open(path, 0)) < 0){
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_FILE:
    printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
    break;

  case T_DIR:
    // 检查路径名是否过长，如果超出了 BUFSIZE 的限制，则输出提示信息并退出函数（两次+1是因为'/'的出现和0的出现）
    // 下面代码将de.name 中的文件名拷贝到缓存区 buf中，为什么不直接使用de.name呢：在 Unix/Linux 系统中，目录项名称的长度是有限制的，如果我们直接使用 de.name 字符数组，则可能会导致缓存区溢出和内存泄漏等问题（关于缓存区溢出和内存泄露这一部分还是有点抽象难以理解）
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }
    // 第一次+1
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';
    // 依次读取目录内容，获取每个文件的 dirent 信息
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);
      // 第二次+1，0即为了形成C风格字符串
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  close(fd);
}

int
main(int argc, char *argv[])
{
  int i;

  if(argc < 2){
    ls(".");
    exit(0);
  }
  for(i=1; i<argc; i++)
    ls(argv[i]);
  exit(0);
}
```

**find.c**

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

// 格式化路径path，ls.c中的fmtname()是包括./还有空格，这里需要在memset()函数中进行修改
char *fmtname(char *path){
  static char buf[DIRSIZ+1];
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;

  if(strlen(p) >= DIRSIZ)
    return p;
  memmove(buf, p, strlen(p));
  memset(buf+strlen(p), 0, DIRSIZ-strlen(p)); 
  return buf;
}

void find(char *path, char *filename){
    char buf[512], *p; 
    int fd;            
    struct dirent de;  
    struct stat st;    

    if((fd = open(path, 0)) < 0){
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    // 进行判断
    if( strcmp(filename, fmtname(path)) == 0){
        printf("%s\n", path);
    }

    switch(st.type){
        case T_DIR:
            if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
                fprintf(2, "find: path too long\n");
                break;
            }
            strcpy(buf, path);
            p = buf+strlen(buf);
            *p++ = '/';
            // 遍历目录下的每一个文件
            while(read(fd, &de, sizeof(de)) == sizeof(de)){
                if(de.inum == 0)
                    continue;
                // 不递归至.以及..（ls.c中会ls出.和..）
                if(strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
                    continue;
                memmove(p, de.name, DIRSIZ); 
                p[DIRSIZ] = 0;               
                if(stat(buf, &st) < 0){
                    fprintf(2, "find: cannot stat %s\n", buf);
                    continue;
                }
                // 递归调用find()
                find(buf, filename);
            }
            break;
    }
    close(fd);
}

int main(int argc, char *argv[]){
    // 先判断参数
    if(argc != 3){
        fprintf(2, "Usage: find <directory> <file>\n");
        exit(1);
    }
    else{
        find(argv[1], argv[2]);
        exit(0);            
    }
}
```

## xargs (moderate)

Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file `user/xargs.c`.

The following example illustrates xarg's behavior:

```
$ echo hello too | xargs echo bye
bye hello too
$ 
```

Note that the command here is "echo bye" and the additional arguments are "hello too", making the command "echo bye hello too", which outputs "bye hello too".

Please note that xargs on UNIX makes an optimization where it will feed more than argument to the command at a time. We don't expect you to make this optimization. To make xargs on UNIX behave the way we want it to for this lab, please run it with the -n option set to 1. For instance

```
$ echo "1\n2" | xargs -n 1 echo line
line 1
line 2
$ 
```

Some hints:

- Use `fork` and `exec` to invoke the command on each line of input. Use `wait` in the parent to wait for the child to complete the command.
- To read individual lines of input, read a character at a time until a newline ('\n') appears.
- `kernel/param.h` declares `MAXARG`, which may be useful if you need to declare an argv array.
- Add the program to `UPROGS` in Makefile.
- Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.

xargs, find, and grep combine well:

```
$ find . b | xargs grep hello 
```

will run "grep hello" on each file named b in the directories below ".".

To test your solution for xargs, run the shell script xargstest.sh. Your solution is correct if it produces the following output:

```
$ make qemu
...
init: starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
$ $   
```

```c
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/param.h"

int main(int argc, char *argv[]){
    char *argv_array[MAXARG];
    int i;
    // 将xargs的所有参数装入argv_array数组中
    for(i = 0; i < argc; i++){
        argv_array[i] = argv[i];
    }
    // 用于接收父进程输出
    char buff[256];
    for( ; ;){
        int j = 0;
        // 接收一行输入，直到读完了或者出现换行符'\n'
        while(read(0, buff + j, sizeof(char)) != 0 && buff[j] != '\n'){
            j++;
        }
        // j=0表示read()返回0（进入for新循环后j被初始化为0，并且while循环中没有改变），即输入已全部读取完毕，跳出循环，结束父进程
        if(j == 0)
            break;
        buff[j] = 0;
        // 将父进程输出加入参数数组中
        argv_array[i] = buff;
        // 设置exec()结束
        argv_array[i+1] = 0;
        // 创建子进程，执行exec()调用
        if(fork() == 0){
            exec(argv_array[1], argv_array+1);
        } else{
            // 父进程等待子进程结束
            wait( (int *) 0);
        }
    }
    exit(0);
}
```

