# 第二部分 基本套接字编程

# 第三章 套接字编程简介



## 3.1 概述

本章开始讲解套接字API。我们从套接字地址结构开始讲解，本书中几乎每个例子都用到它们。这些结构可以在两个方向上传递：从进程到内核和从内核到进程。其中从内核到进程方向的传递是值一结果参数的一个例子，我们会在本书中讲到这些参数的许多例子。

地址转换函数在地址的文本表达和它们存放在套接字地址结构中的二进制值之间进行转换。多数现存的IPv4代码使用`inet_addr`和`inet_ntoa`这两个函数，不过两个新函数`inet_pton`和`inet_ntop`同时适用于IPv4和IPv6两种代码。

这些地址转换函数存在的一个问题是它们与所转换的地址类型协议相关，要考虑究竟是IPV4地址还是IPV6地址。为克服这个问题，我们开发了一组名字以`sock_`开头的函数，它们以协议无关方式使用套接字地址结构。我们将贯穿全书使用这组函数，使我们的代码与协议无关。



## 3.2 套接字地址结构

大多数套接字函数都需要一个指向套接字地址结构的指针作为参数。每个协议族都定义它自己的套接字地址结构。这些结构的名字均以`sockaddr_`开头，并以对应每个协议族的唯一后缀结尾。

### 3.2.1 IPv4 套接字地址结构

IPv4套接字地址结构通常也称为“网际套接字地址结构”，它以`sockaddr_in`命名，定义在`<netinet/in.h>`头文件中。下面给出了它的POSIX定义。

~~~c
struct in_addr {
    in_addr_t s_addr;   /* 32-bit IPv4 address (network byte order) */
};
struct sockaddr_in {
    uint8_t			sin_len;	/* length of structure */
    sa_family_t    	sin_family; /* Address family: AF_INET */
    in_port_t      	sin_port;   /* Port number (network byte order) */
    struct in_addr 	sin_addr;   /* Internet address */

    /* Pad to size of 'struct sockaddr' */
    unsigned char  sin_zero[8];
};
~~~

（实际文件中的定义glibc 版本）

~~~c
/* Internet address.  */
typedef uint32_t in_addr_t;
struct in_addr
{
	in_addr_t s_addr;
};

/* Structure describing an Internet socket address.  */
struct sockaddr_in
{
    __SOCKADDR_COMMON (sin_);
    in_port_t sin_port;			/* Port number.  */
    struct in_addr sin_addr;		/* Internet address.  */

    /* Pad to size of `struct sockaddr'.  */
    unsigned char sin_zero[sizeof (struct sockaddr)
                           - __SOCKADDR_COMMON_SIZE
                           - sizeof (in_port_t)
                           - sizeof (struct in_addr)];
};
//------------------------------------
#define __SOCKADDR_COMMON(sa_prefix) \
    sa_family_t sa_prefix##family
~~~

下面将解释`sockaddr_in`的定义：

`sockaddr_in`中的宏是 **glibc 为了可移植性 + ABI 兼容而使用的“宏展开式写法”**，比书本上的“直写版”更底层。以`x86_64`为例，首先：

~~~c
__SOCKADDR_COMMON(sin_)
//等价于
sa_family_t sin_family;
~~~

其次：

| 项目                                    | 大小 |
| --------------------------------------- | ---- |
| `sizeof(struct sockaddr)`               | 16   |
| `__SOCKADDR_COMMON_SIZE` (`sin_family`) | 2    |
| `sizeof(in_port_t)`                     | 2    |
| `sizeof(struct in_addr)`                | 4    |

~~~c
#define __SOCKADDR_COMMON_SIZE (sizeof (sa_family_t))
// 在 64 位 Linux 下：
sizeof(sa_family_t) == 2
// 所以：16-2-2-4=8
unsigned char sin_zero[
    sizeof (struct sockaddr)
  - __SOCKADDR_COMMON_SIZE
  - sizeof (in_port_t)
  - sizeof (struct in_addr)
];
//等价于
unsigned char sin_zero[8]   
~~~

`sin_zero[8]`的作用:把 IPv4 结构“凑齐到 16 字节”,保证`(sockaddr_in *) <-> (sockaddr *)`必须 **安全强转、不越界、不截断**。`sockaddr_in` 与 `sockaddr` 的二进制对齐关系:

~~~(空)
struct sockaddr      (16B)
+-------------------+
| sa_family (2B)    |
| sa_data (14B)     |
+-------------------+

struct sockaddr_in   (16B)
+-------------------+
| sin_family (2B)   |
| sin_port   (2B)   |
| sin_addr   (4B)   |
| sin_zero   (8B)   |
+-------------------+

~~~

展开完整版：这里没有`sin_len`字段，是因为**内核设计差异的原因，Linux的内核从来都没有这个字段**。

~~~c
struct sockaddr_in {
    sa_family_t    sin_family; /* AF_INET */
    in_port_t      sin_port;   /* Port number (network byte order) */
    struct in_addr sin_addr;   /* IPv4 address */

    unsigned char  sin_zero[8]; /* Padding, must be zero */
};

~~~

书中对套戒指地址结构的说明：

* 长度字段`sin_len`是为增加对 OSI 协议的支持而随 4.3 BSD-Reno 添加的。在此之前，第一个成员是`sin_family`，它是一个无符号短整数（`unsigned short`)。并不是所有的厂家都支持套接字地址结构的长度字段，而且POSIX规范也不要求有这个成员。该成员的数据类型`uint8_t`是典型的，符合POSIX的系统都提供这种形式的数据类型。正因为有长度字段，才简化了长度可变套接字地址结构的处理。**【这里说明：Linux 的设计原则是“长度由系统调用参数显式提供，而不是嵌在用户结构里。”】**
* 即使有长度字段，我们也无须设置和检查它，除非涉及路由套接字（见第18章）。它是由处理来自不同协议族的套接字地址结构的例程（例如路由表处理代码）在内核中使用的。
* POsIX规范只需要这个结构中的3个字段：`sin_family`、`sin_addr`和`sin_port`。对于符合POSIX的实现来说，定义额外的结构字段是可以接受的，这对于网际套接字地址结构来说也是正常的。几乎所有的实现都增加了`sin_zero`字段，所以所有的套接字地址结构大小都至少是16字节。
* 我们给出了字段`sin_addr`、`sin_family`和`sin_port`的POSIX数据类型。`in_addr_t`数据类型必须是一个至少32位的无符号整数类型，`in_port_t`必须是一个至少16位的无符号整数类型，而`sa_family_t`可以是任何无符号整数类型。在支持长度字段的实现中，`sa_family_t`通常是一个8位的无符号整数，而在不支持长度字段的实现中，它则是一个16位的无符号整数。
* 我们还将遇到数据类型`u_char`、`u_short`、`u_int`和`u_long`，它们都是无符号的。POSIX规范定义这些类型时特地标记它们已过时，仅是为向后兼容才提供的。
* IPv4地址和TCP或UDP端口号在套接字地址结构中总是以网络字节序来存储。在使用这些字段时，我们必须牢记这一点。我们将在3.4节中详细说明主机字节序与网络字节序的区别。
* 32位IPv4地址存在两种不同的访问方法。举例来说，如果`serv`定义为某个网际套接字地址结构，那么`serv.sin_addr`将按`in_addr`结构引l用其中的32位IPv4地址，而`serv.sin_addr.s_addr`将按`in_addr_t`（通常是一个无符号的32位整数）引用同一个32位IPv4地址。因此，我们必须正确地使用IPv4地址，尤其是在将它作为函数的参数时，因为编译器对传递结构和传递整数的处理是完全不同的。
* `sin_zero`字段未曾使用，不过在填写这种套接字地址结构时，我们总是把该字段置为0。按照惯例，我们总是在填写前把整个结构置为0，而不是单单把`sin_zero`字段置为0。
* 套接字地址结构仅在给定主机上使用：虽然结构中的某些字段（例如IP地址和端口号）用在不同主机之间的通信中，但是结构本身并不在主机之间传递。



### 3.2.2 通用套接字地址结构

当作为一个参数传递进任何套接字函数时，套接字地址结构总是以引用形式（也就是以指向该结构的指针）来传递。然而以这样的指针作为参数之一的任何套接字函数必须处理来自所支持的任何协议族的套接字地址结构。
在如何声明所传递指针的数据类型上存在一个问题。有了ANSIC后解决办法很简单：`void*`是通用的指针类型。然而套接字函数是在ANSIC之前定义的，在1982年采取的办法是在`<sys/socket.h>`头文件中定义一个通用的套接字地址结构。

~~~c
struct sockaddr {
    uint8_t			sa_len;
    sa_family_t		sa_family;		/* address family: AF_XXX value */
    char			sa_data[14];	/* protocol-specific address */
};
~~~

（实际文件中的定义glibc 版本）:

~~~c
#include <bits/socket.h>

/* Structure describing a generic socket address.  */
struct sockaddr
  {
    __SOCKADDR_COMMON (sa_);	/* Common data: address family and length.  */
    char sa_data[14];		/* Address data.  */
  };
#define	__SOCKADDR_COMMON(sa_prefix) \
  sa_family_t sa_prefix##family
~~~

实际上同`sockaddr_in`，`sockaddr`也没有`len`这个参数，宏也是类似的展开方法这里不展开。

于是套接字函数被定义为以指向某个通用套接字地址结构的一个指针作为其参数之一，这正如`bind`函数的ANSI C函数原型所示：`int bind(int，struct sockaddr *，socklen_t);`这就要求对这些函数的任何调用都必须要将指向特定于协议的套接字地址结构的指针进行类型强制转换（casting)，变成指向某个通用套接字地址结构的指针，例如：

~~~c
struct sockaddr_in serv;
/* fill in serv{} */
bind(sockfd, (struct sockaddr *) &serv, sizeof(serv));
~~~

~~~c
NAME
       bind - bind a name to a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
~~~

如果我们省略了其中的类型强制转换部分`(struct sockaddr *)`，并假设系统的头文件中有bind函数的一个ANSIC原型，那么C编译器就会产生这样的警告信息：

~~~(空)
“warning:passing arg2 of'bind' from incompatible pointer type.”
（警告：把不兼容的指针类型传递给‘bind’函数的第二个参数。）
~~~

从应用程序开发人员的观点看，这些通用套接字地址结构的唯一用途就是对指向特定于协议的套接字地址结构的指针执行类型强制转换。



### 3.2.3 IPv6 套接字地址结构

IPv6套接字地址结构在`<netinet/in.h>`头文件中定义。

~~~c
struct in6_addr {
    uint8_t	s6_addr[16];		/* 128-bit IPv6 address */
    							/* network byte ordered */
};
#define	SIN6_LEN				/* required for compile-time tests */
struct sockaddr_in6 {
    uint8_t			sin6_len;		/* length of this struct (28) */
    sa_family_t		sin6_family;	/* AF_INET6 */
    in_port_t		sin6_port;		/* transport layer port# */
    
    uint32_t		sin6_flowinfo;	/* flow information, undefined */
    struct in6_addr	sin6_addr;		/* IPv6 address */
    
    uint32_t		sin6_scope_id	/* set of interfaces for a scope */
};
~~~

（实际文件中的定义glibc 版本）:

~~~c
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
/* Ditto, for IPv6.  */
struct sockaddr_in6
  {
    __SOCKADDR_COMMON (sin6_);
    in_port_t sin6_port;	/* Transport layer port # */
    uint32_t sin6_flowinfo;	/* IPv6 flow information */
    struct in6_addr sin6_addr;	/* IPv6 address */
    uint32_t sin6_scope_id;	/* IPv6 scope-id */
  };
~~~

`sockaddr_in6`的分析方法与之前的类似不再说明，这里解释一下`in6_addr`的定义：

1. **union 共享一块内存：**

~~~c
union
{
    uint8_t  __u6_addr8[16];
    uint16_t __u6_addr16[8];
    uint32_t __u6_addr32[4];
} __in6_u;
/* 同一块内存：
 * +------------------------------------------------+
 * | byte0 | byte1 | ... | byte15                  |
 * +------------------------------------------------+
 * |      uint16[0]     |     uint16[1]   | ...    |
 * +------------------------------------------------+
 * |         uint32[0]          |   uint32[1] ...  |
 * +------------------------------------------------+
 */
~~~

2. **结构体外壳封装**

~~~c
struct in6_addr
{
    union { ... } __in6_u;
};
// 真实 IPv6 地址就存在 addr.__in6_u 这 16 个字节中
~~~

* 方便扩展
* 防止命名污染
* 将具体实现藏在 `__in6_u` 内部

3. **宏定义：给 union 成员起“公开别名”**

~~~c
#define s6_addr __in6_u.__u6_addr8
//让你可以这样写：
struct in6_addr addr;
addr.s6_addr[0] = 0x20;
//而不是：
addr.__in6_u.__u6_addr8[0]
~~~

4. **条件宏 `__USE_MISC` 的作用（兼容性控制）**

   > 这是为了 **标准兼容性 + 向后兼容 BSD**。

下面是书中提到的注意事项：

* 如果系统支持套接字地址结构中的长度字段，那么SIN6_LEN常值必须定义。（linux下不用考虑）

* IPv6的地址族是`AF_INET6`，而IPV4的地址族是`AF_INET`。

* 结构中字段的先后顺序做过编排，使得如果`sockaddr_in6`结构本身是64位对齐的，那么128位的`sin6_addr`字段也是64位对齐的。在一些64位处理机上，如果64位数据存储在某个64位边界位置，那么对它的访问将得到优化处理。（实际上再linux内核的实现中，`sockaddr_in` (IPv4) = 16 字节，`sockaddr_in6` (IPv6) = 28 字节，都是4字节对齐，最大对齐要求只有 4 字节。）

* `sin6_flowinfo`这里分成两部分，首先是原书内容：

  > `sin6_flowinfo`字段分成两个字段：
  >
  > * 低序20bit是流标（flow label）
  > * 高序12bit保留

  较新的说明：

  > `sin6_flowinfo`对应 IPv6 首部中的 Flow Label + Traffic Class，用于：
  >
  > * 标识同一“数据流（Flow）”的分组
  > * 为 QoS（服务质量）、实时流、策略路由提供依据
  > * 让路由器按“流”进行快速转发
  >
  > ~~~(空)
  > 0                   1                   2                   3
  > 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  > +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  > |Version| Traffic Class |           Flow Label                  |
  > +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  >  4bit        8bit                   20bit
  > 
  > sin6_flowinfo (32bit) =
  > ┌────────────┬────────────────┬────────────────────────┐
  > │ Version(4) │ TrafficClass(8)│ FlowLabel (20)         │
  > └────────────┴────────────────┴────────────────────────┘
  > ~~~
  >
  > 严格来说 **Version 不应该由应用设置**，但历史原因 glibc 把整个 32bit 暴露给了用户。
  >
  > 但是大部分情况下不使用该字段，默认填0即可。

* 对于具备范围的地址（scoped address)，`sin6_scope_id`字段标识其范围（scope)，最常见的是链路局部地址（link-local address）的接口索引（interface index）。



### 3.2.4 新的通用套接字地址结构

作为IPv6套接字API的一部分而定义的新的通用套接字地址结构克服了现有`struct sockaddr`的一些缺点。不像`struct sockaddr`，新的`struct sockaddr_storage`足以容纳系统所支持的任何套接字地址结构。

`sockaddr_storage`结构在<netinet/in.h>头文件中定义。

~~~c
struct sockaddr_storage {
  	uint8_t			ss_len;		/* length of this struct (implementation dependent) */
    sa_family_t		ss_family;	/* address family: AF_XXX value */
    /* implementation-dependent elements to provide:
	 * a） alignment sufficient to fulfill the alignment requirements of
	 * 	   all socket address types that the system supports.
	 * b) enough storage to hold any type of socket address that the
	 *     system supports.
	 */
};
~~~

（实际文件中的定义glibc 版本）:

~~~c
#define _SS_SIZE 128
#define __SOCKADDR_COMMON_SIZE sizeof (sa_family_t)

/* Structure large enough to hold any socket address (with the historical
   exception of AF_UNIX).  */
#define __ss_aligntype	unsigned long int
#define _SS_PADSIZE \
  (_SS_SIZE - __SOCKADDR_COMMON_SIZE - sizeof (__ss_aligntype))
struct sockaddr_storage
  {
    __SOCKADDR_COMMON (ss_);	/* Address family, etc.  */
    char __ss_padding[_SS_PADSIZE];
    __ss_aligntype __ss_align;	/* Force desired alignment.  */
  };
~~~

下面是对结构实现的分析：

1. **第一层展开：`__SOCKADDR_COMMON(ss_)`**

   > 展开后：`sa_family_t  ss_family;`

2. **第二层宏：`__ss_aligntype`**

   > ~~~c
   > #define __ss_aligntype unsigned long int
   > ~~~
   >
   > 选一个 **“平台上对齐要求最大的基础类型”**，通常：
   >
   > * x86_64 → `unsigned long` = **8 字节，对齐 8**
   > * x86_32 → 4 字节，对齐 4
   >
   > `sockaddr_storage` 能满足 **所有 socket 地址结构体的最大对齐要求**。

3. **关键宏：`_SS_PADSIZE` 的真实含义**

   > ~~~c
   > #define _SS_PADSIZE \
   >   (_SS_SIZE - __SOCKADDR_COMMON_SIZE - sizeof (__ss_aligntype))
   > ~~~
   >
   > 它的意思是：**整个结构体最终大小 − family 字段 − 对齐保留字段 = 中间 padding 的大小。**
   >
   > 也就是说：总大小 = family + padding + align，以x86_64体系为例：
   >
   > ~~~(空)
   > 偏移 0   : sa_family_t ss_family  (2B)
   > 偏移 2   : padding[118]           (118B)
   > 偏移 120 : unsigned long align    (8B)
   > ---------------------------------------
   > 总大小：128 字节
   > 整体对齐：8 字节
   > ~~~
   >
   > **最终大小 = 128 B，对齐 = 8 B（由 unsigned long 决定）。**

4. **`__ss_align` 的真实作用（非常关键）**

   > ~~~c
   > __ss_aligntype __ss_align;
   > ~~~
   >
   > 不是为了存数据，而是为了 **强制结构体拥有“最大基础类型对齐”能力**。
   >
   > 如果没有它：
   >
   > * 某些平台可能只 4 字节对齐
   > * 但 `sockaddr_in6` 可能需要 8 对齐
   > * 强制转型会造成 **未对齐访问 → 总线错误（某些架构）**
   >
   > 防止未对齐访问的 **硬件安全保险字段**。



### 3.2.5 套接字地址结构的比较

| 结构体                    | 支持协议   | 典型大小 | 是否通用 | 主要用途                         |
| ------------------------- | ---------- | -------- | -------- | -------------------------------- |
| `struct sockaddr`         | 通用“基类” | 16B      | ✅        | 作为接口统一参数使用（需强转）   |
| `struct sockaddr_in`      | IPv4       | 16B      | ❌        | IPv4 地址与端口                  |
| `struct sockaddr_in6`     | IPv6       | 28B      | ❌        | IPv6 地址与端口                  |
| `struct sockaddr_storage` | 任意协议   | 128B     | ✅        | 通用安全容器，防止越界与对齐问题 |
| `struct sockaddr_un`      | 本地 UNIX  | ~110B    | ❌        | Unix 域套接字（文件路径）        |
| `struct sockaddr_ll`      | 原始链路层 | 20+B     | ❌        | 抓包、原始以太网帧               |

#### 关键结论（记住这 4 点就够了）

* ✅ **API 传参统一用 `struct sockaddr \*`**

- ✅ **接收未知协议地址用 `struct sockaddr_storage`**
- ✅ **IPv4 用 `sockaddr_in`，IPv6 用 `sockaddr_in6`**
- ✅ **`sockaddr_storage` 是唯一“协议无关 + 安全 + 对齐正确”的结构**



## 3.3 值-结果参数

我们提到过，当往一个套接字函数传递一个套接字地址结构时，该结构总是以引用形式来传递，也就是是说传递的是指向该结构的一个指针。该结构的长度也作为一个参数来传递，不过其传递方式和取决于该结构的传递方向：是从进程到内核，还是从内核到进程。

（1）从进程到内核传递套接字地址结构的函数有三个：bind、connect和sendto。这些函数的一个参数是指向某个套接字地址结构的指针，另一个参数是该结构体的整数大小，例如：

~~~c
struct sockaddr_in serv;
/* fill in serv{} */
connect(sockfd, (struct sockaddr *) &serv, sizeof(serv));
~~~

~~~c
NAME
       connect - initiate a connection on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);
~~~

既然指针和指针所指内容的大小都传递给了内核，于是内核知道到底需从进程复制多少数据进来。下展示了这个情形。

~~~(空)
                             用户进程
           +--------------------------------------+
           |       int                            |
           |   +-----------+    +---------------+ |
           |   |    长度   |     |     套接字     | |
           |   +-----------+    |     地址结构   |  |
           |         |          +---------------+ |
           |      值 |                   |         |
           |         |                   |         |
           +---------+-------------------+---------+
                     |                   |     
                     |                   | 协议地址
           +---------+-------------------+----------+
           |         |                   |          |
           |         v       内核         v          |
           |                                        |
           +----------------------------------------+
~~~

我们将在下一章中看到，套接字地址结构大小的数据类型实际上是`socklen_t`，而不是`int`，不过POSIX规范建议将`socklen_t`定义为`uint32_t`。

（2）从内核到进程传递套接字地址结构的函数有4个：`accept`、`recvfrom`、`getsockname`和`getpeername`。这4个函数的其中两个参数是指向某个套接字地址结构的指针和指向表示该结构大小的整数变量的指针。例如：

~~~c
struct sockaddr_un	cli;	/* Unix domain */
socklen_t len;

len = sizeof(cli); 			/* len is a value */
getpeername(unixfd, (struct sockaddr *) &cli, &len); /*len may havve changed  */
~~~

~~~c
NAME
       getpeername - get name of connected peer socket

SYNOPSIS
       #include <sys/socket.h>

       int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
~~~

把套接字地址结构大小这个参数从一个整数改为指向某个整数变量的指针，其原因在于：当函数被调用时，结构大小是一个值（value)，它告诉内核该结构的大小，这样内核在写该结构时不至于越界；当函数返回时，结构大小又是一个结果（result)，它告诉进程内核在该结构中究竟存储了多少信息。这种类型的参数称为值-结果（value-result）参数。下图展示了这个情形。

~~~(空)
                             用户进程
           +--------------------------------------+
           |       int*                           |
           |   +-----------+    +---------------+ |
           |   |    长度   |     |     套接字     | |
           |   +-----------+    |     地址结构   |  |
           |       |    ^       +---------------+ |
           |    值 |    | 结果          ^          |
           |       |    |              |          |
           +-------+----+--------------+----------+
                   |    |              |     
                   |    |              |  协议地址
           +-------+----+--------------+----------+
           |       |    |              |          |
           |       v       内核                    |
           |                                      |
           +--------------------------------------+
~~~

当使用值一结果参数作为套接字地址结构的长度时，如果套接字地址结构是固定长度的，那么从内核返回的值总是那个固定长度，例如IPv4的`sockaddr_in`长度是16，IPv6的`sockaddr_in6`长度是28。然而对于可变长度的套接字地址结构（例如Unix域的`sockaddr_un`)，返回值可能小于该结构的最大长度。

在网络编程中，值一结果参数最常见的例子是所返回套接字地址结构的长度。不过本书中我们还会碰到其他值一结果参数，以后再说。



## 3.4 字节排序函数

考虑一个16位整数，它由2个字节组成。内存中存储这两个字节有两种方法：一种是将低序字节存储在起始地址，这称为小端（little-endian）字节序；另一种方法是将高序字节存储在起始地址，这称为大端（big-endian）字节序。

~~~
          （高字节    低字节）
0x1234 = 0001 0010 0011 0100
            12h       34h  
------------------------------------------------
大端存储（Big-Endian）: 高位字节 → 低地址         
低地址 → → → → → → → → → → → → → 高地址
+---------+---------+
|  0x12   |  0x34   |
+---------+---------+
 ↑低地址             ↑高地址
------------------------------------------------
小端存储（Little-Endian）: 低位字节 → 低地址
低地址 → → → → → → → → → → → → → 高地址
+---------+---------+
|  0x34   |  0x12   |
+---------+---------+
 ↑低地址             ↑高地址
~~~

在c代码中：

~~~c
uint16_t x = 0x1234;
uint8_t *p = (uint8_t *)&x;

printf("%02x %02x\n", p[0], p[1]);
~~~

* 输出 `12 34` → 大端
* 输出 `34 12` → 小端（x86 就是这样）

### linux 上获取本机字节序方法

* **用户态 - 最标准的方式**：`<endian.h>`（glibc / Linux）Linux 用户态最推荐的方法

~~~c
#include <endian.h>

#if __BYTE_ORDER == __LITTLE_ENDIAN
    // 小端
#elif __BYTE_ORDER == __BIG_ENDIAN
    // 大端
#else
    #error "Unknown byte order"
#endif

~~~

| 宏                | 含义             |
| ----------------- | ---------------- |
| `__BYTE_ORDER`    | 当前主机字节序   |
| `__LITTLE_ENDIAN` | 小端常量         |
| `__BIG_ENDIAN`    | 大端常量         |
| `__PDP_ENDIAN`    | PDP 端（极少见） |

* **GCC / Clang 内建宏（最底层、跨平台）**: **不依赖任何头文件**的方式

~~~(空)
#if __BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__
    // 小端
#elif __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
    // 大端
#endif
~~~

这是 **编译器级别最稳的方案（内核、裸机常用）**。

* **内核态判断（你如果在写驱动）**: 

~~~c
#include <asm/byteorder.h>

#if defined(__LITTLE_ENDIAN)
#elif defined(__BIG_ENDIAN)
#endif
// 或者
#include <linux/byteorder.h>

#if BYTE_ORDER == LITTLE_ENDIAN
#endif
~~~



C 标准本身不定义“字节序宏”，必须靠 glibc、编译器或运行期探测；在 Linux 下最规范的是 `<endian.h>`，在跨平台项目中最稳的是 `__BYTE_ORDER__`。推荐使用方法：

~~~c
#if __BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__
    #define HOST_LITTLE_ENDIAN 1
#elif __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
    #define HOST_BIG_ENDIAN 1
#else
    #error "Unknown endian"
#endif

~~~



## 3.5 字节操纵函数

==本节将不再记录书中的函数方法，将直接从Linux的man手册中复制函数的说明及用法。==

操纵多字节字段的函数有两组，它们既不对数据作解释，也不假设数据是以空字符结束的C字符串。当处理套接字地址结构时，我们需要这些类型的函数，因为我们需要操纵诸如IP地址这样的字段，这些字段可能包含值为0的字节，却并不是C字符串。以空字符结尾的C字符串是由在`<string.h>`头文件中定义、名字以`str`（表示字符串）开头的函数处理的。

名字以`b`（表示字节）开头的第一组函数起源于`4.2 BSD`，几乎所有现今支持套接字函数的系统仍然提供它们。名字以`mem`（表示内存）开头的第二组函数起源于`ANSI C`标准，支持`ANSI C`函数库的所有系统都提供它们。

我们首先给出源自Berkeley的函数，本书中我们只使用其中一个——`bzero`。（我们使用它是因为它只有2个参数，比起3个参数的`memset`函数来要容易记些，这在前边已解释过。）其他两个函数`bcopy`和`bcmp`你也许会在现有的应用程序中见到。

### bzero、bcopy、bcmp

~~~c
SYNOPSIS
       #include <strings.h>
NAME
       bzero, explicit_bzero - zero a byte string
       void bzero(void *s, size_t n);
NAME
       bcopy - copy byte sequence
       void bcopy(const void *src, void *dest, size_t n);
NAME
       bcmp - compare byte sequences
       int bcmp(const void *s1, const void *s2, size_t n);
~~~

`bzero`把目标字节串中指定数目的字节置为0。我们经常使用该函数来把一个套接字地址结构初始化为0。`bcopy`将指定数目的字节从源字节串移到目标字节串。`bcmp`比较两个任意的字节串，若相同则返回值为0，否则返回值为非0。

### memset、memcpy、memcmp

~~~c
SYNOPSIS
       #include <string.h>
NAME
       memset - fill memory with a constant byte
       void *memset(void *s, int c, size_t n);
NAME
       memcpy - copy memory area
       void *memcpy(void *dest, const void *src, size_t n);
NAME
       memcmp - compare memory areas
       int memcmp(const void *s1, const void *s2, size_t n);
~~~

`memset`把目标字节串指定数目的字节置为值c。`memcpy`类似`bcopy`，不过两个指针参数的顺序是相反的。==**当源字节串与目标字节串重叠时，`bcopy`能够正确处理，但是`memcpy`的操作结果却不可知**==。这种情形下必须改用`ANSI C`的`memmove`函数。

> 记住`memcpy`两个指针参数顺序的方法之一是记着它们是按照与C中的赋值语句相同的顺序从左到右书写的：`dest = src;`。记住memset最后两个参数顺序的方法之一是认识到所有ANSIC的`memxxx`函数都需要一个长度参数，而且它总是最后一个参数。

`memcmp`比较两个任意的字节串，若相同则返回0，否则返回一个非0值，是大于0还是小于0则取决于第一个不等的字节：如果`s1`所指字节串中的这个字节大于`s2`所指字节中的对应字节，那么大于0，否则小于0。我们的比较操作是在假设两个不等的字节均为无符号字符(`unsigned char`）的前提下完成的。



## 3.6 inet_ation、inet_addr 和 inet_ntoa 函数

在本节和下一节，我们介绍两组地址转换函数。它们在`ASCII`字符串（这是人们偏爱使用的格式）与网络字节序的二进制值（这是存放在套接字地址结构中的值）之间转换网际地址。

（1）`inet_aton`、`inet_addr`和`inet_ntoa`在点分十进制数串（例如“`206.168.112.96`”)与它长度为32位的网络字节序二进制值间转换IPv4地址。你可能会在许多现有代码中见到这些函数。

（2）两个较新的函数`inet_pton`和`inet_ntop`对于IPv4地址和IPv6地址都适用。我们将在下一节中讲解它们并在全书中使用它们。

~~~c
NAME
       inet_aton,  inet_addr,  inet_network,  inet_ntoa, inet_makeaddr, inet_lnaof, inet_netof - Internet ad‐
       dress manipulation routines

SYNOPSIS
       #include <sys/socket.h>
       #include <netinet/in.h>
       #include <arpa/inet.h>

       int inet_aton(const char *cp, struct in_addr *inp);

       in_addr_t inet_addr(const char *cp);

       in_addr_t inet_network(const char *cp);

       char *inet_ntoa(struct in_addr in);

       struct in_addr inet_makeaddr(in_addr_t net, in_addr_t host);

       in_addr_t inet_lnaof(struct in_addr in);

       in_addr_t inet_netof(struct in_addr in);

   Feature Test Macro Requirements for glibc (see feature_test_macros(7)):
~~~

第一个函数`inet_aton`将`cp`所指C字符串转换成一个32位的网络字节序二进制值，并通过指针`inp`来存储。若成功则返回1，否则返回0。

>  `inet_aton`函数有一个没写入正式文档中的特征：如果`inp`指针为空，那么该函数仍然对输入的字符串执行有效性检查，但是不存储任何结果。

`inet_addr`进行相同的转换，返回值为32位的网络字节序二进制值。该函数存在一个问题：所有`2^32`个可能的二进制值都是有效的IP地址（从`0.0.0.0`到`255.255.255.255`)，但是当出错时该函数返回`INADDR_NONE`常值（通常是一个32位均为1的值）。这意味着点分十进制数串`255.255.255.255`（这是IPv4的有限广播地址，不能由该函数处理，因为它的二进制值被用来指涉该函数失败。

> `inet_addr`函数还存在一个潜在的问题：一些手册页面声明该函数出错时返回`-1`而不是`INADDR_NONE`。这样在对该函数的返回值（一个无符号的值）和一个负常值（`-1`）进行比较时可能会发生问题，具体取决于C编译器。

如今`inet_addr`已被废弃，新的代码应该改用`inet_aton`函数。更好的办法是使用下一节中介绍的新函数，它们对于`IPv4`地址和`IPv6`地址都适用。

`inet_ntoa`函数将一个32位的网络字节序二进制IPv4地址转换成相应的点分十进制数串。由该函数的返回值所指向的字符串驻留在静态内存中。这意味着该函数是不可重入的。最后需要留意，该函数以一个结构而不是以指向该结构的一个指针作为其参数。



## 3.7 inet_pton 和 inet_ntop 函数

这两个函数是随`IPv6`出现的新函数，对于`IPv4`地址和`IPv6`地址都适用。本书通篇都在使用这两个函数。函数名中`p`和`n`分别代表 表达（`presentation`）和数值（`numeric`)。地址的表达格式通常是ASCI字符串，数值格式则是存放到套接字地址结构中的二进制值。

~~~c
SYNOPSIS
       #include <arpa/inet.h>
NAME
       inet_pton - convert IPv4 and IPv6 addresses from text to binary form
       int inet_pton(int af, const char *src, void *dst);
NAME
       inet_ntop - convert IPv4 and IPv6 addresses from binary to text form
       const char *inet_ntop(int af, const void *src,
                             char *dst, socklen_t size);
--------------------------------------------------------------------------------
AF_INET
		src  points  to  a struct in_addr (in network byte order) which 
     	is converted to an IPv4 network address in the dotted-decimal format, 
		"ddd.ddd.ddd.ddd".  The  buffer  dst  must  be  at  least 
        INET_ADDRSTRLEN bytes long.

AF_INET6
    	src  points to a struct in6_addr (in network byte order) which is 
        converted to a representation of this address in the most appropriate 
        IPv6 network address format for this address. The buffer dst must be 
        at least INET6_ADDRSTRLEN bytes long.
~~~

这两个函数的`family`参数既可以是`AF_INET`，也可以是`AF_INET6`。如果以不被支持的地址族作为`family`参数，这两个函数就都返回一个错误，并将`errno`置为`EAFNOSUPPORT`。第一个函数尝试转换由`char *src`指针所指的字符串，并通过`void *dst`指针存放二进制结果。若成功则返回值为1，否则如果对所指定的`family`而言输入的字符串不是有效的表达格式，那么返回值为0。

`inet_ntop`进行相反的转换，从数值格式（`void *src`)转换到表达格式（`char *dst`)。len参数是目标存储单元的大小，以免该函数溢出其调用者的缓冲区。为有助于指定这个大小，在`<netinet/in.h>`头文件中有如下定义：

~~~c
#define INET_ADDRSTRLEN 16
#define INET6_ADDRSTRLEN 46
~~~

如果`len`太小，不足以容纳表达格式结果（包括结尾的空字符），那么返回一个空指针，并置`errno`为`ENOSPC`。

`inet_ntop`函数的`dst`参数不可以是一个空指针。调用者必须为目标存储单元分配内存并指定其大小。调用成功时，这个指针就是该函数的返回值。

`inet_pton` 的返回值

```
返回 1  → 成功
返回 0  → 非法地址字符串
返回 -1 → af 不支持（errno 设置）
```

`inet_ntop` 的返回值

```
成功 → 返回 buf 指针
失败 → 返回 NULL（errno 设置）
```

**使用示例：**

* IPv4：字符串 ⇄ 二进制

~~~c
#include <stdio.h>
#include <arpa/inet.h>

int main(void)
{
    const char *ip_str = "192.168.1.100";
    struct in_addr addr;

    /* 字符串 -> 网络字节序二进制 */
    if (inet_pton(AF_INET, ip_str, &addr) != 1) {
        perror("inet_pton IPv4 failed");
        return 1;
    }

    printf("IPv4 binary (network order): 0x%08x\n", addr.s_addr);

    /* 二进制 -> 字符串 */
    char buf[INET_ADDRSTRLEN];   // 16 字节
    if (inet_ntop(AF_INET, &addr, buf, sizeof(buf)) == NULL) {
        perror("inet_ntop IPv4 failed");
        return 1;
    }

    printf("IPv4 string: %s\n", buf);
    return 0;
}

~~~



* IPv6：字符串 ⇄ 二进制

~~~c
#include <stdio.h>
#include <arpa/inet.h>

int main(void)
{
    const char *ip6_str = "2001:db8::1";
    struct in6_addr addr6;

    /* 字符串 -> 网络字节序二进制 */
    if (inet_pton(AF_INET6, ip6_str, &addr6) != 1) {
        perror("inet_pton IPv6 failed");
        return 1;
    }

    /* 二进制 -> 字符串 */
    char buf[INET6_ADDRSTRLEN];  // 46 字节
    if (inet_ntop(AF_INET6, &addr6, buf, sizeof(buf)) == NULL) {
        perror("inet_ntop IPv6 failed");
        return 1;
    }

    printf("IPv6 string: %s\n", buf);
    return 0;
}

~~~



## 3.8 sock_ntop 和相关函数

【sock_ntop 是自建函数不是系统调用】

`inet_ntop`的一个基本问题是：它要求调用者传递一个指向某个二进制地址的指针，而该地址通常包含在一个套接字地址结构中，这就要求调用者必须知道这个结构的格式和地址族。这就是说，为了使用这个函数，我们必须为IPv4编写如下代码：

~~~c
struct sockaddr_in addr;
inet_ntop(AF_INET, &addr.sin_addr, str, sizeof (str)）;
~~~

或为IPv6编写如下代码：

~~~c
struct sockaddr_in6 addr6;
inet_ntop(AF_INET6, &addr6.sin6_addr, str, sizeof (str)）;
~~~

这就使得我们的代码与协议相关了。为了解决这个问题，我们将自行编写一个名为`sock_ntop`的函数，它以指向某个套接字地址结构的指针为参数，查看该结构的内部，然后调用适当的函数返回该地址的表达格式。

~~~c
char *sock_ntop(const struct sockaddr *sockaddr, socklen_t addrlen);
				// 返回: 若成功则非空指针，若出错则为NULL
~~~

> 这就是本书通篇使用的我们自己定义的函数（非标准系统函数）的说明形式：包围函数原型和返回值的方框是虚线。开头包括的头文件通常是我们自己的`unp.h`。

`sockaddr`指向一个长度为`addrlen`的套接字地址结构。本函数用它自己的静态缓冲区来保存结果，而指向该缓冲区的一个指针就是它的返回值。

> 注意：对结果进行静态存储导致该函数不可重入且非线程安全。这些概念我们将在11.18节中进一步讨论。对于该函数我们作这样的设计决策是为了让本书中的简单例子方便地调用它。

表达格式就是在一个`IPv4`地址的点分十进制数串格式之后，或者在一个括以方括号的`IPv6`地址的十六进制数串格式之后，跟一个终止符（我们使用一个分号，类似于URL语法)，再跟一个十进制的端口号，最后跟一个空字符。因此，缓冲区大小对于`IPv4`至少为`INET_ADDRSTRLEN`加上6个字节（16+6=22），对于`IPv6`至少为`INET6_ADDRSTRLEN`加上8个字节（46+8=54)。

~~~c
#include	"unp.h"

#ifdef	HAVE_SOCKADDR_DL_STRUCT
#include	<net/if_dl.h>
#endif

/* include sock_ntop */
char *
sock_ntop(const struct sockaddr *sa, socklen_t salen)
{
    char		portstr[8];
    static char str[128];		/* Unix domain is largest */

	switch (sa->sa_family) {
	case AF_INET: {
		struct sockaddr_in	*sin = (struct sockaddr_in *) sa;

		if (inet_ntop(AF_INET, &sin->sin_addr, str, sizeof(str)) == NULL)
			return(NULL);
		if (ntohs(sin->sin_port) != 0) {
			snprintf(portstr, sizeof(portstr), ":%d", ntohs(sin->sin_port));
			strcat(str, portstr);
		}
		return(str);
	}
/* end sock_ntop */

#ifdef	IPV6
	case AF_INET6: {
		struct sockaddr_in6	*sin6 = (struct sockaddr_in6 *) sa;

		str[0] = '[';
		if (inet_ntop(AF_INET6, &sin6->sin6_addr, str + 1, sizeof(str) - 1) == NULL)
			return(NULL);
		if (ntohs(sin6->sin6_port) != 0) {
			snprintf(portstr, sizeof(portstr), "]:%d", ntohs(sin6->sin6_port));
			strcat(str, portstr);
			return(str);
		}
		return (str + 1);
	}
#endif

#ifdef	AF_UNIX
	case AF_UNIX: {
		struct sockaddr_un	*unp = (struct sockaddr_un *) sa;

			/* OK to have no pathname bound to the socket: happens on
			   every connect() unless client calls bind() first. */
		if (unp->sun_path[0] == 0)
			strcpy(str, "(no pathname bound)");
		else
			snprintf(str, sizeof(str), "%s", unp->sun_path);
		return(str);
	}
#endif

#ifdef	HAVE_SOCKADDR_DL_STRUCT
	case AF_LINK: {
		struct sockaddr_dl	*sdl = (struct sockaddr_dl *) sa;

		if (sdl->sdl_nlen > 0)
			snprintf(str, sizeof(str), "%*s (index %d)",
					 sdl->sdl_nlen, &sdl->sdl_data[0], sdl->sdl_index);
		else
			snprintf(str, sizeof(str), "AF_LINK, index=%d", sdl->sdl_index);
		return(str);
	}
#endif
	default:
		snprintf(str, sizeof(str), "sock_ntop: unknown AF_xxx: %d, len %d",
				 sa->sa_family, salen);
		return(str);
	}
    return (NULL);
}

char *
Sock_ntop(const struct sockaddr *sa, socklen_t salen)
{
	char	*ptr;

	if ( (ptr = sock_ntop(sa, salen)) == NULL)
		err_sys("sock_ntop error");	/* inet_ntop() sets errno */
	return(ptr);
}
~~~

我们还为操作套接字地址结构定义了其他几个函数，它们将简化我们的代码在`IPv4`与`IPv6`之间的移植。

~~~c
int		 sock_bind_wild(int, int);
int		 sock_cmp_addr(const SA *, const SA *, socklen_t);
int		 sock_cmp_port(const SA *, const SA *, socklen_t);
int		 sock_get_port(const SA *, socklen_t);
void	 sock_set_addr(SA *, socklen_t, const void *);
void	 sock_set_port(SA *, socklen_t, int);
void	 sock_set_wild(SA *, socklen_t);
~~~

说明：略。



## 3.9 readn、writen 和 readline 函数

【readn、writen 和 readline 是自建函数不是系统调用】

字节流套接字（例如TCP套接字）上的`read`和`write`函数所表现的行为不同于通常的文件`I/O`。字节流套接字上调用`read`或`write`输入或输出的字节数可能比请求的数量少，然而这不是出错的状态。这个现象的原因在于内核中用于套接字的缓冲区可能已达到了极限。此时所需的是调用者再次调用`read`或`write`函数，以输入或输出剩余的字节。有些版本的Unix在往一个管道中写多于4096字节的数据时也会表现出这样的行为。这个现象在`read`一·个字节流套接字时很常见，但是在`write`一个字节流套接字时只能在该套接字为非阻塞的前提下才出现。尽管如此，为预防万一，不让实现返回一个不足的字节计数值，我们总是改为调用`writen`函数来取代`write`函数。我们提供的以下3个函数是每当我们读或写一个字节流套接字时总要使用的函数。

~~~c
ssize_t	 readn(int, void *, size_t);
ssize_t	 writen(int, const void *, size_t);

ssize_t	 readline(int, void *, size_t);
~~~

具体实现：

* readn

~~~c
/* include readn */
#include	"unp.h"

ssize_t						/* Read "n" bytes from a descriptor. */
readn(int fd, void *vptr, size_t n)
{
	size_t	nleft;
	ssize_t	nread;
	char	*ptr;

	ptr = vptr;
	nleft = n;
	while (nleft > 0) {
		if ( (nread = read(fd, ptr, nleft)) < 0) {
			if (errno == EINTR)
				nread = 0;		/* and call read() again */
			else
				return(-1);
		} else if (nread == 0)
			break;				/* EOF */

		nleft -= nread;
		ptr   += nread;
	}
	return(n - nleft);		/* return >= 0 */
}
/* end readn */

ssize_t
Readn(int fd, void *ptr, size_t nbytes)
{
	ssize_t		n;

	if ( (n = readn(fd, ptr, nbytes)) < 0)
		err_sys("readn error");
	return(n);
}

~~~

* writen

~~~c
/* include writen */
#include	"unp.h"

ssize_t						/* Write "n" bytes to a descriptor. */
writen(int fd, const void *vptr, size_t n)
{
	size_t		nleft;
	ssize_t		nwritten;
	const char	*ptr;

	ptr = vptr;
	nleft = n;
	while (nleft > 0) {
		if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
			if (nwritten < 0 && errno == EINTR)
				nwritten = 0;		/* and call write() again */
			else
				return(-1);			/* error */
		}

		nleft -= nwritten;
		ptr   += nwritten;
	}
	return(n);
}
/* end writen */

void
Writen(int fd, void *ptr, size_t nbytes)
{
	if (writen(fd, ptr, nbytes) != nbytes)
		err_sys("writen error");
}

~~~

* readline

~~~c
/* include readline */
#include	"unp.h"
/* PAINFULLY SLOW VERSION-- example Only */
ssize_t
readline(int fd, void *vptr, size_t maxlen)
{
	ssize_t	n, rc;
	char	c, *ptr;

	ptr = vptr;
	for (n = 1; n < maxlen; n++) {
    again:
		if ( (rc = read(fd, &c, 1)) == 1) {
			*ptr++ = c;
			if (c == '\n')
				break;		/* newline is stored, like fgets{} */
		} else if (rc == 0) {
			*ptr = 0;
			return(n - 1);	/* EOF, n -1 bytes were read */

		} else
            if (errno == EINTR)
                goto again;
			return(-1);		/* error, errno set, by read{} */
	}
	*ptr = 0;				/* null terminate like fgets{} */
	return(n);
}
/* end readline */

ssize_t
Readline(int fd, void *ptr, size_t maxlen)
{
	ssize_t		n;

	if ( (n = readline(fd, ptr, maxlen)) == -1)
		err_sys("readline error");
	return(n);
}

~~~

上述三个函数查找`EINTR`错误（表示系统调用被一个捕获的信号中断，我们将在5.9节中更详细地讨论)，如果发生该错误则继续进行读或写操作。既然这些函数的作用是避免让调用者来处理不足的字节计数值，那么我们就地处理该错误，而不是强迫调用者再次调用`readn`或`writen`函数。在14.3节我们会提到，`MSG_WAITALL`标志可随`recv`函数一起使用来取代独立的`readn`函数。

注意，这个`readline`函数每读一个字节的数据就调用一次系统的`read`函数。这是非常低效率的，为此我们特意在代码中注明“`PAINFULLYSLOW`（极端地慢)”。当面临从某个套接字读入文本行这一需求时，改用标准`I/O`函数库（称为stdio）相当诱人。我们将在14.8节中详细讨论这种方法，不过预先指出这是种危险的方法。解决本性能问题的stdio缓冲机制却引发许多后勤问题，可能导致在应用程序中存在相当隐蔽的缺陷。**究其原因在于stdio缓冲区的状态是不可见的。**为便于深入解释，让我们考虑客户和服务器之间的一个基于文本行的协议，而使用该协议的多个客户程序和服务器程序可能是在一段时间内先后实现的（这种情形其实相当普遍，举例来说，按照`HTTP`规范独立编写的Web浏览器程序和Web服务器程序就相当之多）。良好的防御性编程（defensive programming）技术要求这些程序不仅能够期望它们的对端程序也遵循相同的网络协议，而且能够检查出未预期的网络数据传送并加以修正（恶意企图自然也被检查出来），这样使得网络应用能够从存在问题的网络数据传送中恢复，可能的话还会继续工作。==为了提升性能而使用stdio来缓冲数据违背了这些目标，因为这样的应用进程在任何时刻都没有办法分辨stdio缓冲区中是否持有未预期的数据。==

基于文本行的网络协议相当多，警如`SMTP`、`HTTP`、`FTP`的控制连接协议以及`finger`等。因此针对文本行操作这一需求一再被提出。然而我们的建议是依照缓冲区而不是文本行的要求来考虑编程。编写从缓冲区中读取数据的代码，当期待一个文本行时，就查看缓冲区中是否含有那一行。

下图给出了`readline`函数的一个较快速版本，它使用自己的而不是stdio提供的缓冲机制。其中重要的是`readline`内部缓冲区的状态是暴露的，这使得调用者能够查看缓冲区中到底收到了什么。即使使用这个特性，`readline`仍可能存在问题，具体见6.3节。诸如`select`等系统函数仍然不可能知道`readline`使用的内部缓冲区，因此编写不严谨的程序很可能发现自己在`select`上等待的数据早已收到并存放在`readline`的缓冲区中了。由于这个原因，混合调用`readn`和`readline`不会像预期的那样工作，除非把`readn`修改成也检查该内部缓冲区。

~~~c
/* include readline */
#include	"unp.h"

static int	read_cnt;
static char	*read_ptr;
static char	read_buf[MAXLINE];

static ssize_t
my_read(int fd, char *ptr)
{

	if (read_cnt <= 0) {
again:
		if ( (read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
			if (errno == EINTR)
				goto again;
			return(-1);
		} else if (read_cnt == 0)
			return(0);
		read_ptr = read_buf;
	}

	read_cnt--;
	*ptr = *read_ptr++;
	return(1);
}

ssize_t
readline(int fd, void *vptr, size_t maxlen)
{
	ssize_t	n, rc;
	char	c, *ptr;

	ptr = vptr;
	for (n = 1; n < maxlen; n++) {
		if ( (rc = my_read(fd, &c)) == 1) {
			*ptr++ = c;
			if (c == '\n')
				break;	/* newline is stored, like fgets() */
		} else if (rc == 0) {
			*ptr = 0;
			return(n - 1);	/* EOF, n - 1 bytes were read */
		} else
			return(-1);		/* error, errno set by read() */
	}

	*ptr = 0;	/* null terminate like fgets() */
	return(n);
}

ssize_t
readlinebuf(void **vptrptr)
{
	if (read_cnt)
		*vptrptr = read_ptr;
	return(read_cnt);
}
/* end readline */

ssize_t
Readline(int fd, void *ptr, size_t maxlen)
{
	ssize_t		n;
	if ( (n = readline(fd, ptr, maxlen)) < 0)
		err_sys("readline error");
	return(n);
}

~~~

内部函数`my_read`每次最多读`MAXLINE`个字符，然后每次返回一个字符。`readline`函数本身的唯一变化是用`my_read`调用取代`read`。`readlinebuf`这个新函数能够展露内部缓冲区的状态，便于调用者查看在当前文本行之后是否收到了新的数据。

> 但是，在`readline.c`中使用静态变量实现跨相继函数调用的状态信息维护，其结果是这些函数变得不可重入或者说非线程安全了。我们将在11.18节和26.5节中讨论这一点。在后边我们将使用特定于线程的数据开发一个线程安全的版本。



# END











