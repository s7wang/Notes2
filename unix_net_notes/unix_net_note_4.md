# 第二部分 基本套接字编程

# 第四章 基本TCP套接字编程

## 4.1 概述

本章讲解编写一个完整的TCP客户/服务器程序所需要的基本套接字函数。讲解完即将使用的所有基本套接字函数之后，我们就在下一章中开发这个客户/服务器程序。我们将围绕该客户/服务器程序展开本书，并多次对它加以改进。

我们还讲解并发服务器，它是在同时有大量的客户连接到同一服务器上时用于提供并发性的一种常用Unix技术。每个客户连接都迫使服务器为它派生（`fork`）一个新的进程。本章中我们只考虑使用`fork`实施的每客户单进程模型，然而当在第26章讨论线程时，我们将考虑称为每客户单线程的另外一种模型。

一对TCP客户与服务器进程之间发生的一些典型事件的时间表。服务器首先启动，稍后某个时刻客户启动，它试图连接到服务器。我们假设客户给服务器发送一个请求，服务器处理该请求，并且给客户发回一个响应。这个过程一直持续下去，直到客户关闭连接的客户端，从而给服务器发送一个`EOF`（文件结束）通知为止。服务器接着也关闭连接的服务器端，然后结束运行或者等待新的客户连接。



## 4.2 socket 函数

为了执行`网络I/O`，一个进程必须做的第一件事情就是调用`socket`函数，指定期望的通信协议类型（使用IPv4的TCP、使用IPv6的UDP、Unix域字节流协议等）。

~~~c
NAME
       socket - create an endpoint for communication

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int socket(int domain, int type, int protocol);
RETURN VALUE
       On  success, a file descriptor for the new socket is returned.  On error, 
       -1 is returned, and errno is set appropriately.
domain //-------
       Name                Purpose                         Man page
       AF_UNIX, AF_LOCAL   Local communication             unix(7)
       AF_INET             IPv4 Internet protocols         ip(7)
       AF_INET6            IPv6 Internet protocols         ipv6(7)
       AF_IPX              IPX - Novell protocols
       AF_NETLINK          Kernel user interface device    netlink(7)
       AF_X25              ITU-T X.25 / ISO-8208 protocol  x25(7)
       AF_AX25             Amateur radio AX.25 protocol
       AF_ATMPVC           Access to raw ATM PVCs
       AF_APPLETALK        AppleTalk                       ddp(7)
       AF_PACKET           Low level packet interface      packet(7)
       AF_ALG              Interface to kernel crypto API

type	//------------
    	//流式　——　有序的　可靠的　双工的　基于链接的　字节流的　
       SOCK_STREAM     Provides sequenced, reliable, two-way, 
					　　connection-based byte　streams. An out-of-band 
                       data transmission mechanism may be supported.
		//报式
       SOCK_DGRAM      Supports datagrams (connectionless, unreliable 
                       messages of a fixed maximum length).
		//有序分组式　————　安全可靠的报式传输
       SOCK_SEQPACKET  Provides  a sequenced, reliable, two-way 
                       connection-based data transmission path for 
                       datagrams of fixed maximum length; a consumer 
                       is required to read an  entire  packet with 
                       each input system call.
		//网络协议层的访问
       SOCK_RAW        Provides raw network protocol access.
		//数据层的访问
       SOCK_RDM        Provides a reliable datagram layer that does 
                       not guarantee ordering.
		//不要随便用这个
       SOCK_PACKET     Obsolete and should not be used in new 
                       programs; see packet(7).
~~~

其中domain参数指明协议族。该参数也往往被称为协议域。type 参数指明套接字类型。protocol参数应设为某个协议类型常值，或者设为0，以选择所给定domain和type组合的系统默认值。

~~~c
/* Linux 扩展 Flag（可“或”进 type 中） */
SOCK_NONBLOCK   // 创建即非阻塞
SOCK_CLOEXEC    // exec 时自动关闭 fd
socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

/* protocol（具体协议）有效值 */
// 最常用：写 0（让内核自动选择）
socket(AF_INET, SOCK_STREAM, 0);   // 自动选 TCP
socket(AF_INET, SOCK_DGRAM,  0);   // 自动选 UDP
~~~

| 协议宏         | 适用      | 说明         |
| -------------- | --------- | ------------ |
| `IPPROTO_TCP`  | TCP       | 显式指定 TCP |
| `IPPROTO_UDP`  | UDP       | 显式指定 UDP |
| `IPPROTO_ICMP` | ping      | 仅 RAW       |
| `IPPROTO_RAW`  | 原始 IP   | 需 root      |
| `IPPROTO_IPV6` | IPv6 原始 |              |

并非所有套接字domain与type的组合都是有效的，一些有效的组合和对应的真正协议。其中标为“是”的项也是有效的，但还没有找到便捷的缩略词。而空白项则是无效组合。

~~~(空)
Server (监听端)                               Network                               Client (发起端)
-----------------------------------------------------------------------------------
+--------------------+         <IP:PORT 1.2.3.4:9000>         +-----------------+
| int listen_fd =    |  socket()  ---------------->  socket() | int sock =      |
| socket(AF_INET,...)|  (创建监听套接字)                         |  (创建套接字)    |
+--------------------+                                        +-----------------+
        |
        | bind(listen_fd, addr(0.0.0.0:9000))
        | (绑定到本地地址与端口)
        v
+----------------------+
| listen(listen_fd,    |
|        backlog)      |
| (进入监听状态)         |
+----------------------+
        |                   <SYN, SYN-ACK, ACK>  三次握手完成
        | accept(listen_fd, &peer)
        | (阻塞等待 -> 返回 conn_fd)
        v
+-------------------+             <connection>             +-------------------+
| int conn_fd =     | <=========== 数据通道 ==============> | connected socket  |
| accept(...)       |   recv(conn_fd)/send(conn_fd)        | (connect 成功)    |
| (返回已连接套接字)   |   send()/recv() 或 read()/write()    +-------------------+
+-------------------+             或 write()/read()                 |
        |                                                           |
        | 在并发服务中：fork()/pthread_create()                        |
        | 或 epoll/select 处理多个 conn_fd                            |
        | close(conn_fd)   (会话结束)                                 |
        v                                                            v
+-------------------+                                         close(sock)
| close(listen_fd)  |                                          (断开连接)
+-------------------+


~~~

* 非法组合（必然失败）

| 错误组合                              | 原因              |
| ------------------------------------- | ----------------- |
| `AF_INET + SOCK_STREAM + IPPROTO_UDP` | STREAM 只能配 TCP |
| `AF_INET + SOCK_DGRAM + IPPROTO_TCP`  | DGRAM 不能配 TCP  |
| `AF_UNIX + IPPROTO_TCP`               | UNIX 不走 IP 协议 |
| `SOCK_SEQPACKET + IPPROTO_UDP`        | 语义冲突          |
| `AF_PACKET + SOCK_STREAM`             | 链路层无流概念    |

* 参数选择逻辑

~~~swift
AF_INET
   ↓
SOCK_STREAM   →  IPPROTO_TCP
SOCK_DGRAM    →  IPPROTO_UDP
SOCK_RAW      →  IPPROTO_ICMP / IPPROTO_TCP / 自定义

AF_UNIX
   ↓
SOCK_STREAM   →  protocol 必须为 0
SOCK_DGRAM    →  protocol 必须为 0

~~~



## 4.3 connect 函数

TCP客户用connect函数来建立与TCP服务器的连接。

~~~c
NAME
       connect - initiate a connection on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
~~~

sockfd是由socket函数返回的套接字描述符，第二个、第三个参数分别是一个指向套接字地址结构的指针和该结构的大小，如3.3节所述。套接字地址结构必须含有服务器的IP地址和端口号。

客户在调用函数connect前不必非得调用bind函数（我们在下一节介绍该函数)，因为如果需要的话，内核会确定源IP地址，并选择一个临时端口作为源端口。如果是TCP套接字，调用connect函数将激发TCP的三路握手过程（2.6节），而且仅在连接建立成功或出错时才返回，其中出错返回可能有以下几种情况。

> （1）若TCP客户没有收到`SYN`分节的响应，则返回`ETIMEDOUT`错误。举例来说，调用`connect`函数时，4.4 BSD内核发送一个`SYN`，若无响应则等待 6s 后再发送一个，若仍无响应则等待 24s 后再发送一个（TCPv2第828页）。若总共等了 75s 后仍未收到响应则返回本错误。有些系统提供对超时值的管理性控制，见TCPv1的附录E。
>
> （2）若对客户的`SYN`的响应是`RST`（表示复位)，则表明该服务器主机在我们指定的端口上没有进程在等待与之连接（例如服务器进程也许没在运行）。这是一种硬错误（`hard error`)，客户一接收到`RST`就马上返回`ECONNREFUSED`错误。`RST`是TCP在发生错误时发送的一种TCP分节。产生`RST`的三个条件是：目的地为某端口的`SYN`到达，然而该端口上没有正在监听的服务器（如前所述）；TCP想取消一个已有连接；TCP接收到一个根本不存在的连接上的分节。（TCPv1第246~250页有更详细的信息。）
>
> （3）若客户发出的`SYN`在中间的某个路由器上引发了一个“destination unreachable”（目的地不可达）`ICMP`错误，则认为是一种软错误（`soft error`)。客户主机内核保存该消息，并按第一种情况中所述的时间间隔继续发送`SYN`。若在某个规定的时间（4.4 BSD规定 75s）后仍未收到响应，则把保存的消息（即`ICMP`错误）作为`EHOSTUNREACH`或`ENETUNREACH`错误返回给进程。以下两种情形也是有可能的：一是按照本地系统的转发表，根本没有到达远程系统的路径；二是`connect`调用根本不等待就返回。

按照TCP状态转换图`connect`函数导致当前套接字从`CLOSED`状态（该套接字自从由`socket`函数创建以来一直所处的状态）转移到`SYN_SENT`状态，若成功则再转移到`ESTABLISHED`状态。若`connect`失败则该套接字不再可用，必须关闭，我们不能对这样的套接字再次调用`connect`函数。当循环调用函数`connect`为给定主机尝试各个IP地址直到有一个成功时，在每次`connect`失败后，都必须`close`当前的套接字描述符并重新调用`socket`。



## 4.4 bind 函数

`bind`函数把一个本地协议地址赋予一个套接字。对于网际网协议，协议地址是32位的`IPv4`地址或128位的`IPv6`地址与16位的TCP或UDP端口号的组合。

~~~c
NAME
       bind - bind a name to a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);


NAME
       inet_pton - convert IPv4 and IPv6 addresses from text to binary form

SYNOPSIS
       #include <arpa/inet.h>
			//将一个ip地址转换成二进制的数
    		/*
    		af	协议族 AF_INET / AF_INET6 / AF_UNIX
    		src	ip地址　
    		dst	结果保存位置　
    		*/
       int inet_pton(int af, const char *src, void *dst);
RETURN VALUE
       On success, zero is returned.  On error, -1 is returned, 
       and errno is set appropriately.
~~~

第二个参数是一个指向特定于协议的地址结构的指针，第三个参数是该地址结构的长度。对于TCP，调用`bind`函数可以指定一个端口号，或指定一个IP地址，也可以两者都指定，还可以都不指定。

* 服务器在启动时捆绑它们的众所周知端口。如果一个TCP客户或服务器未曾调用`bind`捆绑一个端口，当调用`connect`或`listen`时，内核就要为相应的套接字选择一个临时端口。让内核来选择临时端口对于TCP客户来说是正常的，除非应用需要一个预留端口；然而对于TCP服务器来说却极为罕见，因为服务器是通过它们的众所周知端口被大家认识的。

  > 这个规则的例外是远程过程调用（Remote Procedure Call，RPC）服务器。它们通常就由内核为它们的监听套接字选择一个临时端口，而该端口随后通过RPC端口映射器进行注册。客户在`connect`这些服务器之前，必须与端口映射器联系以获取它们的临时端口。这种情况也适用于使用UDP的RPC服务器。

* 进程可以把一个特定的IP地址捆绑到它的套接字上，不过这个IP地址必须属于其所在主机的网络接口之一。对于TCP客户，这就为在该套接字上发送的IP数据报指派了源IP地址。对于TCP服务器，这就限定该套接字只接收那些目的地为这个IP地址的客户连接。TCP客户通常不把IP地址捆绑到它的套接字上。当连接套接字时，内核将根据所用外出网络接口来选择源IP地址，而所用外出接口则取决于到达服务器所需的路径（TCPv2第737页）。如果TCP服务器没有把IP地址捆绑到它的套接字上，内核就把客户发送的`SYN`的目的IP地址作为服务器的源IP地址（TCPv2第943页）。

正如我们所说，调用`bind`可以指定IP地址或端口，可以两者都指定，也可以都不指定。如果指定端口号为0，那么内核就在`bind`被调用时选择一个临时端口。然而如果指定IP地址为通配地址，那么内核将等到套接字已连接（TCP）或已在套接字上发出数据报（UDP）时才选择一个本地IP地址。

对于IPv4来说，通配地址由常值`INADDR_ANY`来指定，其值一般为0。它告知内核去选择IP地址。

~~~c
struct sockaddr_in  servaddr;
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);  /* wildcard */
~~~

如此赋值对IPv4是可行的，因为其IP地址是一个32位的值，可以用一个简单的数字常值表示（本例中为0)，对于IPv6，我们就不能这么做了，因为128位的IPv6地址是存放在一个结构中的。（在C语言中，赋值语句的右边无法表示常值结构。）为了解决这个问题，我们改写为：

~~~c
struct sockaddr_in6  serv;
serv.sin6_addr = in6addr_any;  	/* wildcard */
//----------------------------------------------------------------------------
/* IPv6 address */
struct in6_addr
  {
    union
      {
	uint8_t	__u6_addr8[16];
	uint16_t __u6_addr16[8];
	uint32_t __u6_addr32[4];
      } __in6_u;
#define s6_addr			__in6_u.__u6_addr8
#ifdef __USE_MISC
# define s6_addr16		__in6_u.__u6_addr16
# define s6_addr32		__in6_u.__u6_addr32
#endif
  };
#endif /* !__USE_KERNEL_IPV6_DEFS */

extern const struct in6_addr in6addr_any;        /* :: */
extern const struct in6_addr in6addr_loopback;   /* ::1 */
#define IN6ADDR_ANY_INIT { { { 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 } } }
#define IN6ADDR_LOOPBACK_INIT { { { 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1 } } }
~~~

系统预先分配`in6addr_any`变量并将其初始化为常值`IN6ADDR_ANY_INIT`。头文件`<netinet/in.h>`中含有`in6addr_any`的`extern`声明。无论是网络字节序还是主机字节序，`INADDR_ANY`的值（为0）都一样，因此使用`htonl`并非必需。不过既然头文件`<netinet/in.h>`中定义的所有`INADDR_`常值都是按照主机字节序定义的，我们应该对任何这些常值都使用`htonl`。

如果让内核来为套接字选择一个临时端口号，那么必须注意，函数`bind`并不返回所选择的值。实际上，由于`bind`函数的第二个参数有`const`限定词，它无法返回所选之值。为了得到内核所选择的这个临时端口值，必须调用函数`getsockname`来返回协议地址。

进程捆绑非通配IP地址到套接字上的常见例子是在为多个组织提供Web服务器的主机上(TCPv3的14.2节)。首先，每个组织都得有各自的域名，譬如这样的形式：www.organization.com。其次，每个组织的域名都映射到不同的IP地址，不过通常仍在同一个子网上。举例来说，如果子网是`198.69.10`，那么第一个组织的IP地址可以是`198.69.10.128`，第二个组织的可以是`198.69.10.129`，等等。然后，把所有这些IP地址都定义成单个网络接口的别名（警如在4.4 BSD系统上就使用`ifconfig`命令的`alias`选项来定义)，这么一来，IP层将接收所有目的地为任何一个别名地址的外来数据报。最后，为每个组织启动一个HTTP服务器的副本，每个副本仅仅捆绑相应组织的IP地址。

从bind函数返回的一个常见错误是`EADDRINUSE`（“Address alreadyin use”，地址已使用）。到7.5节讨论`SO_REUSEADDR`和`SO_REUSEPORT`这两个套接字选项时我们再详细说明。



## 4.5 listen 函数

listen函数仅由TCP服务器调用，它做两件事情。

（1）当`socket`函数创建一个套接字时，它被假设为一个主动套接字，也就是说，它是一个将调用`connect`发起连接的客户套接字。`listen`函数把一个未连接的套接字转换成一个被动套接字，指示内核应接受指向该套接字的连接请求。根据TCP状态转换图，调用`listen`导致套接字从`CLOSED`状态转换到`LISTEN`状态。

（2）本函数的第二个参数规定了内核应该为相应套接字排队的最大连接个数。

~~~c
NAME
       listen - listen for connections on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int listen(int sockfd, int backlog);
RETURN VALUE
       On success, zero is returned.  On error, -1 is returned, 
       and errno is set appropriately.
~~~

本函数通常应该在调用`socket`和`bind`这两个函数之后，并在调用`accept`函数之前调用。为了理解其中的`backlog`参数，我们必须认识到内核为任何一个给定的监听套接字维护两个队列：

（1）未完成连接队列（incomplete connection queue)，每个这样的`SYN`分节对应其中一项：已由某个客户发出并到达服务器，而服务器正在等待完成相应的TCP三路握手过程。这些套接字处于`SYN_RCVD`状态。

（2）已完成连接队列（completed connection queue)，每个已完成TCP三路握手过程的客户对应其中一项。这些套接字处于`ESTABLISHED`状态。

> 未完成连接队列 和 已完成连接队列 （处于`SYN_RCVD`状态和处于`ESTABLISHED`状态）的所有SYN分节不超过`backlog`的大小。

每当在未完成连接队列中创建一项时，来自监听套接字的参数就复制到即将建立的连接中。连接的创建机制是完全自动的，无需服务器进程插手。

当来自客户的SYN到达时，TCP在未完成连接队列中创建一个新项，然后响应以三路握手的第二个分节：服务器的`SYN`响应，其中捎带对客户`SYN`的`ACK`。这一项一直保留在未完成连接队列中，直到三路握手的第三个分节（客户对服务器`SYN`的`ACK`）到达或者该项超时为止。（源自Berkeley的实现为这些未完成连接的项设置的超时值为75s。）如果三路握手正常完成，该项就从未完成连接队列移到已完成连接队列的队尾。当进程调用`accept`时（该函数在下一节讲解)，已完成连接队列中的队头项将返回给进程，或者如果该队列为空，那么进程将被投入睡眠，直到TCP在该队列中放入一项才唤醒它。

关于这两个队列的处理，以下几点需要考虑。

* listen函数的backlog参数曾被规定为这两个队列总和的最大值。

  > `backlog`的含义从未有过正式的定义。4.2 BSD的手册页面宣称它定义的是：“the maximum length the queue ofpending connections may grow to”（由未处理连接构成的队列可能增长到的最大长度）。许多手册页面甚至POSIX规范也逐字复制该定义，然而该定义并未解释未处理连接是处于`SYN_RCVD`状态的连接，还是尚未由进程接受的处于`ESTABLISHED`状态的连接，亦或两者皆可。这个历史性的定义出自追溯到4.2 BSD版本的Berkeley的实现，后来被许多其他实现复制。

* 源自Berkeley的实现给`backlog`增设了一个模糊因子（`fudgefactor`)：把它乘以`1.5`得到未处理队列最大长度（TCPv1第257页和TCPv2第462页)。举例来说，通常指定为5的`backlog`值实际上允许最多有8项在排队。

  > 增设该模糊因子的理由已无可考证 [Joy 1994]，但是如果我们把`backlog`看成是内核能为某套接字排队的最大已完成连接数目（[Boman 1997c]，稍后讨论），那么增加模糊因子的理由就是把队列中的未完成连接也计算在内。

* 不要把`backlog`定义为0，因为不同的实现对此有不同的解释。如果你不想让任何客户连接到你的监听套接字上，那就关掉该监听套接字。
* 在三路握手正常完成的前提下（也就是说没有丢失分节，从而没有重传），未完成连接队列中的任何一项在其中的存留时间就是一个`RTT`，而`RTT`的值取决于特定的客户与服务器。TCPv3的14.4节指出，对于一个Web服务器，许多客户与单个服务器之间的中值`RTT`为`187ms`。（既然出现一些大值可能显著扭曲均值，对于该统计量通常使用中值。）

* 历来沿用的样例代码总是给出值为`5`的`backlog`，因为这是4.2 BSD支持的最大值。这个值在20世纪80年代是足够的，当时繁忙的服务器一天也就处理几百个连接。然而随着万维网（WorldWideWeb，WWW）的发展，繁忙的服务器一天要处理几百万个连接，这个偏小的值就根本不够了（TCPv3第187~192页）。繁忙的HTTP服务器必须指定一个大得多的`backlog`值，而且较新的内核必须支持较大的`backlog`值。
  
  > 当前的许多系统允许管理员修改`backlog`的最大值。

* 问题是既然`backlog`值为`5`往往不够，那么应用进程应该指定多大值的`backlog`呢？这个问题不好回答。当今的HTTP服务器指定了一个较大的值，但是如果这个指定值在源代码中是一个常值，那么增长其大小需要重新编译服务器程序。另一个方法是设定一个默认值，不过允许通过命令行选项或环境变量覆写该默认值。指定一个比内核能够支持的值还要大的backlog也是可接受的，因为内核应该悄然把所指定的偏大值截成自身支持的最大值，而不返回错误（TCPv2第456页)。我们通过修改`listen`函数的包裹函数就能够提供解决本问题的一个简单办法。下面给出了实际的代码。我们允许环境变量`LISTENQ`覆写由调用者指定的值。

~~~c
/* include Listen */
void
Listen(int fd, int backlog)
{
	char	*ptr;

		/*4can override 2nd argument with environment variable */
	if ( (ptr = getenv("LISTENQ")) != NULL)
		backlog = atoi(ptr);

	if (listen(fd, backlog) < 0)
		err_sys("listen error");
}
/* end Listen */
~~~

* 注意：实际上在Linux中并没有`LISTENQ`这个环境变量，实际上`listen() `第二个参数（`backlog`）会被限制为一个最大值，通过`/proc/sys/net/core/somaxconn`，如果你写`listen(sockfd, 1024);`，其实Linux会取`min(1024, somaxconn);`， TCP `SYN`队列（半连接）有自己的队列长度`/proc/sys/net/ipv4/tcp_max_syn_backlog`。

~~~(空)
cat /proc/sys/net/core/somaxconn  			----> 4096
cat /proc/sys/net/ipv4/tcp_max_syn_backlog 	----> 2048
~~~

* 手册和书本历来声称：将固定数目的未处理连接排成队列是为了处理服务器进程在相继的`accept`调用之间处于忙状态的情况。这就隐含着如此意义：在这两个队列中，已完成队列通常应该比未完成队列有更多的项。繁忙的Web服务器再次表明这是不对的。指定较大`backlog`值的理由在于：随着客户`SYN`分节的到达，未完成连接队列中的项数可能增长，它们等着三路握手的完成。

* 当一个客户`SYN`到达时，若这些队列是满的，TCP就忽略该分节（TCPv2第930~931页），也就是不发送`RST`。这么做是因为：这种情况是暂时的，客户TCP将重发`SYN`，期望不久就能在这些队列中找到可用空间。要是服务器TCP立即响应以一个`RST`，客户的`connect`调用就会立即返回一个错误，强制应用进程处理这种情况，而不是让TCP的正常重传机制来处理。另外，客户无法区别响应`SYN`的`RST`究竟意味着“该端口没有服务器在监听”，还是意味着“该端口有服务器在监听，不过它的队列满了”。

  > 有些实现在这些队列满时确实发送`RST`。由于上述原因，这种做法是不正确的，我们最好忽略其存在的可能性，除非客户明确要求与这样的服务器交互。处理这种情况的额外代码编写会降低客户程序的健壮性，在正常的`RST`情况下（即确实没有服务器在客户请求的端口上监听），也增加了网络的负荷。

* 在三路握手完成之后，但在服务器调用`accept`之前到达的数据应由服务器TCP排队，最大数据量为相应已连接套接字的接收缓冲区大小。



## 4.6 accept 函数

`accept`函数由TCP服务器调用，用于从已完成连接队列队头返回下一个已完成连接。如果已完成连接队列为空，那么进程被投入睡眠（假定套接字为默认的阻塞方式)。

~~~c
NAME
       accept, accept4 - accept a connection on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

       #define _GNU_SOURCE             /* See feature_test_macros(7) */
       #include <sys/socket.h>

       int accept4(int sockfd, struct sockaddr *addr,
                   socklen_t *addrlen, int flags);
RETURN VALUE
       On success, these system calls return a file descriptor for the 
       accepted socket (a  nonnegative  integer).  On error, -1 is returned, 
       errno is set appropriately, and addrlen is left unchanged.
~~~

参数`struct sockaddr *addr`和`socklen_t *addrlen`用来返回已连接的对端进程（客户）的协议地址。`addrlen`是值一结果参数：调用前，我们将由*addrlen所引用的整数值置为由`addr`所指的套接字地址结构的长度，返回时，该整数值即为由内核存放在该套接字地址结构内的确切字节数。

如果`accept`成功，那么其返回值是由内核自动生成的一个全新描述符，代表与所返回客户的TCP连接。在讨论`accept`函数时，我们称它的第一个参数为监听套接字（listening socket）描述符（由`socket`创建，随后用作`bind`和`listen`的第一个参数的描述符)，称它的返回值为已连接套接字（connected socket）描述符。区分这两个套接字非常重要。一个服务器通常仅仅创建一个监听套接字，它在该服务器的生命期内一直存在。内核为每个由服务器进程接受的客户连接创建一个已连接套接字（也就是说对于它的TCP三路握手过程已经完成)。当服务器完成对某个给定客户的服务时，相应的已连接套接字就被关闭。

本函数最多返回三个值：一个既可能是新套接字描述符也可能是出错指示的整数、客户进程的协议地址（由`addr`指针所指）以及该地址的大小（由`addrlen`指针所指)。如果我们对返回客户协议地址不感兴趣，那么可以把`addr`和`addrlen`均置为空指针。

已连接套接字每次都在循环中关闭，但监听套接字在服务器的整个有效期内都保持开放。我们还看到`accept`的第二和第三个参数都是空指针，因为我们对客户的身份不感兴趣。

### accept 说明

`accept()` 用于从服务器监听 `socket`（通常是 `listen()` 之后的 `sockfd`）中**取出一个已建立连接**，并生成一个新的“已连接 socket”用于通信。监听 socket 仍然保持监听状态。

* 参数说明

| 参数      | 说明                                                        |
| --------- | ----------------------------------------------------------- |
| `sockfd`  | 监听 socket 的文件描述符（必须先调用 `listen()`）           |
| `addr`    | 用于返回客户端地址信息（IP + 端口），可为 NULL              |
| `addrlen` | 输入输出参数，传入结构体长度，返回实际地址长度；可以为 NULL |

* 返回值

| 返回值  | 含义                                           |
| ------- | ---------------------------------------------- |
| **>=0** | 成功，返回一个新的 socket fd，用于与客户端通信 |
| **-1**  | 失败，errno 指示错误                           |

* 常见 errno

| errno                  | 含义                       |
| ---------------------- | -------------------------- |
| `EAGAIN`/`EWOULDBLOCK` | 非阻塞 socket 且无连接到达 |
| `ECONNABORTED`         | 客户端在三次握手后又断开   |
| `EMFILE`               | 进程文件描述符用完         |
| `ENFILE`               | 系统文件描述符用完         |

### accept4 说明

`accept4()` —— 带 flags 的增强版 accept，多了一个 `flags` 参数。

* 常见 flags：

| flag            | 作用                                          |
| --------------- | --------------------------------------------- |
| `SOCK_NONBLOCK` | 创建的新 socket 为非阻塞                      |
| `SOCK_CLOEXEC`  | 自动设置 FD_CLOEXEC 标志（避免子进程继承 fd） |

`accept4()` 的好处

* 一次系统调用完成设置，不需要：

```
fcntl(fd, F_SETFL, O_NONBLOCK);
fcntl(fd, F_SETFD, FD_CLOEXEC);
```

* 更高效
* 避免 race condition（竞态）



## 4.7 fork 和 exec 函数

### 4.7.1 fork 函数

`fork` 函数（包括有些系统可能提供的它的各种变体）是Unix中派生新进程的唯一方法。

~~~c

NAME
       fork - create a child process

SYNOPSIS
       #include <sys/types.h>
       #include <unistd.h>

       pid_t fork(void);
RETURN VALUE
       On  success, the PID of the child process is returned in the
       parent, and 0 is returned in the child.  On failure,  -1  is
       returned in the parent, no child process is created, and er‐
       rno is set appropriately.
~~~

如果你以前从未接触过该函数，那么理解`fork`最困难之处在于调用它一次，它却返回两次。它在调用进程（称为父进程）中返回一次，返回值是新派生进程（称为子进程）的进程ID号；在子进程又返回一次，返回值为0。因此，返回值本身告知当前进程是子进程还是父进程。

`fork`在子进程返回0而在父进程中返回子进程ID的原因在于：任何子进程只有一个父进程，而且子进程总是可以通过调用`getppid`取得父进程的进程ID。相反，父进程可以有许多子进程，而且无法获取各个子进程的进程ID。如果父进程想要跟踪所有子进程的进程ID，那么它必须记录每次调用`fork`的返回值。

父进程中调用`fork`之前打开的所有描述符在`fork`返回之后由子进程分享。我们将看到网络服务器利用了这个特性：父进程调用`accept`之后调用`fork`。所接受的已连接套接字随后就在父进程与子进程之间共享。**通常情况下，子进程接着读写这个已连接套接字，父进程则关闭这个已连接套接字**。

`fork`有两个典型用方法：

> （1）一个进程创建一个自身的副本，这样每个副本都可以在另一个副本执行其他任务的同时处理各自的某个操作。这是网络服务器的典型用法。我们将在本书后面看到许多这样的例子。
>
> （2）一个进程想要执行另一个程序。既然创建新进程的唯一办法是调用fork，该进程于是首先调用fork创建一个自身的副本，然后其中一个副本（通常为子进程）调用exec（接下去介绍）把自身替换成新的程序。这是诸如shell之类程序的典型用法.



### 4.7.2 exec 函数

存放在硬盘上的可执行程序文件能够被Unix执行的唯一方法是：由一个现有进程调用六个`exec`函数中的某一个。（当这6个函数中是哪一个被调用并不重要时，我们往往把它们统称为`exec`函数。）`exec`把当前进程映像替换成新的程序文件，而且该新程序通常从`main`函数开始执行。进程ID并不改变。我们称调用`exec`的进程为调用进程（calling process)，称新执行的程序为新程序（new program)。

这6个`exec`函数之间的区别在于：（a）待执行的程序文件是由文件名（filename）还是由路径名（pathname）指定；（b）新程序的参数是一一列出还是由一个指针数组来引用；（c）把调用进程的环境传递给新程序还是给新程序指定新的环境。

~~~c
NAME
       execl, execlp, execle, execv, execvp, execvpe - execute a file

SYNOPSIS
       #include <unistd.h>

       extern char **environ;

       int execl(const char *pathname, const char *arg, ...
                       /* (char  *) NULL */);
       int execlp(const char *file, const char *arg, ...
                       /* (char  *) NULL */);
       int execle(const char *pathname, const char *arg, ...
                       /*, (char *) NULL, char *const envp[] */);
       int execv(const char *pathname, char *const argv[]);
       int execvp(const char *file, char *const argv[]);
       int execvpe(const char *file, char *const argv[],
                       char *const envp[]);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):

       execvpe(): _GNU_SOURCE
RETURN VALUE
       The  exec()  functions return only if an error has occurred.  The re‐
       turn value is -1, and errno is set to indicate the error.
~~~

下是 **exec 系列函数（execl / execlp / execle / execv / execvp / execvpe）的作用、使用方法、区别、示例。** 

#### 1 exec 簇核心概念

所有 exec 函数做的事 **都是一样的**：

> **用另一个程序替换当前进程映像**。
>  调用成功后不会返回，失败才返回 -1。

区别只有两种：

**1）传参方式不同**

- **l 结尾 = list（用可变参数传参）**
- **v 结尾 = vector（用 char\* 数组传参）**

**2）是否搜索 PATH**

- 有 **p** 的版本会按 PATH 搜索可执行文件
- 没 p 的版本必须给出文件绝对路径

**3）是否自定义环境变量 envp**

- **e 结尾的版本允许自定义 envp**
- 没 e 的版本继承当前进程环境变量

####  2 exec 簇函数差异一览表

| 函数        | 路径是否必须绝对路径 | 参数形式    | 能否自定义 envp | 是否搜索 PATH |
| ----------- | -------------------- | ----------- | --------------- | ------------- |
| **execl**   | 必须是 pathname      | l：可变参数 | ❌               | ❌             |
| **execlp**  | file（可搜索 PATH）  | l：可变参数 | ❌               | ✔️             |
| **execle**  | 必须 pathname        | l：可变参数 | ✔️               | ❌             |
| **execv**   | 必须 pathname        | v：argv[]   | ❌               | ❌             |
| **execvp**  | file（可搜索 PATH）  | v：argv[]   | ❌               | ✔️             |
| **execvpe** | file（可搜索 PATH）  | v：argv[]   | ✔️               | ✔️             |

#### 3 说明 & 示例

* execl(path, arg0, arg1, ..., NULL)

> **l 表示 list：逐个参数传入**
>
> **不搜索 PATH，必须使用绝对路径**

~~~c
execl("/bin/ls", "ls", "-l", "/home", NULL);
~~~

---------------------

* execlp(file, arg0, arg1, ..., NULL)

> **p：根据 PATH 搜索可执行程序**
>
> **不需要绝对路径**

~~~c
execlp("ls", "ls", "-l", NULL);
~~~

---------------------

* execle(path, arg0, ..., NULL, envp)

> 和 execl 一样以 list 传参，但最后多一个环境变量数组：
>
> **可自定义环境变量 envp**

~~~c
char *envp[] = {"PATH=/usr/bin", "MYVAR=123", NULL};
execle("/usr/bin/env", "env", NULL, envp);
~~~

---------------------

* execv(path, argv[])

> **v：参数通过 char\* 数组传递**
>
> 必须用绝对路径

~~~c
char *argv[] = {"ls", "-l", NULL};
execv("/bin/ls", argv);
~~~

---------------------

* execvp(file, argv[])

> **v + p = 用数组传参 + 搜索 PATH**

~~~c
char *argv[] = {"ls", "-l", NULL};
execvp("ls", argv);
~~~

---------------------

* execvpe(file, argv[], envp)

> **v + p + e = 搜索 PATH + 数组传参 + 自定义 envp**
>
> 最强版本

~~~c
char *argv[] = {"env", NULL};
char *envp[] = {"MYVAR=hello", NULL};
execvpe("env", argv, envp);
~~~

---------------------

常见用法：

~~~c
char *argv[] = {"ls", "-l", "/tmp", NULL};
execvp("ls", argv);
perror("execvp failed");
exit(1);
~~~

进程在调用`exec`之前打开着的描述符通常跨`exec`继续保持打开。我们使用限定词“通常”是因为本默认行为可以使用`fcntl`设置`FD_CLOEXEC`描述符标志禁止掉。inetd服务器就利用了这个特性，我们将在13.5节讲述这一点。



## 4.8 并发服务器





~~~c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <sys/wait.h>

void sigchld_handler(int sig) {
    while (waitpid(-1, NULL, WNOHANG) > 0);  // 回收僵尸进程
}

void handle_client(int connfd) {
    char buf[1024];
    int n;

    while ((n = read(connfd, buf, sizeof(buf))) > 0) {
        write(connfd, buf, n);   // 回显
    }

    close(connfd);
    exit(0); // 子进程退出
}

int main() {
    int listenfd, connfd;
    struct sockaddr_in servaddr, cliaddr;
    socklen_t clilen;

    signal(SIGCHLD, sigchld_handler);  // 处理僵尸进程

    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(8888);

    bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr));
    listen(listenfd, 10);

    printf("Server running on port 8888...\n");

    while (1) {
        clilen = sizeof(cliaddr);
        connfd = accept(listenfd, (struct sockaddr*)&cliaddr, &clilen);

        pid_t pid = fork();
        if (pid == 0) {
            close(listenfd);   // 子进程不用监听socket
            handle_client(connfd);
        } else {
            close(connfd);     // 父进程关闭连接，继续 accept
        }
    }
}

~~~

重点在accept后的处理逻辑，执行fork后，如果是父进程直接关闭connfd然后等待或处理新到来的connect，子进程处理连接的任务（这里做了回显处理，即将client发送的数据原封不动地发送回去）。

> 验证方法：编译后运行程序，看到“Server running on port 8888...”为启动成功。使用 `sudo ss -lntp | grep 8888`看到如下类似输出，即为端口监听成功。
>
> ~~~(空)
> LISTEN 0      10      172.24.71.53:8888       0.0.0.0:*    users:(("test",pid=43927,fd=3))
> ~~~
>
> 使用新的终端或可连接到此IP的终端，使用 `telnet <IP> 8888` 连接后，键入字符即可验证。这里有两种情况不同的telnet可能时使用不同的字符处理模式，如果是**行缓冲模式（line buffered）**你输入一整行（按下 Enter），一次性发送给服务器，所以是每输入一行回显一次；但某些 telnet 会进入 **字符模式（character mode）**你每打一个字符 telnet 都会立刻发送给服务器，就会变成每输入一个字符就回显一个。所以两种情况都能表示server运行正常。

我们在2.6节说过，对一个TCP套接字调用`close`会导致发送一个`FIN`，随后是正常的TCP连接终止序列。为什中父进程对`connfd`调用`close`没有终止它与客户的连接呢？为了便于理解，我们必须知道每个文件或套接字都有一个引用计数。引用计数在文件表项中维护（APUE第58~59页)，它是当前打开着的引用该文件或套接字的描述符的个数。`socket`返回后与`listenfd`关联的文件表项的引用计数值为1。`accept`返回后与`connfd`关联的文件表项的引用计数值也为1。然而fork返回后，这两个描述符就在父进程与子进程间共享（也就是被复制)，因此与这两个套接字相关联的文件表项各自的访问计数值均为2。这么一来，当父进程关闭`connfd`时，它只是把相应的引用计数值从2减为1。该套接字真正的清理和资源释放要等到其引用计数值到达0时才发生。这会在稍后子进程也关闭`connfd`时发生。



## 4.9 close 函数

通常Unix `close`函数也用来关闭套接字，并终止TCP连接。

~~~c
SYNOPSIS
       #include <unistd.h>

       int close(int fd);

RETURN VALUE
       close() returns zero on success.  On error, -1 is returned, and errno is set
       appropriately.
~~~

`close`一个TCP套接字的默认行为是把该套接字标记成已关闭，然后立即返回到调用进程。该套接字描述符不能再由调用进程使用，也就是说它不能再作为`read`或`write`的第一个参数。然而TCP将尝试发送已排队等待发送到对端的任何数据，发送完毕后发生的是正常的TCP连接终止序列（2.6节）。

我们将在7.5节介绍的`SO_LINGER`套接字选项可以用来改变TCP套接字的这种默认行为。我介绍TCP应用进程必须怎么做才能确信对端应用进程已收到所有未处理数据。

### 描述符引用计数

在4.8节末尾我们提到过，并发服务器中父进程关闭已连接套接字只是导致相应描述符的引用计数值减1。既然引用计数值仍大于0，这个`close`调用并不引发TCP的四分组连接终止序列。对于父进程与子进程共享已连接套接字的并发服务器来说，这正是所期望的。如果我们确实想在某个TCP连接上发送一个`FIN`，那么可以改用`shutdown`函数（6.6节）以代替`close`。我们将在6.5节阐述这么做的动机。

我们还得清楚，如果父进程对每个由`accept`返回的已连接套接字都不调用`close`，那么并发服务器中将会发生什么。首先，父进程最终将耗尽可用描述符，因为任何进程在任何时刻可拥有的打开着的描述符数通常是有限制的。不过更重要的是，没有一个客户连接会被终止。当子进程关闭已连接套接字时，它的引用计数值将由2递减为1且保持为1，因为父进程永不关闭任何已连接套接字。这将妨碍TCP连接终止序列的发生，导致连接一直打开着。



## 4.10 getsockname 和 getpeername 函数

这两个函数或者返回与某个套接字关联的本地协议地址（`getsockname`)，或者返回与某个套接字关联的外地协议地址（`getpeername`)。

### 4.10.1 getsockname()

~~~c
NAME
       getsockname - get socket name

SYNOPSIS
       #include <sys/socket.h>

       int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

DESCRIPTION
       getsockname()  returns  the  current  address  to which the socket sockfd is
       bound, in the buffer pointed to by addr.  The  addrlen  argument  should  be
       initialized  to  indicate the amount of space (in bytes) pointed to by addr.
       On return it contains the actual size of the socket address.

       The returned address is truncated if the buffer provided is  too  small;  in
       this  case,  addrlen  will  return  a value greater than was supplied to the
       call.

RETURN VALUE
       On success, zero is returned.  On error, -1 is returned, and  errno  is  set
       appropriately.
~~~

**作用：获取本端（本地）套接字的地址信息**

也就是说，调用这个函数，是为了知道 **自己绑定的 IP、端口是多少**。

**常见用途：调用 connect() 之后，查看系统自动分配的本地端口：**

~~~c
sockfd = socket(...);
connect(sockfd, ...);   // 未bind，本地端口由内核自动分配

struct sockaddr_in local;
socklen_t len = sizeof(local);
getsockname(sockfd, (struct sockaddr *)&local, &len);

printf("local port = %d\n", ntohs(local.sin_port));

~~~

**或者服务器调用 accept() 后，查看与客户端通信使用的本地端口**，虽然大多数时候和监听端口一样，但也可能不同（如 TCP 负载均衡、`SO_REUSEPORT`）。



### 4.10.2 getpeername()

~~~c
NAME
       getpeername - get name of connected peer socket

SYNOPSIS
       #include <sys/socket.h>

       int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

DESCRIPTION
       getpeername()  returns  the  address  of  the  peer  connected to the socket
       sockfd, in the buffer pointed to by addr.  The addrlen  argument  should  be
       initialized  to  indicate the amount of space pointed to by addr.  On return
       it contains the actual size of the name returned (in bytes).   The  name  is
       truncated if the buffer provided is too small.

       The  returned  address  is truncated if the buffer provided is too small; in
       this case, addrlen will return a value greater  than  was  supplied  to  the
       call.

RETURN VALUE
       On  success,  zero  is returned.  On error, -1 is returned, and errno is set
       appropriately.
~~~

**作用：获取对端（远程）套接字的地址信息**

也就是知道 **连接的另一端的 IP 和端口**。

**常见用途：服务器中打印客户端 IP/端口：**

~~~c
int connfd = accept(...);

struct sockaddr_in peer;
socklen_t len = sizeof(peer);

getpeername(connfd, (struct sockaddr *)&peer, &len);

printf("client = %s:%d\n",
       inet_ntoa(peer.sin_addr),
       ntohs(peer.sin_port));

~~~

**或者TCP 客户端查看自己的服务器连接对象是谁**，检查当前连接是否被重定向、代理等。

### 4.10.3 总结

| 函数              | 返回的是什么地址？              | 使用场景               |
| ----------------- | ------------------------------- | ---------------------- |
| **getsockname()** | 本地地址（local address）       | 知道自己绑定的 IP/端口 |
| **getpeername()** | 对端地址（remote/peer address） | 知道对方的 IP/端口     |

**Q: **如果对`listenfd`调用`getpeername `，对`connfd`调用`getsockname`会发生什么？

**A:** 

1. 对 **listenfd** 调用 **getpeername()** 会返回错误，通常为 `ENOTCONN`（未连接）。

   > 原因：因为只有 **accept() 产生的 connfd 才是真正的连接**，listenfd 不对应任何客户端。
   >
   > - listenfd 是监听套接字，只负责接受连接（accept）
   > - 它**没有与任何对端建立连接**
   > - 所以它没有“peer address”

2. 对**connfd**调用**getsockname()**会正常工作，会返回服务器的“本端地址和端口”。

   > 返回 **服务器端用于与该客户端通信的 IP 和端口号**，对典型的 TCP 服务：
   >
   > ~~~(空)
   > getsockname(connfd, ...) → 返回服务器 IP 和端口，比如 0.0.0.0:80
   > ~~~
   >
   > 如果服务器监听 INADDR_ANY：
   >
   > ~~~(空)
   > local ip = 服务端实际使用的网卡IP（如 192.168.x.x）
   > local port = 监听端口（如 8080）
   > ~~~
   >
   > 如果用了 SO_REUSEPORT 可能不同线程会得到不同端口。

说明：`listenfd` 是半成品，只提供连接队列，不对应“一个具体的 TCP 会话”。`accept()` 返回的 `connfd` 是：

- 已经完成三次握手
- 有明确的对端地址（客户端）
- 有明确的本端地址（服务器为这个连接分配）

所以：

- 对 **connfd** 调 **getpeername()** → 得到客户端地址
- 对 **connfd** 调 **getsockname()** → 得到服务器地址

| 套接字       | getpeername()                  | getsockname()              |
| ------------ | ------------------------------ | -------------------------- |
| **listenfd** | ❌ 不行（ENOTCONN），它没有对端 | ✔ 可以，返回监听的本地地址 |
| **connfd**   | ✔ 得到客户端地址               | ✔ 得到服务器地址           |



# END

