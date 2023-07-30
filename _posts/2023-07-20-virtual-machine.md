---
layout: post
title: "MIT 6.S081—虚拟机"
subtitle: "Virtual Machines"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - OS
---

## Why Virtual Machine?

虚拟机可以认为是对于计算机的一种模拟，这种模拟足够能运行一个操作系统。虚拟机提供了额外的灵活性：将操作系统内核从之前的内核空间上移至宿主机的用户空间，并在操作系统内核之下增加新的一层来对接底层硬件。虚拟机实际上应用的非常非常广泛，并且它也有着很长的历史。虚拟机最早出现在1960年代，经过了一段时间的开发才变得非常流行且易用。我们为什么需要虚拟机呢？在实际中有很多场景使得人们会在一个计算机上运行多个相互独立的操作系统：

- 在一个大公司里面，你需要大量的服务，例如DNS，Firewall等等，但是每个服务并没有使用太多的资源，单独为这些服务购买物理机器有点浪费，但是将这些低强度的服务以虚拟机的形式运行在一个物理机上可以节省时间和资金。
- 虚拟机在云计算中使用的也非常广泛。云厂商通常不想直接出借物理服务器给用户，因为这很难管理。它们想向用户出借的是可以随意确定不同规格的服务器，或许有两个用户在一台物理服务器上，但是他们并没有太使用计算机，这样云厂商可以继续向同一个物理服务器上加入第三或者第四个用户。这样可以不使用额外的资金而获得更高的收益。
- 开发内核，比如xv6的运行是通过QEMU来模拟的，这样更加方便，调试也更容易。
- 最后就是我们通常使用虚拟机的一些普遍优势了，比如快照，易迁移，安全性和严格的隔离性等。

在虚拟机架构的最底层，**位于硬件之上存在一个Virtual Machine Monitor（VMM），它取代了标准的操作系统内核。VMM的工作是模拟多个计算机用来运行Guest操作系统。**VMM往上一层，如果对比一个操作系统的架构应该是用户空间，但是由于VMM的出现现在是叫做Guest空间。所以在下面的架构图里面，上面是Guest空间，下面是Host空间（也就是上面运行Guest操作系统，下面运行VMM）。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MUfCyqoGivm65u9nIVv%2F-MUfSJ2on6Jc_oFwKrXI%2Fimage.png?alt=media&token=d2ee9bac-d1fb-4b32-ad7f-b95a846403f9">

在Guest空间，会有一个或者多个Guest操作系统内核，比如Linux kernel。这里的Linux kernel会觉得自己就是个普通的内核，并在自己之上还运行一堆用户进程，例如VI，C Compiler。我们或许还有另一个Guest运行了Windows操作系统，同时也包含了Windows用户进程。所以，在Host空间运行的是VMM，在Guest空间运行的是普通的操作系统。除此之外，在Guest空间又可以分为Guest Supervisor Mode，也就是Guest操作系统内核运行的模式，和Guest User Mode。

>VMM（Virtual Machine Monitor，虚拟机监视器）是一种软件或硬件层，用于创建和管理虚拟机。它提供了一个虚拟化环境，允许在物理计算机上同时运行多个虚拟机。
>
>Hypervisor（也称为虚拟机监视器）是一种特定类型的VMM，它直接在物理硬件上运行，用于管理和分配计算机资源给虚拟机。Hypervisor可以分为两种类型：
>
>1. Type 1 Hypervisor（裸金属Hypervisor）：也称为本地Hypervisor，**它直接运行在物理硬件上，控制和管理虚拟机的资源**。它提供了更高的性能和效率，因为它直接与硬件交互，而不需要操作系统的干预。一些常见的Type 1 Hypervisor包括VMware ESXi、Microsoft Hyper-V和Xen。
>
>2. Type 2 Hypervisor（主机Hypervisor）：也称为主机上的Hypervisor，**它运行在操作系统之上**。在这种情况下，操作系统充当了中间层，虚拟机通过Type 2 Hypervisor与硬件进行通信。Type 2 Hypervisor适用于桌面虚拟化场景，常见的例子是Oracle VirtualBox和VMware Workstation。

## Trap and Emulate (Trap)

**关于QEMU**

QEMU是一款由法布里斯·贝拉等人编写的通用且免费的可执行硬件虚拟化的开源仿真器，它通过动态的二进制转换，模拟CPU，并且提供一组设备模型，使它能够运行多种未修改的客户机OS。QEMU的架构由纯软件实现，并在Guest与Host中间，来处理Guest的硬件请求，并由其转译给真正的硬件。然而因为QEMU是纯软件实现的，所有的指令都要经过QEMU，使得性能很差，而配合KVM则可以解决这一问题。QEMU虚拟化的思路是：提取Guest代码，翻译为TCG中间代码，而后翻译为Host代码。相当于实现了一个“中间人”的角色。

> 纯软件解析的虚拟机方案应用的并不广泛，因为它们很慢。在云计算中，这种实现方式非常不实用，人们并不会通过软件解析来在生产环境中构建虚拟机。

实际中，一种广泛使用的策略是在真实的CPU上运行Guest指令。比如我们要在VMM之上运行XV6，**我们需要先将XV6的指令加载到内存中，之后再跳转到XV6的第一条指令**，这样你的计算机硬件就能直接运行XV6的指令。当然，这要求你的计算机拥有XV6期望的处理器（RISC-V）。但是，当你的Guest操作系统执行了一个只能在kernel mode执行的privileged指令之后，就会出现问题。

前面说过，我们将Guest kernel按照一个Linux中的普通用户进程来运行，所以Guest kernel现在运行在User mode，而在User mode执行privileged指令是个非法的操作，这会导致虚拟机crash，但显然我们又不能让Guest Kernel运行在宿主机的Supervisor Mode。

相应的，这里会使用一些技巧。首先将Guest kernel运行在宿主机的User mode，这是最基本的策略。当通过VMM启动了一个XV6系统，VMM会将XV6的kernel指令加载到内存的某处，再设置好合适的Page Table使得XV6看起来自己的内存是从地址0开始向高地址走。**之后VMM会使用trap或者sret指令来跳转到位于User mode的Guest操作系统的第一条指令**，这样不论拥有多少条指令，Guest操作系统就可以一直执行下去。**一旦Guest操作系统需要使用privileged指令，因为它当前运行在User mode而不是Supervisor mode，会使得它触发trap并回到VMM中**（注，在一个正常操作系统中，如果在User mode执行privileged指令，会通过trap走到内核，但是现在VMM替代了内核），之后我们就可以获得控制权，并且我们的VMM也可以查看是什么指令引起的trap，并做适当的处理。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MUx2_oyVzbUMibBKtW_%2F-MV-0ZPcHzhf4bQG59Z7%2Fimage.png?alt=media&token=1d2a3846-8dae-4f26-96a9-f70ee4be50e1">

## Trap and Emulate (Emulate)

VMM会为每一个Guest维护一套虚拟状态信息，比如各种寄存器。当Guest操作系统运行指令需要读取某个privileged寄存器时，首先会通过trap走到VMM，因为在用户空间读取privileged寄存器是非法的。之后VMM会检查这条指令并发现这是一个比如说读取SEPC寄存器的指令，之后VMM会模拟这条指令，并将自己维护的虚拟SEPC寄存器，拷贝到trapframe的用户寄存器中。之后，VMM会将trapframe中保存的用户寄存器拷贝回真正的用户寄存器，通过sret指令，使得Guest从trap中返回。这时，用户寄存器a0里面保存的就是SEPC寄存器的值了，之后Guest操作系统会继续执行指令。最终，Guest读到了VMM替自己保管的虚拟SEPC寄存器。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MV-8-7clfUpY4jEznb2%2F-MV5uGULvI0pp5wnAKuC%2Fimage.png?alt=media&token=d0a803e1-9c45-4934-96f2-895c60f9a5b1">

在这种虚拟机的实现中，Guest整个运行在用户空间，任何时候它想要执行需要privilege权限的指令时，会通过trap走到VMM，VMM可以模拟这些指令。这种实现风格叫做**Trap and Emulate**。

## Trap and Emulate (Page Table)

考虑一个问题，以xv6为例，Guest操作系统在很多时候会修改SATP寄存器（保存了Page Table的地址的寄存器），按照前面的描述这会变成一个trap走到VMM，之后VMM可以接管。但是我们不想让VMM只是简单的替Guest设置真实的SATP寄存器，因为这样的话Guest就可以访问任意的内存地址，而不只是VMM分配给它的内存地址，所以我们不能让Guest操作系统简单的设置SATP寄存器。但是我们的确又需要为SATP寄存器做点什么，因为我们需要让Guest操作系统觉得Page Table被更新了。此外，当Guest上的软件运行了load或者store指令时，或者获取程序指令来执行时，我们需要数据或者指令来自于内存的正确位置，也就是Guest操作系统认为其PTE指向的内存位置。**所以当Guest设置SATP寄存器时，真实的过程是，我们不能直接使用Guest操作系统的Page Table，VMM会生成一个新的Page Table来模拟Guest操作系统想要的Page Table**。

那么虚拟机中的Page Table翻译地址是怎样的呢？首先是Guest kernel包含了Page Table，但是这里是将Guest中的虚拟内存地址映射到了Guest的物理内存地址。Guest物理地址是VMM分配给Guest的地址空间，例如32GB。并且VMM会告诉Guest这段内存地址从0开始，并一直上涨到32GB。但是在真实硬件上，这部分内存并不是连续的。所以我们不能直接使用Guest物理地址，因为它们不对应真实的物理内存地址。

相应的，VMM会为每个虚拟机维护一个映射表，将Guest物理内存地址映射到真实的物理内存地址。这个映射表与Page Table类似，对于每个VMM分配给Guest的Guest物理内存Page，都有一条记录表明真实的物理内存Page是什么。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MVEUD4jeuwnqkjfntX6%2F-MVEZObcKPRNuW90bpUT%2Fimage.png?alt=media&token=ef14ffe5-cd93-4382-995f-b51dad150d8e">

当Guest向SATP寄存器写了一个新的Page Table时，在对应的trap handler中，VMM会创建一个Shadow Page Table，Shadow Page Table的地址将会是VMM向真实SATP寄存器写入的值，它将gva映射到了hpa。所以，Guest kernel认为自己使用的是一个正常的Page Table，但是实际的硬件使用的是Shadow Page Table。这种方式可以阻止Guest从被允许使用的内存中逃逸。**Shadow Page Table只能包含VMM分配给虚拟机的主机物理内存地址。Guest不能向Page Table写入任何VMM未分配给Guest的内存地址。这是VMM实现隔离的一个关键部分**。

- 从Guest Page Table中取出每一条记录，查看gpa。
- 使用VMM中的映射关系，将gpa翻译成hpa。
- 再将gva和hpa存放于Shadow Page Table。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MVJJg8Nfx_zF_O0Avaf%2F-MVJM4IInp1iGSrmEIx1%2Fimage.png?alt=media&token=e8fda2cf-82e8-4ff0-a508-5a4ff0a8c8e7">

## Trap and Emulate (Devices)

在虚拟机实现中，如何让Guest操作系统认为它所需要的外部设备是存在的呢？主要有三种策略：

1. 模拟一些常用的设备，例如磁盘。也就是说，Guest并不是拥有一个真正的磁盘设备，只是VMM使得与Guest交互的磁盘看起来好像真的存在一样。Guest操作系统仍然会像与真实硬件设备交互一样，通过Memory Map控制寄存器与设备进行交互。操作系统会假设硬件已经将自己的控制寄存器映射到了内核地址空间的某个地址上。但在VMM中不会映射这些内存地址对应的Page，相应的会将这些Page设置成无效。这样当Guest操作系统尝试使用硬件时，一访问这些地址就会通过trap走到VMM。VMM查看指令并解析出Guest的具体事件。VMM中会对磁盘或者串口设备有一些模拟，通过这些模拟，VMM知道如何响应Guest的指令，之后再恢复Guest的执行。这也是QEMU实现UART的方式。这是一种常见的实现方式，但是这种方式可能会非常的低效，因为每一次Guest与外设硬件的交互，都会触发一个trap。
2. 在现代的世界中，操作系统在最底层是知道自己运行在虚拟机之上的。所以第二种策略是提供虚拟设备，而不是模拟一个真实的设备。通过在VMM中构建特殊的设备接口，可以使得Guest中的设备驱动与VMM内支持的设备进行高效交互。这种方式比直接模拟硬件设备性能要更高，因为你可以在VMM中设计设备接口使得并不需要太多的trap。
3. 第三个策略是对于真实设备的pass-through，这里典型的例子就是网卡。现代的网卡具备硬件的支持，可以与VMM运行的多个Guest操作系统交互。你可以配置你的网卡，使得它表现的就像多个独立的子网卡，每个Guest操作系统拥有其中一个子网卡。经过VMM的配置，Guest操作系统可以直接与它在网卡上那一部分子网卡进行交互，并且效率非常的高。所以这是现代的高性能方法。在这种方式中，Guest操作系统驱动可以知道它们正在与这种特别的网卡交互。

## Hardware-Support-Virtual-Machine

也就是为虚拟机提供直接的硬件支持，主要介绍Inter的VT-X。前面描述的Trap and Emulate虚拟机方案中，经常会涉及到大量高成本的trap，所以这种方案性能并不特别好。通过硬件上的支持，是为了让人们能够更容易地构建运行更快的虚拟机，现在在构建虚拟机时使用的非常非常广泛。**在Trap and Emulate方案中，VMM会为每个Guest在软件中保存一份虚拟状态信息，而现在，这些虚拟状态信息会保存在硬件中，而不用通过tarp走到VMM**。这样Guest中的软件可以直接执行privileged指令来修改保存在硬件中的虚拟寄存器，而不是通过trap走到VMM来修改VMM中保存在软件中的虚拟寄存器。

当我们使用这种新的硬件支持的方案时，我们的**VMM会使用真实的控制寄存器，而当VMM通知硬件切换到Guest mode时，硬件里还会有一套完全独立，专门为Guest mode下使用的虚拟控制寄存器**。在Guest mode下可以直接读写控制寄存器，但是读写的是寄存器保存在硬件中的拷贝，而不是真实的寄存器。硬件会对Guest操作系统的行为做一些额外的操作，以确保Guest不会滥用这些寄存器并从虚拟机中逃逸。Intel将Guest mode被称为non-root mode，Host mode中会使用真实的寄存器，被称为root mode。所以，硬件中保存的寄存器的拷贝，或者叫做虚拟寄存器是为了在non-root mode下使用，真实寄存器是为了在root mode下使用。

在VMM的内存中，通过一个结构体与VT-x硬件进行交互。这个结构体称为VMCS（Virtual Machine Control Structure）。当VMM要创建一个新的虚拟机时，它会先在内存中创建这样一个结构体，并填入一些配置信息和所有寄存器的初始值，VMM会告诉VT-x硬件说我想要运行一个新的虚拟机，并且虚拟机的初始状态存在于VMCS中。Intel通过一些新增的指令来实现这里的交互。

- VMLAUNCH，这条指令会创建一个新的虚拟机。你可以将一个VMCS结构体的地址作为参数传给这条指令，再开始运行Guest kernel。

- VMRESUME。在某些时候，Guest kernel会通过trap走到VMM，然后需要VMM中需要通过执行VMRESUME指令恢复代码运行至Guest kernel。

- VMCALL，这条新指令在non-root模式下使用，它会使得代码从non-root mode中退出，并通过trap走到VMM。比如定时器中断，VMM获得控制权，以及多个Guest分时共享CPU。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MVZ9QyUZmhZ4KkDb7zC%2F-MViVeDbwrTJvtFii3AE%2Fimage.png?alt=media&token=573b01cb-6af8-4dd5-9d74-d7010f5da75a">

VT-x机制中的另外一大部分是对于Page Table的支持。VT-x使得Guest可以加载任何想要的值到CR3寄存器，进而设置Page Table。而硬件也会执行Guest的这些指令，现在Guest kernel可以在不用通过trap走到VMM再来加载Page Table。但是我们也不能让Guest任意的修改它的Page Table，**所以VT-x的方案中，还存在另一个重要的寄存器：EPT（Extended Page Table）。EPT会指向一个Page Table。当VMM启动一个Guest kernel时，VMM会为Guest kernel设置好EPT，并告诉硬件这个EPT是为了即将运行的虚拟机准备的**。

之后，当计算机上的MMU在翻译Guest的虚拟内存地址时，它会先根据Guest设置好的Page Table，将Guest虚拟地址翻译到Guest 物理地址。之后再通过EPT，将Guest物理地址翻译成主机物理地址。硬件会为每一个Guest的每一个内存地址都自动完成这里的两次翻译。EPT使得VMM可以控制Guest可以使用哪些内存地址。Guest可以非常高效的设置任何想要的Page Table，因为它现在可以直接执行privileged指令。**但是Guest能够使用的内存地址仍然被EPT所限制，而EPT由VMM所配置，所以Guest只能使用VMM允许其使用的物理内存Page**。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MVitJVsv3i6fyJJ8mgb%2F-MVj6H2tHSyRzmPq5IOv%2Fimage.png?alt=media&token=dffda5be-a6bb-47ab-869f-18c2f2d680fc">
