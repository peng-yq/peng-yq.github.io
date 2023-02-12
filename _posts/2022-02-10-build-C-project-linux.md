---
layout: post
title: "Linux环境下创建C/C++项目"
subtitle: "Build C/C++ Project in Linux"
author: "PYQ"
header-img: "img/post-bg-linux.jpg"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - Linux
  - C/C++
---

## Ⅰ. 软件包安装

本文采用 Ubuntu 20.04.3 LTS 进行演示。

首先我们需要检查Linux系统是否配置了C语言环境，即编译器（gcc/g++）和调试器（gdb）。

```shell
gcc -v
```

```shell
g++ -v
```

```shell
gdb -v
```

若系统已配置好C语言环境，请直接进入第二步。若未配置好环境，则需要我们自己配置。

```shell
#一次性安装编译链
sudo apt install build-essential

#或者安装所需要的编译链
sudo apt install gcc
sudo apt install g++
sudo apt install gdb
```

## Ⅱ. gcc/g++常用编译指令

gcc 与 g++ 分别是 gnu 的 c & c++ 编译器 gcc/g++ 在执行编译工作的时候，总共需要4步：

- 预处理：生成 .i 的预处理文件 [预处理器cpp]
- 编译：将预处理文件转换成.s的汇编文件 [编译器egcs]
- 汇编：将汇编文件变为目标代码(机器代码)生成 .o 的目标文件 [汇编器as]
- 链接：链接目标代码、库文件和启动文件，生成可执行程序（Windows为.exe；Linux为.out） [链接器ld]

**1.gcc -E source_file.c**

-E，只执行到预处理，直接输出预处理结果。

**2.gcc -S source_file.c** 

-S，只执行源代码到汇编代码的转换，输出汇编代码。

**3.gcc -c source_file.c**

-c，只执行源代码到目标文件的转换，输出目标文件。

**4.gcc (-E/S/c/) source_file.c -o output_filename**

-o，指定输出文件名，可以配合以上三种标签使用，可以被省略，以默认名称输出。
-E，预编译结果将被输出到标准输出端口（通常是显示器）
-S，生成名为source_file.s的汇编代码
-c，生成名为source_file.o的目标文件。
无标签情况：生成名为output_filename.out的可执行文件

**5.gcc -g source_file.c** 

-g，生成供调试用的可执行文件，可以在gdb中运行。由于文件中包含了调试信息因此运行效率很低，且文件也大不少。这里可以用strip命令重新将文件中debug信息删除，这时会发现生成的文件甚至比正常编译的输出更小了，这是因为strip把原先正常编译中的一些额外信息（如函数名之类）也删除了，用法为 strip a.out。

**6.gcc -s source_file.c**

-s，直接生成与运用strip同样效果的可执行文件（删除了所有符号信息）。

**7.gcc -O source_file.c**

-O（大写的字母O），编译器对代码进行自动优化编译，输出效率更高的可执行文件。
-O 后面还可以跟上数字指定优化级别，如：
gcc -O2 source_file.c
数字越大，越加优化。但是通常情况下，自动的东西都不是太聪明，太大的优化级别可能会使生成的文件产生一系列的bug。

**8.gcc -Wall source_file.c**

-W，在编译中开启一些额外的警告（warning）信息。-Wall，将所有的警告信息全开。

**9.gcc source_file.c -L/path/to/lib  -I/path/to/include**
-L，指定函数库所在的文件夹，本例中链接器会尝试搜索/path/to/lib文件夹。

-I， 指定头文件所在的文件夹，本例中预编译器会尝试搜索/path/to/include文件夹。

更多关于Linux下使用gdb进行调试以及makefile文件编写可参考博客[LinuxC语言编程02:编译与调试C程序](https://blog.csdn.net/ncepu_Chen/article/details/103414441)，[跟我一起写makefile](https://seisman.github.io/how-to-write-makefile/index.html)。

## Ⅲ. 示例

在工作区文件夹建立`include`、`lib`、`src`和`output`四个文件夹，分别用于存储用户编写的头文件、编写头文件对应的源码、main.cpp和输出。同时在相应文件夹编写创建对应程序，目录树如下。

```shell
├── include
│   └── test.h
├── lib
│   ├── test.cpp
├── output
└── src
    └── main.cpp
```

通过对目录树的分析可知，我们首先需要将test.cpp编译为目标文件test.o，再链接main.cpp test.o，从而生成可执行文件。


> **test.h**

```cpp
#pragma once

void test();
```

> **test.cpp**

```c
#include <stdio.h>
#include "test.h"

void test(){
     printf("Hello!\n");
}
```

> **main.cpp**

```c
#include <stdio.h>
#include "test.h"

int main(void){
    test();
    return 0;
}
```

为了更方便的进行编译和输出，在工作区文件夹下建立`bin`文件夹并创建`build.sh`脚本。

```shell
├── bin
│   └── build.sh
├── include
│   └── test.h
├── lib
│   ├── test.cpp
│   └── test.o
├── output
│   └── output
└── src
    └── main.cpp
```

> **build.sh**

```shell
cd ../src
g++ main.cpp /home/pyq/Project/Test/lib/test.cpp -o /home/pyq/Project/Test/output/output -I /home/pyq/Project/Test/include -I /usr/include
cd ../output
./output
```

进入`bin`文件夹，并通过如下命令执行脚本。

```shell
sh ./build.sh
```

## Ⅳ Visual Studio Code

为了更加便利的在Linux下编写与调试C语言，我们采用Visual Studio Code。

### 4.1 在Ubuntu Software中安装

打开Ubuntu Software搜索Visual Studio Code，输入密码并下载。

### 4.2 使用apt源安装

Visual Studio Code 在官方的微软 Apt 源仓库中可用。想要安装它，按照下面的步骤来：

**1.以 sudo 用户身份运行下面的命令，更新软件包索引，并且安装依赖软件**

```shell
sudo apt update
sudo apt install software-properties-common apt-transport-https wget
```

**2.使用 wget 命令插入 Microsoft GPG key** 

```shell
wget -q https://packages.microsoft.com/keys/microsoft.asc -O- | sudo apt-key add -
```

**3.启用 Visual Studio Code 源仓库，输入**

```shell
sudo add-apt-repository "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main"
```

**4.一旦 apt 软件源被启用，安装 Visual Studio Code 软件包**

```shell
sudo apt install code
```

**5.当一个新版本被发布时，你可以通过你的桌面标准软件工具，或者在你的终端运行命令，来升级 Visual Studio Code 软件包**

```shell
sudo apt update
sudo apt upgrade
```

### 4.3 开启Visual Studio Code

打开VS Code可通过两种方法，即activities和命令行。

- 直接在activities中搜索Visual Studio Code，双击打开
- 在terminal中通过`code filename`，打开名为filename的文件夹

## Ⅴ. 在Visual Studio Code中创建C项目

Visual Studio Code汉化可通过Chinese插件。

安装C/C++插件。

创建[Ⅲ. 示例](#Ⅲ. 示例)中的文件目录树（不含bin目录）。

### 5.1 扩展头文件目录

在创建C项目时，往往我们会编写相应的自定义头文件，因此我们需要进行配置使得cpp文件可以找到相应的头文件。

`crtl`+`shift`+`p`打开设置，输入edit configuration，选择**C/C++：编辑配置(JSON)**，生成`.vscode`文件夹和`c_cpp_properties.json` 。

`.vscode`用于保存VS Code工作区配置文件，`c_cpp_properties.json` 用于保存相应参数和配置。

默认生成的`c_cpp_properties.json`如下。

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "gnu17",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-x64"
        }
    ],
    "version": 4
}
```

我们只需更新`“include”`参数即可。

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "${workspaceFolder}/include/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "cStandard": "gnu17",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-x64"
        }
    ],
    "version": 4
}
```

### 5.2 编译

Visual Studio Code作为编辑器，不认识我们书写的代码，已经需要转换为何种形式，因此需要我们告诉Visual Studio Code做什么。

`ctrl`+`shft`+`b`配置生成任务，选择**C/C++:g++生成活动文件夹（编译器/usr/bin/g++）**，在`.vscode`文件夹下生成`task.json`文件。

`task.json`用于保存编译命令即前面演示的`g++ -o`等命令。 

默认生成的`tasks.json`如下。

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cppbuild",
			"label": "C/C++: g++ 生成活动文件",
			"command": "/usr/bin/g++",
			"args": [
				"-fdiagnostics-color=always",
				"-g",
				"${file}",
				"-o",
				"${fileDirname}/${fileBasenameNoExtension}"
			],
			"options": {
				"cwd": "${fileDirname}"
			},
			"problemMatcher": [
				"$gcc"
			],
			"group": "build",
			"detail": "编译器: /usr/bin/g++"
		}
	]
}
```

我们需要进行更改的参数为`“args”`，这里用于保存shell具体执行的编译指令。关于**Visual Studio Code中预定义变量可参考[官方文档](https://code.visualstudio.com/docs/editor/variables-reference)**。此外我们还需要更改`“group”`变量，使得我们按下`ctrl`+`shift`+`b`时执行生成任务。

```json
{
	"version": "2.0.0",
	"tasks": [
		{
			"type": "cppbuild",
			"label": "C/C++: g++ 生成活动文件",
			"command": "/usr/bin/g++",
			"args": [
				"-fdiagnostics-color=always",
				"-g",
				"${file}",
				"${workspaceFolder}/lib/*.cpp",
				"-o",
				"${workspaceFolder}/output/${fileBasenameNoExtension}",
				"-I",
				"${workspaceFolder}/include"
			],
			"options": {
				"cwd": "${fileDirname}"
			},
			"problemMatcher": [
				"$gcc"
			],
			"group": {
				"isDefault": true,
				"kind": "build"
			},
			"detail": "编译器: /usr/bin/g++"
		}
	]
}
```

使用`ctrl`+`shift`+`b`执行任务。

可以看到无论是编辑器配置还是其他插件，**本质还是在shell中执行gcc/g++指令，指令内容即tasks.json中args参数内容**。

成功生成后将在`output`文件下生成可执行文件，在终端即可执行。

### 5.3 调试

调试C程序，可通过`f5`，选择**C++(GDB/LLDB)**，在`.vscode`目录下生成`launch.json`文件用于调试。

默认生成的`launch.json`如下。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++ - 生成和调试活动文件",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "C/C++: g++ 生成活动文件",
            "miDebuggerPath": "/usr/bin/gdb"
        }
    ]
}
```

我们只需要更改`"program"`参数，将`tasks.json`中目标文件的保存参数即`-o`后面一行替换当前内容即可。

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++ - 生成和调试活动文件",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/output/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "C/C++: g++ 生成活动文件",
            "miDebuggerPath": "/usr/bin/gdb"
        }
    ]
}
```

使用`f5`即可进行调试。

