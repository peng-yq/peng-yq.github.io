---
layout: post
title: "MIT 6.S081—网络"
subtitle: "Networking"
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - OS 
---

虽然本科专业是网络工程，但由于学校文科式的教学，对网络的了解仅限于八股文。Robert这节课讲的很好，甚至可以说让我豁然开朗，海大垃圾的本科教育:(

## Introduction

相近的主机连接在同一个局域网中。例如有一个以太网设备，可能是交换机或者单纯的线缆，然后有一些主机（笔记本、服务器或者路由器）连接到了这个以太网设备。每个主机上会有不同的应用程序，或许其中一个主机有网络浏览器，另一个主机有HTTP server，它们需要通过这个局域网来相互通信。一个局域网需要能让其中的主机都能收到彼此发送的packet，有时主机需要广播packet到局域网中的所有主机。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNxebj8T-Lc_jJYjLzL%2F-MNzmLt_xBZ6Q-fgSRZ4%2Fimage.png?alt=media&token=5f6c3ad7-0e4a-4a79-a4da-d9496b8f6566">

但一个局域网的大小是有极限的，局域网中只有25个甚至100个主机时，是没有问题的。但是你不能构建一个多于几百个主机的局域网。所以为了解决这个问题，大型网络是这样构建的：首先有多个独立的局域网，假设其中一个局域网是MIT，另一个局域网是Harvard，还有一个很远的局域网是Stanford，在这些局域网之间会有一些路由器Router将它们连接在一起。其中一个Router接入到了MIT的局域网，同时也接入到了Harvard的局域网。路由器是组成互联网的核心，路由器之间的链路，最终将多个局域网连接在了一起。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNxebj8T-Lc_jJYjLzL%2F-MNzob0uAH9YylTEG5Aw%2Fimage.png?alt=media&token=e21d16d9-4c08-4d73-917c-08bc6910f577">

现在MIT有一个主机需要与Stanford的一个主机通信，它们之间需要经过一系列的路由器，路由器之间的转发称为Routing。所以我们需要有一种方法让MIT的主机能够寻址到Stanford的主机，并且我们需要让连接了MIT的路由器能够在收到来自MIT的主机的packet的时候，能够知道这个packet是发送给Harvard的呢，还是发送给Stanford的。从网络协议的角度来说，局域网通信由以太网协议决定；而局域网之上的长距离网络通信由IP协议决定。

## Ethernet

当两个主机非常靠近时，通过相同的线缆、同一个wifi网络或同一个以太网交换机连接。当局域网中的两个主机彼此间要通信时，最底层的协议是以太网协议。Host1通过以太网将Frame发送给Host2。Frame是以太网中用来描述packet的概念，本质上这就是两个主机在以太网上传输的一个个的数据Byte。以太网协议会在Frame中放入足够的信息让主机能够识别彼此，并且识别这是不是发送给自己的Frame。每个以太网packet在最开始都有一个Header，其中包含了3个数据。Header之后才是payload数据。Header中的3个数据是：目的以太网地址，源以太网地址，以及packet的类型。

```c
#define ETHADDR_LEN 6

struct eth {
	uint8 dhost[ETHADDR_LEN];      // 48位（8*6，也就是每两个16进制为一组，共6组）
 	 uint8 shost[ETHADDR_LEN]; 
	uint16 type;
} _attribute_((packed));

#define ETHTYPE_IP     0x0800  // IP协议
#define ETHTYPE_ARP 0x0806  // ARP协议
```

> attribute((packed)) 是GCC编译器的一种扩展语法，用于告诉编译器在内存中紧凑地存储结构体或变量，而不进行对齐操作。它指示编译器以紧凑的方式存储eth该结构体。这意味着各成员变量之间没有额外的填充字节，dhost、shost 和 type 成员将以连续的方式存储。这在网络协议的数据包处理中非常常见，因为网络协议要求数据的字节顺序和内存布局是紧凑和连续的。

每一个以太网地址（mac地址）都是48位的数字，例如00-16-EA-AE-3C-40，这个数字唯一识别了一个网卡。16进制数00-16-EA代表网络硬件制造商的编号，它由IEEE分配；而后3个字节，16进制数AE-3C-40代表该制造商所制造的某个网络产品(如网卡)的系列号。packet的类型会告诉接收端的主机该如何处理这个packet。接收端主机侧更高层级的网络协议会按照packet的类型检查并处理以太网packet中的payload。

整个以太网packet，包括了48bit+48bit的以太网地址，16bit的类型，以及任意长度的payload。除此之外，虽然对于软件来说是不可见的，但是在packet的开头还有被硬件识别的表明packet起始的数据（注，Preamble + SFD），在packet的结束位置还有几个bit表明packet的结束（注，FCS）。packet的开头和结束的标志不会被系统内核所看到，其他的部分会从网卡送到系统内核。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MNzsFhzcgT0Lqe-M1YG%2F-MO40ML0u3lTAh70m_1u%2Fimage.png?alt=media&token=bdf22df8-bdbd-454d-b9f6-7a3374932edb">

通过tcpdump查看以太网帧。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO40Vq8POBpsayQpuQp%2F-MO8zlyQREcVbqVrEmiE%2Fimage.png?alt=media&token=74a61a3f-838f-4683-abb5-b0011ed50ba9">

- 第一行是人类可读的信息，包括接收packet的时间、协议、packet的目的和长度
- 前48bit是一个广播地址，0xffffffffffff。广播地址是指packet需要发送给局域网中的所有主机。
- 之后的48bit是发送主机的以太网地址，0x52550a000202。
- 接下来的16bit是以太网packet的类型，这里的类型是0x0806，对应的协议是ARP。
- 剩下的部分是ARP packet的payload。

虽然以太网地址是唯一的，但是出了局域网，它们对于定位目的主机的位置是没有帮助的。如果网络通信的目的主机在同一个局域网，那么目的主机会监听发给自己的地址的packet。但是如果网络通信发生在两个国家的主机之间，那就需要IP协议了。

## ARP

在以太网层面，每个主机都有一个以太网地址。但是为了能在互联网上通信，你需要有32位的IP地址。IP地址的高位bit包含了在整个互联网中唯一的网络号，路由器会检查IP地址的高bit位，并决定将这个packet转发给互联网上的哪个路由器。IP地址的低bit位代表了在局域网中特定的主机。当一个经过互联网转发的packet到达了局域以太网，我们需要从32bit的IP地址，找到对应主机的48bit以太网地址。这里是通过一个动态解析协议完成的，也就是Address Resolution Protocol，ARP协议。当一个packet到达路由器并且需要转发给同一个以太网中的另一个主机，或者一个主机将packet发送给同一个以太网中的另一个主机时。发送方首先会在局域网中广播一个ARP packet，来表示任何拥有了这个32bit的IP地址的主机，请将你的48bit以太网地址返回过来。如果相应的主机存在并且开机了，它会向发送方发送一个ARP response packet。

```c
struct arp{
    uint16 hrd;    // format of hardware address
    uint16 pro;    // format of protocol address
    uint8   hln;    // length of hardware address
    uint8   pln;    // length of protocol address
    uint16 op;     // operation
    
    char sha[ETHADDR_LEN];    // sender hardware address
    uint32 sip;                                  // sender IP address
    char tha[ETHADDR_LEN];    // target hardware address
    uint32 tip;                                  // target IP address
}_attribute_((packed));

#define ARP_HRD_ETHER 1 // Ethernet

enum {
    ARP_OP_REQUEST = 1, // request hw addr given protocol addr
    ARP_OP_REPLY = 2,       // replies a hw addr given protocol addr
};
```

ARP包会出现在一个以太网packet的payload中。接收到packet的主机通过查看以太网header中的16bit类型（0x0806）可以知道这是一个ARP packet，并将这个packet发送给ARP协议处理代码。有关ARP packet的内容，包含了不少信息，但是基本上就是在说，现在有一个IP地址，我想将它转换成以太网地址，如果你拥有这个IP地址，请响应我。

同样的，我们也可以通过tcpdump来查看这些packet。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO40Vq8POBpsayQpuQp%2F-MO97sCnlq2lUQVrRkzq%2Fimage.png?alt=media&token=0f6d13f3-a37d-487e-9bcc-a2538e471240">

tcpdump能够解析出ARP packet，并将数据打印在第一行。对应ARP packet的格式，在第一个packet中，10.0.2.2是SIP，10.0.2.15是DIP。在第二个packet中，52:54:00:12:34:56对应SHA。

同时，我们也可以自己分析packet的原始数据。对于第一个packet：

- 前14个字节是以太网header，包括了48bit目的以太网地址，48bit源以太网地址，16bit类型（0x806表明是ARP协议）。
- 从后往前看，倒数4个字节0a00020f（10.0.2.15）是TIP，也就是发送方想要找出对应以太网地址的IP地址。
- 再向前数6个字节，是THA，也就是目的地的以太网地址，现在还不知道所以是全0。
- 再向前数4个字节是SIP，也就是发送方的IP地址，0a000202（10.0.2.2）。
- 再向前数6个字节是SHA，也就是发送方的以太网地址。
- 剩下的8个字节0x0001 0800 0604 0001表明了以太网和IP地址格式、长度以及ARP类型（01）。

第二个packet是第一个packet的响应。

> 学生提问：ethernet header中已经包括了发送方的以太网地址，为什么ARP packet里面还要包含发送方的以太网地址？
>
> Robert教授：我并不清楚为什么ARP packet里面包含了这些数据，我认为如果你想的话是可以精简一下ARP packet。或许可以这么理解，**ARP协议被设计成也可以用在其他非以太网的网络中，所以它被设计成独立且不依赖其他信息，所以ARP packet中包含了以太网地址。现在我们是在以太网中发送ARP packet，以太网packet也包含了以太网地址，所以，如果在以太网上运行ARP，这些信息是冗余的。但是如果在其他的网络上运行ARP，你或许需要这些信息，因为其他网络的packet中并没有包含以太网地址**。
>
> 学生提问：tcpdump中原始数据的右侧是什么内容？
>
> Robert教授：这些是原始数据对应的ASCII码，“.”对应了一个字节并没有相应的ASCII码，0x52对应了R，0x55对应了U。当我们发送的packet包含了ASCII字符时，这里的信息会更加有趣。

从上面ARP包格式可以看到，网络协议和网络协议header是嵌套的。ARP包中，在ethernet payload中，首先出现的是ARP header，对于ARP来说并没有的payload。但是在ethernet packet中还可以包含其他更复杂的结构，比如说ethernet payload中包含一个IP packet，IP packet中又包含了一个UDP packet，所以IP header之后是UDP header。如果在UDP中包含另一个协议，那么UDP payload中又可能包含其他的packet，例如DNS packet。整个packet是在发送过程中逐渐构建起来的。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MO9AUbfTj-IlKsNex3Z%2F-MOEQdZYLWbFzmTOqCsg%2Fimage.png?alt=media&token=bb58cc93-8ee6-4e14-88bb-0bb32b5499d9">

类似的，当一个操作系统收到了一个packet，它会先解析第一个header并知道这是Ethernet，经过一些合法性检查之后，Ethernet header会被剥离，操作系统会解析下一个header。软件会解析每个header，做校验，剥离header，并得到下一个header。一直重复这个过程直到得到最后的数据。这就是嵌套的packet header。

## IP

Ethernet header足够在一个局域网中将packet发送到一个host。如果你想在局域网发送一个IP packet，那么你可以使用ARP获得以太网地址。但是IP协议更加的通用，IP协议能帮助你向互联网上任意位置发送packet。

```c
struct ip {
  uint8  ip_vhl; // version << 4 | header length >> 2
  uint8  ip_tos; // type of service
  uint16 ip_len; // total length
  uint16 ip_id;  // identification
  uint16 ip_off; // fragment offset field
  uint8  ip_ttl; // time to live
  uint8  ip_p;   // protocol
  uint16 ip_sum; // checksum
  uint32 ip_src, ip_dst;
};

#define IPPROTO_ICMP 1  // Control message protocol
#define IPPROTO_TCP  6  // Transmission control protocol
#define IPPROTO_UDP  17 // User datagram protocol
```

[IP头部详解](https://zhuanlan.zhihu.com/p/371723473)

在一个packet发送到世界另一端的网络的过程中，IP header会被一直保留，而Ethernet header在离开本地的以太网之后会被剥离。或许packet在被路由的过程中，在每一跳（hop）会加上一个新的Ethernet header。**但是IP header从源主机到目的主机的过程中会一直保留**。

IP header具有全局的意义，而Ethernet header只在单个局域网有意义。所以IP header必须包含足够的信息，这样才能将packet传输给互联网上遥远的另一端。对于我们来说，关键的信息是三个部分，目的IP地址（ip_dst），源IP地址（ip_src）和协议（ip_p）。目的IP地址是我们想要将packet送到的目的主机的IP地址。地址中的高bit位是网络号，它会帮助路由器完成路由。IP header中的协议字段会告诉目的主机如何处理IP payload。

接下来我们看一下包含了IP packet的tcpdump输出。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MOEZ9iqKkt92zbewl4r%2F-MOPw1qnEjiOAkSTe3Yi%2Fimage.png?alt=media&token=913186dd-c85a-4fd8-bc3f-31e21b8744df">

> 上面这个包有点问题，以太网头中的目的以太网地址不应该是广播地址

从后向前看：

- 目的IP地址是0x0a000202，也就是10.0.2.2。
- 源IP地址是0x0a00020f，也就是10.0.2.15。
- 再向前有16bit的checksum，也就是0x3eae。IP相关的软件需要检查这个校验和，如果结果不匹配应该丢包。
- 再向前一个字节是protocol，0x11对应的是10进制17，表明了下一层协议是UDP

IP header中的protocol字段告诉了目的主机的网络协议栈，这个packet应该被UDP软件处理。

## UDP

IP header足够让一个packet传输到互联网上的任意一个主机，但是我们希望做的更好一些。每一个主机都运行了大量需要使用网络的应用程序，所以我们需要有一种方式能区分一个packet应该传递给目的主机的哪一个应用程序，而IP header明显不包含这种区分方式。有一些其他的协议完成了这里的区分工作，其中一个是TCP，它比较复杂，而另一个是UDP。TCP不仅帮助你将packet发送到了正确的应用程序，同时也包含了序列号等用来检测丢包并重传的功能，这样即使网络出现问题，数据也能完整有序的传输。相比之下，UDP就要简单的多，它以一种“尽力而为”的方式将packet发送到目的主机，除此之外不提供任何其他功能。UDP header中最关键的两个字段是sport源端口和dport目的端口。

[TCP和UDP区别](https://jaminzhang.github.io/network/The-Difference-Between-TCP-And-UDP-Protocol/)

相较于TCP，UDP是面向无连接的、不可靠的，仅适用于对数据传输延迟较高、对数据可靠性要求较低的场景。

采用UDP的应用层协议有很多，以下是一些常见的例子：

1. DNS（Domain Name System）：用于将域名解析为IP地址，或将IP地址解析为域名的协议。DNS使用UDP作为传输层协议，通常在端口53上进行通信。
2. DHCP（Dynamic Host Configuration Protocol）：用于自动分配IP地址和其他网络配置信息给客户端设备的协议。DHCP在局域网中使用UDP广播进行通信，通常使用端口67和68。
3. TFTP（Trivial File Transfer Protocol）：一种简单的文件传输协议，通常用于在局域网中进行快速文件传输。TFTP使用UDP作为传输层协议，在端口69上进行通信。
4. SNMP（Simple Network Management Protocol）：用于网络设备的管理和监控的协议。SNMP使用UDP作为传输层协议，通常在端口161和162上进行通信。

```c
struct udp {
  uint16 sport; // source port
  uint16 dport; // destination port
  uint16 ulen;  // length, including udp header, not including IP header
  uint16 sum;   // checksum
};
```

当你的应用程序需要发送或者接受packet，它会使用socket API，这包含了一系列的系统调用。一个进程可以使用socket API来表明应用程序对于特定目的端口的packet感兴趣。**当应用程序调用这里的系统调用，操作系统会返回一个文件描述符。每当主机收到了一个目的端口匹配的packet，这个packet会出现在文件描述符中，之后应用程序就可以通过文件描述符读取packet**。

这里的端口分为两类：

- 一类是常见的端口，例如53对应的是DNS服务的端口，如果你想向一个DNS server发请求，你可以发送一个UDP packet并且目的端口是53。除此之外，很多常见的服务都占用了特定的端口。
- 除了常见端口，16bit数的剩下部分被用来作为匿名客户端的源端口。

接下来我们看一下UDP packet的tcpdump输出。首先，我们同样会有一个以太网Header，以及20字节的IP header。IP header中的0x11表明这个packet的IP协议号是17，这样packet的接收主机就知道应该使用UDP软件来处理这个packet。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MOEZ9iqKkt92zbewl4r%2F-MOQ8CJUdG6ZCj_7xuAV%2Fimage.png?alt=media&token=aad064fd-585c-45aa-a5b1-f4cf087a2c15">

0x07d0开始的8个字节是UDP header。这里的packet是由lab代码生成的packet，所以它并没有包含常见的端口，源端口是0x07d0，目的端口是0x6403。第4-5个字节是长度，第6-7个字节是校验和。XV6的UDP软件并没有生成UDP的校验和。UDP header之后就是UDP的payload。在这个packet中，应用程序发送的是ASCII文本，所以我们可以从右边的ASCII码看到，内容是“a.message.from.xv6”。所以ASCII文本放在了一个UDP packet中，然后又放到了一个IP packet中，然后又放到了一个Ethernet packet中。最后发布到以太网上。

## Network Stack

假设我们现在在运行Linux或者XV6，我们有一些应用程序比如浏览器，DNS服务器。这些应用程序使用socket API打开了socket layer的文件描述符。Socket layer是内核中的一层软件，它会维护一个表单来记录文件描述符和UDP/TCP端口号之间的关系。同时它也会为每个socket维护一个队列用来存储接收到的packet。

在socket layer之下是UDP和TCP协议层。UDP软件几乎不做任何事情，它只是检查收到的packet，获取目的端口号，并将UDP payload传输给socket layer中对应的队列。TCP软件会复杂的多，它会维护每个TCP连接的状态，比如记录每个TCP连接的序列号，哪些packet没有被ACK，哪些packet需要重传。所以TCP的协议控制模块会记录大量的状态，但是UDP中不会记录任何状态。UDP和TCP通常被称为传输层。在TCP/UDP之下是IP层，IP层的软件通常很简单。与IP层在一起的还有ARP层。再往下的话，我们可以认为还会有一层以太网。但是通常并没有一个独立的以太网层。通常来说这个位置是一个或者多个网卡驱动，这些驱动与实际的网卡硬件交互。网卡硬件与局域网会有实际的连接。

<img src="https://906337931-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MOQGBkHzTXklQuHE9BY%2F-MORMgwY2ntXxHwcJ5rZ%2Fimage.png?alt=media&token=18294f8d-3a68-4a82-bc8d-a2dc31150d96">

当一个packet从网络送达时，网卡会从网络中将packet接收住并传递给网卡驱动。网卡驱动会将packet向网络协议栈上层推送。在IP层，软件会检查并校验IP header，将其剥离，再把剩下的数据向上推送给UDP。UDP也会检查并校验UDP header，将其剥离，再把剩下的数据加入到socket layer中相应文件描述符对应的队列中。**所以一个packet在被收到之后，会自底向上逐层解析并剥离header。当应用程序发送一个packet，会自顶向下逐层添加header，直到最底层packet再被传递给硬件网卡用来在网络中传输。所以内核中的网络软件通常都是被嵌套的协议所驱动**。

在整个处理流程中都会有packet buffer。所以当收到了一个packet之后，它会被拷贝到一个packet buffer中，这个packet buffer会在网络协议栈中传递。通常在不同的协议层之间会有队列，比如在socket layer就有一个等待被应用程序处理的packet队列，这里的队列是一个linked-list。通常整个网络协议栈都会使用buffer分配器，buffer结构。