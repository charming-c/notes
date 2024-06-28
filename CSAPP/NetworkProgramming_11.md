# 网络编程

网络应用依赖于很多在系统研究中已经学到过的概念。例如，进程、信号、字节顺序内存映射以及动态内存分配，都扮演着重要的角色。还有一些新的概念要掌握。我们需要理解基本的客户端-服务器编程模型，以及如何编写使用因特网提供服务的客户端-服务器程序。

## 一、客户端-服务器编程模型

每个网络应用都是基于客户端-服务器模型的。采用这个模型，一个应用是由一个服务器进程和一个或者多个客户端进程组成。服务器管理某种资源，并且通过操作这种资源来为它的客户端提供某种服务。例如，一个 Web 服务器管理着一组磁盘文件，它会代表客户端进行检索和执行。一个 FTP 服务器管理着一组磁盘文件，它会为客户端进行存储和检索。相似地，一个电子邮件服务器管理着一些文件，它为客户端进行读和更新。

客户端-服务器模型中的基本操作时事物（transaction）。一个客户端-服务器事务由以下四步组成。

1. 当一个客户端需要服务时，它向服务器发送一个请求，发起一个事务。例如，当 Web 浏览器需要一个文件时，它就发送一个请求给 Web 服务器。
2. 服务器收到请求后，解释它，并以适当的方式操作它的资源。例如，当 Web 服务器收到浏览器发出的请求后，它就读一个磁盘文件。
3. 服务器给客户端发送一个响应，并等待下一个请求。例如，Web 服务器将文件发送回客户端。
4. 客户端收到响应并处理它。例如，当 Web 浏览器收到来自服务器的一页后，就在屏幕上显示此页。

![image-20231212222759760](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231212222759760.png)

认识到客户端和服务端是进程，而不是常提到的机器或者主机，这是很重要的。一台主机可以同时运行许多不同的客户端和服务器，而且一个客户端和服务器的事务可以在同一台或是不同的主机上。无论客户端和服务器是怎样映射到主机上的，客户端-服务器模型都是相同的。

> 客户端-服务器事务不是数据库事务，没有数据库事务的任何特性，例如原子性。在我们的上下文中，事务仅仅是客户端和服务器执行的一些列步骤。

## 二、网络

客户端和服务器通常运行在不同的主机上，并且通过计算机网络的硬件的软件资源来通信。网络是很复杂的系统，在这里我们只想了解一点皮毛。我们的目标是从程序员的角度给你一个切实可行的思维模型。

对主机而言，网络只是又一种 I/O 设备，是数据源和数据接收方，如下图所示。一个插到 I/O 总线扩展槽的适配器提供了到网络的物理接口。从网络上接收到的数据从适配器经过 I/O 和内存总线复制到内存，通常是通过 DMA 传送。相似地，数据也能从内存复制到网络。

![image-20231212223720502](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231212223720502.png)

物理上而言，网络是一个按照地理远近组成的层次系统。最低层是 LAN（Local Area Network，局域网），在一个建筑或者校园范围内。迄今为止，最流行的局域网技术是以太网（Ethernet）。

一个以太网段（Ethernet segment）包括一个电缆（通常是双绞线）和一个叫做集线器的小盒子，如下图所示。以太网通常跨越一些小的区域，例如某个建筑物的一个房间或者一个楼层。每根电缆都有相同的最大位带宽，通常是 100MB/s 或者 1GB/s。一端连接到主机的适配器，而另一端则连接到集线器的一个端口上。集线器不加分辨地将从一个端口上收到的每个位复制到其他所有端口上。因此，每台主机都能看到每个位。

![image-20231212224455053](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231212224455053.png)

每个以太网适配器都有一个全球唯一的 48 位地址，它存储在这个适配器的非易失性存储器上。一台主机可以发送一段位（称为帧（frame））到这个网段内的其他任何主机。每个帧包括一些固定数量的头部位，用来表示此帧的源和目的地址以及此帧的长度，此后紧随的就是数据位的有效载荷。每个主机适配器都能看到这个帧，但是只有目的主机实际读取它。

使用一些电缆和叫做网桥和小盒子，多个以太网段可以连接成较大的局域网，称为桥接以太网（bridged Ethernet），如下图所示。桥接以太网能够跨越整个建筑物或者校区。在一个桥接以太网里，一些电缆连接网桥与网桥，而另外一些连接网桥和集线器。这些电缆的带宽可以不同的。在我们的示例中，网桥和网桥之间的电缆有 1GB/s 的带宽，而四根网桥和集线器之间的电缆的带宽却是 100MB/s。

![image-20231212225431843](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231212225431843.png)

网桥比集线器更充分地利用了电缆带宽。利用一种聪明的分配算法，它们随着时间自动学习哪个主机可以通过哪个端口可达，然后只有在有必要时，有选择地将帧从一个端口复制到另一个端口。例如，如果主机 A 发送一个人帧到同网段上的主机 B，当该帧到达网桥 X 的输入端口时，X 就会丢弃这个帧，因而减少其他网段上的带宽。然而，如果主机 A 发送一个帧到一个不同网段的主机 C，那么网桥 X 只会把此帧复制到网桥 Y相连的端口上，网桥 Y 会只把此帧复制到与主机 C 的网段连接的端口。

在层次更高级别中，多个不兼容的局域网可以通过叫做路由器（router）的特殊计算机连接起来，组成一个互联网络（internet）。每台路由器对于它所连接到的每个网络都有一个适配器（端口）。路由器也能连接告诉点到点电话连接，这是称为 WAN（Wide-Area-Network，广域网）的网络示例。

![image-20231212230612539](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231212230612539.png)

互联网至关重要的特性是，它能由采用完全不同和不兼容技术的各种局域网和广域网组成。每台主机和其他每台主机都是物理相连的，但是如何能够让某台源主机跨过所有这些不兼容的网络发送数据到另一台目的主机呢？

解决方法是一层运行在每台主机和路由器上的协议软件，它消除了不同网络之间的差异。这个软件实现一种协议，这种协议控制主机和路由器如何协同工作来实现数据传输。这种协议必须提供两种基本能力：

- 命名机制。不同的局域网技术有不同和不兼容的方式来为主机分配地址。互联网络协议通过定义一种一致的主机地址格式消除了这些差异。每台主机都会被分配至少一个这种互联网络地址（internet address），这个地址唯一地标识了这台主机。
- 传送机制。在电缆上编码位和将这些位封装成帧方面，不同的联网技术有不同的和不兼容的方式。互联网络协议通过定义一种把数据位捆扎成不连续的片（称为包）的统一方式，从而消除了这种差异。一个包由包头和有效载荷组成的，其中包头包括包的大小以及源主机和目的主机的地址，有效载荷包括从源主机发出的数据位。

下图展示了主机和路由器如何使用互联网络协议在不兼容的局域网间传送数据的一个示例。这个互联网络示例由两个局域网通过一台路由器连接而成。一个客户端运行在主机 A 上，主机 A 和 LAN1 相连，他发送一串数据字节到运行在主机 B 上的一个服务器端，主机 B 则连接在 LAN2 上。这个过程有 8 个基本步骤。

![image-20231212231856799](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231212231856799.png)

1. 运行在主机 A 上的客户端进行一个系统调用，从客户端的虚拟地址空间复制数据到内核缓冲区
2. 主机 A 上的协议软件通过在数据前附加互联网包头和 LAN1 帧头，创建一个 LAN1 的帧。互联网络包头寻址到互联网络主机 B。LAN1 帧头寻址到路由器。然后传送此帧到适配器。注意，LAN1 帧的有效载荷是一个互联网络包，而互联网络包的有效载荷是实际的用户数据。这种封装是基本的网络互联方法之一。
3. LAN1 适配器复制该帧到网络上。
4. 当此帧到达路由器时，路由器的 LAN1 适配器从电缆上读取它，并把它传送到协议软件。
5. 路由器从互联网络包头中提取处目的互联网络地址，并用它作为路由表的索引，确定向哪里转发这个包，在本例中时 LAN2 路由器剥落旧的 LAN1 帧头，加上寻址到主机 B 的新的 LAN2 帧头，并把得到的帧传送到适配器。
6. 路由器的 LAN2 适配器复制该帧到网络上。
7. 当此帧到达主机 B 时，它的适配器从电缆上读到此帧，并将它传送到协议软件。
8. 最后，主机 B 上的协议软件剥落包头和帧头，当服务器进行一个读取这些数据的系统调用时，协议软件最终将得到的数据复制到服务器的虚拟地址空间。

## 三、全球 IP 因特网

下图展示了因特网客户端-服务器应用程序的基本硬件和软件组织。

![image-20231213233523550](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231213233523550.png)

每台因特网主机都运行实现了 TCP/IP 协议（Transmission Control Protocol/Internet Protocol），传输控制协议/互联网络协议）的软件，几乎每个现代计算机系统都支持这个协议。因特网的客户端和服务器混合使用套接字接口函数和 Unix I/O 函数来进行通信。通常将套接字函数实现为系统调用，这些系统调用会陷入内核，并调用各种内核模式的 TCP/IP 函数。

TCP/IP 实际是一个协议族，其中每一个都提供了不同的功能。例如，IP 协议提供基本的命名方法和递送机制，这种递送机制能够从一台因特网主机往其他主机发送包，也叫做数据报（datagram）。IP 机制从某种意义上而言是不可靠的，因为，如果数据报在网络中丢失或者重复，它并不会试图修复。UDP（Unreliable Datagram Protocol，不可靠数据报协议）稍微扩展了 IP 协议，这样一来，包可以在进程间而不是主机间传送。TCP 是一个构建在 IP 之上的复杂协议，提供了进程间可靠的双全工连接。为了简化讨论，我们将 TCP/IP看做是一个单独的整体协议。我们将不讨论它的内部工作，只讨论 TCP 和 IP 为应用程序提供的某些基本功能。我们将不讨论 UDP。

从程序员的角度，我们可以把因特网看作是一个世界范围的主机集合，满足以下特性：

- 主机集合被映射为一组 32 位的 IP 地址。
- 这组 IP 地址被映射为一组称为因特网域名（Internet domain name）的标识符。
- 因特网主机上的进程能够通过连接到任何其他因特网主机上的进程通信。

### 1. IP 地址

一个 IP 地址就是一个 32 位无符号整数。网络程序将 IP 地址存放在下图所示的 IP 地址结构中：

```c
	struct in_addr {
  uint32_t s_addr;
};
```

把一个标量地址存放在结构中，是套接字接口早期实现的不幸产物。为 IP 地址定义一个标量类型应该更有意义，但是现在更改已经太迟了，因为已经有大量的应用是基于此的。

因为因特网主机可以有不同的主机字节顺序，TCP/IP 为任意整数数据项定义了统一的网络字节顺序（network byte order）（大端字节顺序），例如 IP 地址，它可以放在包头中跨过网络被携带。在 IP 地址结构中存放的地址总是以（大端法）网络字节顺序存放的，即使主机字节顺序是小端法。Unix 提供了下面这样的函数在网络和主机字节顺序件实现转换。

```c
#include<arpa/inet.h>

uint32_t htonl(uint32_t hostlong);
uint16_t htons(uint16_t hostshort);
// 返回：按照网络字节顺序的值

uint32_t ntonl(uint32_t netlong);
uint16_t ntons(uint16_t netshort);
// 返回：按照主机字节顺序的值
```

IP 地址是以一种点分十进制表示法来表示的，这里，每个字节由他的十进制值表示，并且用句点和其他字节区分开来。例如，128.2.194.242 就是地址 0x8002c2f2 的点分十进制表示。在 Linux 系统上，你能够使用 HOSTNAME 命令来确定你的主机的点分十进制地址：

linux> hostname -i

应用程序使用 inet_pton 和 inet_ntop 函数来实现 IP 地址和点分十进制串之间的转换。

```c
#include<arpa/inet.h>

int inet_pton(AF_INET, const char *src, void *dst);
// 返回：若成功则为 1，若 src 为非法点分二进制地址则为 0，若出错则为 -1。

const char *inet_ntop(AF_INET, const void *src, char *dst, socklen_t size);
// 返回：若成功则指向点分十进制字符串的指针，若出错则为 NULL。
```

在这些函数名中，n 代表网络，p 代表表示。它们可以处理 32 位的 IPv4 地址（AF_INET），或者 128 位 IPv6 地址（AF_INET6）。

inet_pton 函数将一个点分十进制串 src ，转换为一个二进制的网络字节顺序的 IP 地址 dst。如果 src 没有指向一个合法的点分十进制字符串，那么该函数就返回 0，任何其他错误返回 -1，并设置 errno。相似地，inet_ntop 函数将一个二进制的网络字节顺序的 IP 地址 src 转换为他所对应的点分十进制表示，并把得到的以 null 结尾的字符串最多 size 个字节复制到 dst。

### 2. 因特网域名

因特网客户端和服务器通信时使用的是 IP 地址。然而，对于人们而言，大整数是很难记住的，所以因特网也定义了一组桁架个性化的域名（domain name），以及一种将域名映射到 IP 地址的映射机制。域名是一串用句点分隔的单词（字母、数字和破折号）。

域名集合形成了一个层次结构，每个域名编码了它在这个层次中的位置。下图展示了域名层次结构的一部分。层次结构可以表示为一棵树。树的节点表示域名，反向到根的节点形成了域名。子树称为子域。层次结构中的第一层是一个未命名的根节点。下一层是一组一级域名，由非营利组织 ICANN 定义，常见的一级域名包括 com、edu、gov、org、net。

![image-20231214113715700](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214113715700.png)

下一层是二级域名，例如 com.edu，这些域名是由 ICANN 的各个授权代理按照先到先服务的基础分配的。一旦一个组织得到了一个二级域名，那么它就可以在这个子域中创建任何新的域名了。

因特网定义了域名集合和 IP 地址集合之间的映射。直到 1988 年，这个映射都是通过一个叫做 HOSTS.TXT 的文本文件手工维护的。从那以后，这个映射通过分布世界范围内的数据库（称为 DNS，域名系统）来维护的。从概念上看，DNS 数据库由上百万的主机条目结构组成，其中每条定义了一组域名和一组 IP 地址之间的映射。从数学意义上讲，可以认为每条主机条目就是一个域名和 IP 地址的等价类。我们可以用 Linux 的 NSLOOKUP 程序来探究 DNS 映射的一些属性，这个程序能展示与某个 IP 对应的域名。

每台因特网主机都有本地域名 localhost，这个域名总是映射为回送地址 127.0.0.1：

![image-20231214114702176](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214114702176.png)

localhost 名字为引用运行在同一台机器上的客户端和服务器提供了一种便利和可移植的方式，这对调试相当有用。我们可以使用 HOSTNAME 来确定本地主机的实际域名。

在最简单的情况下，一个域名和一个 IP 地址之间是 一一映射的，然而在某些情况下，多个域名可以映射为同一个 IP 地址：

![image-20231214115004373](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214115004373.png)

在最通常的情况下，多个域名可以映射到同一组的多个 IP 地址：

![image-20231214115043464](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214115043464.png)

最后，我们注意到某些合法的域名没有映射到任何 IP 地址。

### 3. 因特网连接

因特网客户端和服务器通过在连接上发送和接收字节流来通信。从连接一对进程的意义上而言，连接是点对点的。从数据可以同时双向流动的角度来说，它是双全工的。并且从源进程发出的字节流最终被目的进程以它发出的顺序收到他的角度来说，它是可靠的。

一个套接字是连接的一个端点。每个套接字都有相应的套接字地址，是由一个因特网地址和一个 16 位的整数端口组成，用 ”地址：端口“来表示。

当客户端发起一个连接请求时，客户端套接字地址中的端口是由内核自动分配的，称为临时端口（ephemeral port）。然而，服务器套接字地址中的端口通常是某个知名端口，是和这个服务相对应的。例如，Web 服务器通常使用端口 80，而电子邮件服务器使用端口 25。每个具有知名端口的服务都有一个对应的知名的服务名。例如，Web 服务的知名名字是 http，email 的知名名字是 smtp。文件 /etc/sevices 包含一张这台机器提供的知名名字和知名端口之间的映射。

一个连接是由它两端的套接字地址唯一确定的。这对套接字地址叫做套接字对（socket pair），由下列元组组成：

（cliaddr：cliport，servaddr：servport）

![image-20231214162107533](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214162107533.png)

## 四、套接字接口

套接字接口是一组函数，它们和 Unix I/O 函数结合起来，用以创建网络应用。大多数现代系统上都实现套接字接口，包括所有的 Unix 变种、Windows 和 Macintosh 系统。下图给出一个典型的客户端-服务器事务的上下文的套接字接口概述。

![image-20231214164050420](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214164050420.png)

### 1. 套接字地址结构

从 Linux 内核的角度来看，一个套接字就是通信的一个端点。从 Linux 程序的角度来看，套接字就是一个有相应描述符的打开文件。

因特网的套接字地址存放在下图所示的类型为 sockaddr_in 的 16 字节的结构中。对于因特网应用，sin_family 成员是 AF_INET，sin_port 成员是一个 16 位的端口号，而 sin_addr 成员就是一个 32 位的 IP 地址。IP 地址和端口号总是以网络字节顺序（大端法）存放的。

![image-20231214164905794](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214164905794.png)

connect、bind 和 accept 函数要求一个指向与协议相关的套接字地址结构的指针。套接字接口的设计着面临的问题是，如何定义这些函数，使之能够接收各种类型的套接字结构地址。今天我们可以使用通用的 void * 指针，但是在那时 C 中并不存在这种类型的指针。解决办法是定义套接字函数要求一个指向通用 sockaddr 结构的指针，然后，要求应用程序将与协议特定的结构的指针强制转换成这个通用结构。为了简化代码示例，定义下面的类型：

typedef struct sockaddr SA；

然后无论何时需要将 sockaddr_in 结构转化为通用的 sockaddr结构时，我们都使用这个类型。

### 2. socket 函数

客户端和服务器使用 socket 函数来创建一个套接字描述符（socket descriptor）。

```c
#include<sys/types.h>
#include<sys/socket.h>

int socket(int domain, int type, int protocol);
// 返回：若成功则为非负描述符，若出错则为 -1。
```

如果想要套接字成为连接的一个端点，就用如下的硬编码参数来调用 socket 函数：

clientfd = Socket(AF_INET, SOCKET_STREAM, 0);

其中，AF_INET 表明我们正在使用 IPv4 地址，而 SOCKET_STREAM 表示这个套接字是连接的一个端点。不过最好的方法是用 getaddrinfo 函数来自动生成这些参数。

socket 函数返回的 clientfd 描述符仅仅是部分打开的，还不能用于读写。如何完成打开套接字的工作，取决于我们是客户端还是服务器，下一节我们描述当我们是客户端时如何完成套接字打开工作。

### 3. connect 函数

客户端通过 connect 来建立和服务器的连接。

```c
#include<sys/socket.h>

int connect(int clientfd, const struct socketaddr* addr, socklen_t addlen);
// 返回：若成功则为 0，若出错则为 -1。
```

connect 函数试图与套接字地址为 addr 的服务器建立一个因特网连接，其中 addrlen 是 sizeof(socketaddr_in)。connect 函数会阻塞，一直到连接成功建立或是发生了错误。如果成功，clientfd 描述符现在就准备好可以读写了，并且得到的连接是由套接字对：（x：y，addr.sin_addr：addr.sin_port）刻画的，其中 x 表示客户端的 IP 地址，而 y 表示临时端口，它唯一地确定了客户端主机上的客户端进程。对于 socket ，最好的办法是用 getaddrinfo来为 connect 提供参数。

### 4. bind 函数

剩下的套接字函数 bind、listen 和 accept，服务器用它们来和客户端建立连接。

```c
#include<sys/socket.h>

int bind(int sockfd, const struct socketaddr* addr, socklen_t addlen);
// 返回：若成功则为 0，若出错则为 -1。
```

bind 函数告诉内核将 addr 中的服务器套接字地址和套接字描述符 sockfd 联系起来。参数 addlen 就是 sizeof（sockaddr_in）。对于 socket 和 connect ，最好的方法是用 getaddrinfo 来为 bind 提供参数。

### 5. listen 函数

客户端是发起请求连接的主动实体。服务器是等待来自客户端的连接请求的被动实体。默认情况下，内核会认为 socket 函数创建的描述符对应于主动套接字（active socket），它存在于一个连接的客户端。服务器调用 listen 函数告诉内核，描述符是被服务器而不是客户端使用的。

```c
#include<sys/socket.h>

int listen(int sockfd, int backlog);
// 返回：若成功则为 0，若出错则为 -1。
```

listen 函数将 sockfd 从一个主动套接字转化为一个监听套接字（listening socket），该套接字可以接收来自客户端的连接请求。backlog 参数暗示了内核在开始拒绝连接请求之前，队列中要排队的未完成的连接请求的数量。backlog 参数的确切含义要求对 TCP/IP 协议的理解，这超出我们的讨论范围。通常我们会把它设置成一个较大的值，比如 1024。

### 6. accept 函数

服务器通过调用 accept 函数来等待来自客户端的连接请求。

```c
#include<sys/socket.h>

int accept(int listenfd, struct sockaddr *addr, int *addrlen);
// 返回：若成功则为非负连接描述符，若出错则为 -1。
```

accept 函数等待来自客户端的连接请求到达监听描述符 listenfd，然后在 addr 中填写客户端套接字地址，并返回一个已连接描述符，这个描述符可被用来利用 Unix I/O 函数与客户端通信。

监听描述符和已连接描述符之间的区别让人感到迷惑。监听描述是作为客户端连接请求的一个端点。它通常被创建一次，并存在于服务器的整个生命周期。已连接描述符是客户端和服务器之间已经建立起来了的连接的一个端点。服务器每次接受连接请求时都会创建一次，它只存在于服务器为一个客户端服务的过程中。

![image-20231214202552240](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214202552240.png)

> 为什么要区分监听描述符和已连接描述符？
>
> 区分这两者使得我们可以建立并发服务器。例如，每次一个连接请求到达监听描述符时，我们可以派生（fork）一个新的进程，它通过已连接描述符与客户端通信。

### 7. 主机和服务的转换

Linux 提供一些强大的函数，称为 getaddrinfo 和 getnameinfo 实现二进制套接字地址结构和主机名、主机地址、服务名和端口号的字符串表示之间的相互转化。

#### （1）getaddrinfo 函数

getaddrinfo 函数将主机名、主机地址和端口号的字符串表示转化为套接字地址结构。它是已弃用的 gethostbyname 和 getservbyname 的新的替代品。和以前的那些函数不同，这些函数是可重入的，适用于任何协议。

![image-20231214204654621](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214204654621.png)

给定 host 和 service（套接字地址的两个组成部分），getaddrinfo 返回 result，result 指向一个 addinfo 结构的链表，其中每个结构指向一个对应于 host 和 service 的套接字地址结构。

![image-20231214205132284](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214205132284.png)

在客户端调用了 addrinfo 之后，会遍历这个列表，依次尝试每个套接字地址，直到调用 socket 和 connect 成功，建立起连接。类似地，服务器会尝试遍历列表中的每个套接字地址，直到调用 socket 和 bind 成功，描述符会被绑定到一个合法的套接字地址。为了避免内存泄露，应用程序可以调用 gai_strerror，将该代码转换成消息字符串。

getaddrinfo 的 host 参数可以是域名，也可以是数字地址（如点分十进制 IP 地址）。service 参数可以是服务名（如 http），也可以是十进制端口号。如果不想把主机名转换成地址，可以把 host 设置为 NULL。对 service 来说也是一样的。但是必须指定两者中至少一个。

可选的参数 hints 是一个 addrinfo 结构，它提供对 getaddrinfo 返回的套接字地址列表的更好的控制。如果想要传递 hints 参数，只能设置下列字段：ai_family、ai_socktype、ai_protocol 和 ai_flags 字段。其他字段必须设置为 0（或 NULL）。实际中，我们用 memset 将整个结构清零，然后又选择的设置一些值：

- getaddrinfo 默认可以返回 IPv4 和 IPv6 的套接字地址。ai_family 设置为 AF_INET 会将列表限制为 IPv4 地址。
- 对于 host 关联的每个地址，getaddrinfo 最多返回三个 addrinfo 结构，每个 ai_socktype 字段不同：一个是连接、一个是数据报、一个是原始套接字。ai_socktype 设置为 SOCK_STREAM 将列表限制为对每个地址最多一个 addrinfo 结构，该结构的套接字地址可以作为连接的一个端点。
- ai_flags 字段是一个位掩码，可以进一步修改默认行为，可以把各种值用 OR 组合起来的到该掩码。下面是我们认为有用的值：
    - AI_ADDRCONFIG。如果在使用连接，就推荐使用者标志。它要求只有当本地主机和配置为 IPv4 时，getaddrinfo 返回 IPv4 地址。IPv6 也类似。
    - AI_CANONNMAE。ai_canonname 字段默认为 NULL。如果设置了改标志，就是告诉 getaddrinfo 将列表中第一个 addrinfo 结构的 ai_canonname 字段指向 host 权威的名字。
    - AI_NUMBERICSERV。参数 service 默认可以是服务名或端口号。这个标志强制 service 是端口号。
    - AI_PASSIVE。getaddrinfo 默认返回套接字地址，客户端可以在调用 connect 时用作主动套接字。这个标志告诉该函数，返回的套接字地址可能被服务器用作监听套接字。在这种情况下，参数 host 应该为 NULL。得到的套接字地址结构中的地址字段会是通配符地址，告诉内核这个服务器会接受发送到该主机的所有 IP 地址的请求。这是所有实例服务器所期望的行为。

![image-20231214215054121](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214215054121.png)

当 getaddrinfo 创建输出列表中的 addrinfo 结构时，会填写每个字段，除了 ai_flags。ai_addr 字段指向一个套接字地址结构，ai_addrlen 字段给出这个套接字地址结构的大小，而 ai_next 字段指向列表中的下一个 addrinfo 的结构。其他字段描述这个套接字地址的各种属性。

#### （2）getnameinfo 函数

getnameinfo 函数和 getaddrinfo 是相反的，将一个套接字地址结构转换成相应的主机和服务名字符串。

![image-20231214235139393](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231214235139393.png)

参数 sa 指向大小为 salen 字节的套接字地址，host 指向大小为 hostlen 字节的缓冲区，service 指向大小为 servlen 字节的缓冲区。getnameinfo 函数将套接字地址结构 sa 转换成对应的主机和服务名字符串，并将它们复制到 host 和 service 缓冲区。如果 getnameinfo 返回非零的错误代码，应用程序可以调用 gai_strerror 把它转化成字符串。

如果不想要主机名，可以把 host 设置成 NULL，hostlen 设置成 0，服务字段也一样。

参数 flags 是一个位掩码，能够修改默认行为。可以把各种值用  OR 组合起来得到该掩码，下面是两个有用的值：

- NI_NUMERICHOST。getnameinfo 默认试图返回 host 中的域名，设置改标志会使该函数返回一个数字地址字符串。
- NI_NUMERIVSERV。getnameinfo 默认会检查 /etc/services，如果可能，会返回服务名而不是端口号。设置该标志会使该函数跳过查找，简单的返回端口号。

下图给出一个简单的程序，称为 HOSTINFO，它使用 getaddrinfo 和 getnameinfo 展示出域名到和它相关联的 IP 地址之间的映射。类似于 NSLOOKUP 程序。

![image-20231215000118569](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231215000118569.png)

首先，初始化 hints 结构，使 getaddrinfo 返回我们想要的地址。在这里，我们想查找 32 位的 IP 地址，用作连接的端点。因为只想 getaddrinfo 转换域名，所以用 service 参数为 NULL 来调用它。

调用 getaddrinfo 之后，会遍历 addrinfo 结构，用 getnameinfo 将每个套接字地址转换为点分十进制地址字符串。遍历完列表以后，我们调用 freeaddrinfo 小心地释放这个列表。运行 HOSTINFO 时，我们看到 twitter.com 映射到四个 IP 地址，和 NSLOOKUP 的结果一样。

### 8. 套接字接口的辅助函数

初学时，getnameinfo 函数和套接字接口函数看上去有些可怕。用高级的辅助函数包装一下就方便很多，称为 open_clientfd 和 open_listenfd，客户端和服务器互相通信可以使用这些函数。

#### （1）open_clientfd 函数

客户端调用 open_clientfd 函数建立与服务器的连接。

```c
#include "csapp.h"

int open_clientfd(char *hostname, char *port);
// 返回：若成功则为描述符，若出错则为 -1。
```

open_clientfd 函数建立与服务器的连接，该服务器运行在主机 hostname 上，并在端口号 port 上监听连接请求。它返回一个打开的套接字描述符，该描述符准备好了，可以用 Unix I/O 函数做输入输出，下图给出 open_clientfd 代码。

我们调用 getaddrinfo，它返回 addrinfo 结构的列表，每个结构指向一个套接字地址结构，可用于建立与服务器的连接，该服务器运行在 hostname 上并监听 port 端口号。然后遍历该列表，依次尝试表中的每个条目，直到调用 socket 和 connect 成功。如果 connect 失败，在尝试下一个条目之前，要小心关闭套接字描述符。如果 connect 成功，我们就会释放列表内存，并把套接字描述符返回给客户端，客户端可以立即开始用 Unix I/O 与服务器通信了。

![image-20231215105408366](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231215105408366.png)

注意，所有代码都与任何版本的 IP 无关。socket 和 connect 参数都是用 getaddrinfo 自动产生的，使得代码干净可移植。

#### （2）open_listenfd 函数

调用 open_listenfd 函数，服务器创建一个监听描述符，准备好接收连接请求。

```c
#include "csapp.h"

int open_listenfd(char *port);
// 返回：若成功则为描述符，若出错则为 -1。
```

open_listenfd 函数打开和返回一个监听描述符，这个描述符准备好在端口 port 上接收连接请求。下图展示了具体代码。open_listenfd 风格与 open_clientfd 风格相似。调用 getaddrinfo 函数，然后遍历结果列表，直到调用 socket 和 bind 函数成功。注意，在第 20 行，我们使用 setsockopt 函数来配置服务器，使得服务器能够被终止、重启和立即开始接收连接请求。一个重启的服务器默认将在大约 30s 内拒绝客户端的连接请求，这严重地阻碍了调试。

因为我们的调用 getaddrinfo 时，使用了 AI_PASSIVE 标志并将 host 参数设置为 NULL，每个套接字地址结构中的地址字段会被设置为通配符地址，这告诉内核这个服务器会接收发送到本机所有 IP 地址的请求。

![image-20231215112610918](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231215112610918.png)

最后我们调用 listen 函数，将 listenfd 转化为一个监听描述符，并返回给调用者。如果 listen 失败，要避免内存泄漏，在返回前关闭描述符。

### 9. echo 客户端和服务器示例

学习套接字接口的最好方法是研究示例代码。下面展示了一个 echo 客户端的代码。在和服务器建立连接之后，客户端进入一个循环，反复从标准输入读取文本行，发送文本行给服务器，从服务器读取回送的行，并输出结果到标准输出。当 fgets 在标准输入遇到 EOF 时，或者因为用户在键盘上输入 Ctrl+D，或者因为在一个重定向的输入文件中用尽了所有的文本行时，循环就终止。

![image-20231215150429582](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231215150429582.png)

循环终止后，客户端关闭描述符。这会导致发送一个 EOF 通知到服务器，当服务器从它的 rio_readlineb 函数收到一个为 0 的返回码时，就会检测到这个结果。在关闭它的描述符后，客户端就终止了。既然客户端内核在一个进程终止时会自动关闭所有打开的描述符，第 24 行的 close 就没有必要了。不过，显式地关闭已经打开的任何描述符是一个良好的编程习惯。

下面展示了 echo 服务器的主程序。在打开监听描述符后，他进入一个无限循环。每次循环都等待一个来自客户端的连接请求，输出已连接客户端的域名和 IP 地址，并调用 echo 函数为这些客户端服务。在 echo 程序返回后，主程序关闭已连接描述符。一旦客户端和服务器关闭了它们各自的描述符，连接也就终止了。

第 9 行的 clientaddr 变量是一个套接字地址结构，被传递给 accept。在 accept 返回之前，会在 clientaddr 中填上另一端客户端的套接字地址。注意，我们将 clientaddr 声明为 struct sockaddr_storage 类型，而不是 struct sockaddr_in 类型。根据定义，sockaddr_storage 机构足够装下任何类型的套接字地址，以保证代码的协议无关性。

![image-20231215150752285](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231215150752285.png)

注意，简单的 echo 服务器一次只能处理一个客户端。这种类型的服务器一次一个地在客户端间迭代，称为迭代服务器。在下一章中，我们将学习并发服务器，它能够同时处理多个客户端。

下图是 echo 程序代码，该程序反复读写文本行，直到 rio_readlineb 函数在第 10 行遇到 EOF。

![image-20231215151008581](https://raw.githubusercontent.com/charming-c/image-host/master/img/image-20231215151008581.png)

> EOF 意味着什么？
>
> EOF 是由内核检测的一种条件。应用程序在它接收到一个由 read 函数返回的 0 返回码时，它就会发现出 EOF 条件。对于磁盘文件，当前文件位置超出文件长度时，会发生 EOF。对于因特网连接，当一个进程关闭连接它的一端时，就会发生 EOF。连接另一端的进程在试图读取流中的最后一个字节之后的字节时，会检测到 EOF。

