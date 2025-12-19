# 第二部分 基本套接字编程

# 第五章 TCP 客户/服务程序示例



## 5.1 概述

我们将在本章使用前一章中介绍的基本函数编写一个完整的TCP客户/服务器程序示例。这个简单的例子是执行如下步骤的一个回射服务器：
（1）客户从标准输入读入一行文本，并写给服务器；
（2）服务器从网络输入读入这行文本，并回射给客户；
（3）客户从网络输入读入这行回射文本，并显示在标准输出上。

客户与服务器之间实际上构成一个全双工的TCP连接。fgets和fputs这两个函数来自标准I/O函数库，writen和readline这两个函数详见3.9节。尽管我们将开发自己的回射服务器实现，大多数TCP/IP实现却已经提供了这样的服务器，既有使用TCP的，又有使用UDP的（2.12节）。我们还将与自已的客户一道使用这些服务器。回射输入行这样一个客户/服务器程序是一个尽管简单然而有效的网络应用程序例子。实现任何客户/服务器网络应用所需的所有基本步骤可通过本例子阐明。若想把本例子扩充成你自己的应用程序，你只需修改服务器对来自客户的输入的处理过程。

除了以正常的方式运行本例子的客户和服务器（即键入一行文本并观察它的回射）之外，我们还会探讨它的许多边界条件：客户和服务器启动时发生什么？客户正常终止时发生什么？若服务器进程在客户之前终止，则客户会发生什么？若服务器主机崩溃，则客户发生什么？如此等等。通过观察这些情形，弄清在网络层次发生什么以及它们如何反映到套接字API，我们将更多地理解这些层次的工作原理，并体会如何编写应用程序代码来处理这些情形。

在所有这些例子中，我们把诸如地址和端口之类特定于协议的常值硬编写到代码中。这么做有两个原因：一是我们必须确切了解在特定于协议的地址结构中应存放什么内容：二是我们尚未讨论到可以使得代码更便于移植的库函数，这些库函数将在第11章中讨论。我们现在就留意，随着学习越来越多的网络编程知识，我们将在后续各章中多次修改本章的客户和服务器程序。



## 5.2 TCP 回射服务器程序：main 函数

下面是书中TCP服务的并发程序的main函数。tcpcliserv01.c：

~~~c
#include	"unp.h"

int
main(int argc, char **argv)
{
	int					listenfd, connfd;
	pid_t				childpid;
	socklen_t			clilen;
	struct sockaddr_in	cliaddr, servaddr;

	listenfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family      = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port        = htons(SERV_PORT);

	Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

	Listen(listenfd, LISTENQ);

	for ( ; ; ) {
		clilen = sizeof(cliaddr);
		connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);

		if ( (childpid = Fork()) == 0) {	/* child process */
			Close(listenfd);	/* close listening socket */
			str_echo(connfd);	/* process the request */
			exit(0);
		}
		Close(connfd);			/* parent closes connected socket */
	}
}

~~~

* **创建套接字，捆绑服务器的端口**

  > 11 - 20：创建一个TCP套接字。在待捆绑到该TCP套接字的网际网套接字地址结构中填入通配地址（`INADDR_ANY`）和服务器的众所周知端口（`SERV_PORT`，在头文件`unp.h`中其值定义为`9877`)。捆绑通配地址是在告知系统：要是系统是多宿主机，我们将接受目的地址为任何本地接口的连接。TCP端口号的选择应该比1023大（我们不需要一个保留端口），比5000大（以免与许多源自Berkeley的实现分配临时端口的范围冲突)，比49152小（以免与临时端口号的“正确”范围冲突)，而且不应该与任何已注册的端口冲突。`listen`把该套接字转换成一个监听套接字。

* **等待完成客户连接**

  > 23  - 24：服务器阻塞于`accept`调用，等待客户连接完成。

* **并发服务器**

  > 26 - 31：`fork`为每个客户派生一个处理它们的子进程。正如我们在4.8节讨论的那样，子进程关闭监听套接字，父进程关闭已连接套接字。子进程接着调用`str_echo`处理客户。



## 5.3 TCP回射服务器程序：str_echo 函数

`str_echo`函数执行处理每个客户的服务：从客户读入数据，并把它们回射给客户。

~~~c
#include	"unp.h"

void
str_echo(int sockfd)
{
	ssize_t		n;
	char		buf[MAXLINE];

again:
	while ( (n = read(sockfd, buf, MAXLINE)) > 0)
		Writen(sockfd, buf, n);

	if (n < 0 && errno == EINTR)
		goto again;
	else if (n < 0)
		err_sys("str_echo: read error");
}

~~~

* 读入缓冲区并回射其中内容

  > 10 - 11：`read`函数从套接字读入数据，`writen`函数把其中内容回射给客户。如果客户关闭连接（这是正常情况)，那么接收到客户的`FIN`将导致服务器子进程的`read`函数返回0，这又导致`str_echo`函数的返回，从而中终止子进程。



## 5.4 TCP 回射客户程序：main 函数

下面为TCP客户端的mian程序：tcpcli01.c：

~~~c
#include	"unp.h"

int
main(int argc, char **argv)
{
	int					sockfd;
	struct sockaddr_in	servaddr;

	if (argc != 2)
		err_quit("usage: tcpcli <IPaddress>");

	sockfd = Socket(AF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(SERV_PORT);
	Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);

	Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));

	str_cli(stdin, sockfd);		/* do it all */

	exit(0);
}
~~~

* **创建套接字，装填网际网套接字地址结构**

  > 12 - 17：创建一个TCP套接字，用服务器的IP地址和端口号装填一个网际网套接字地址结构。我们可从命令行参数取得服务器的IP地址，从头文件`unp.h`取得服务器的众所周知端口号（`SERV_PORT`)。

* **连接到服务器**

  > 19 - 21：`connect`建立与服务器的连接。`str_cli`函数完成剩余部分的客户处理工作。



## 5.5 TCP h回射客户程序：str_cli 函数

`str_cli`函数完成客户处理循环：从标准输入读入一行文本，写到服务器上，读回服务器对该行的回射，并把回射行写到标准输出上。

~~~c
#include	"unp.h"

void
str_cli(FILE *fp, int sockfd)
{
	char	sendline[MAXLINE], recvline[MAXLINE];

	while (Fgets(sendline, MAXLINE, fp) != NULL) {

		Writen(sockfd, sendline, strlen(sendline));

		if (Readline(sockfd, recvline, MAXLINE) == 0)
			err_quit("str_cli: server terminated prematurely");

		Fputs(recvline, stdout);
	}
}

~~~

* 读入一行，写到服务器

  > 8 - 10：`fgets`读入一行文本，`writen`把该行发送给服务器。

* 从服务器读入回射行，写到标准输出

  > 12 - 15：`readline`从服务器读入回射行，`````fputs`````把它写到标准输出。

* 结束



## 5.6 正常启动

尽管我们的TCP程序例子很小（两个`main`函数加`str_echo`、`str_cli`、`readline`和`writen`,总共约150行代码)，然而对于我们弄清客户和服务器如何启动，如何终止，更为重要的是当发生某些错误（例如客户主机崩溃、客户进程崩溃、网络连接断开，等等）时将会发生什么，本例子却至关重要。只有搞清这些边界条件以及它们与TCP/IP协议的相互作用，我们才能写出能够处理这些情况的健壮的客户和服务器程序。

【后续我将参考本书的接口写一个较新的linux版的样例（虽然本书的例程可以在linux上成功编译运行）】

一下是书中描述内容，未经过验证：

首先，我们在主机linux上后台启动服务器。

~~~
linux % tcpserv01 &
[1]	17870
~~~

服务器启动后，它调用`socket`、`bind`、`listen`和`accept`，并阻塞于`accept`调用。（我们还没有启动客户）。在启动客户之前，我们运行`netstat`程序来检查服务器监听套接字的状态。

~~~(空)
linux % netatat -a
Active Internet connections (servers and established)
Proto 	Recv-Q 	Send-Q 	Local Address 	Foreign Address 	State
tcp			0		0	*:9877			*:*					LISTEN
~~~

我们这里只给出了输出的第一行（标题）以及我们最关心的那一行。本命令列出系统中所有套接字的状态，可能有大量输出。我们必须指定`-a`标志以查看监听套接字。这个输出正是我们所期望的：有一个套接字处于`LISTEN`状态，它有通配的本地IP地址，本地端口为`9877`。`netstat`用星号“`*`”来表示一个为0的IP地址（`INADDR_ANY`，通配地址）或为0的端口号。我们接着在同一个主机上启动客户，并指定服务器主机的IP地址为`127.0.0.1`（环回地址）。当然我们也可以指定该地址为该主机的普通（非环回）IP地址。

~~~
linux % tcpcli01 127.0.0.1
~~~

客户调用`socket`和`connect`，后者引起TCP的三路握手完成后，客户中的`connect`和服务中的`accept`均返回，链接于是建立。接着发生步骤如下：

（1）客户调用`str_cli`函数，该函数将阻塞于`fgets`调用，因为我们还未曾键入过一行文本。

（2）当服务器中的`accept`返回时，服务器调用fork，再由子进程调用`str_echo`。该函数调用`readline`，`readline`调用`read`，而`read`在等待客户送入一行文本期间阻塞。

（3）另一方面，服务器父进程再次调用`accept`并阻塞，等待下一个客户连接。至此，我们有3个都在睡眠（即已阻塞）的进程：客户进程、服务器父进程和服务器子进程。

> 当三路握手完成时，我们特意首先列出客户的步骤，接着列出服务器的步骤。原因：客户接收到三路握手的第二个分节时，`connect`返回，而服务器要直到接收到三路握手的第三个分节才返回，即在`connect`返回之后再过一半`RTT`才返回。

我们特意在同一个主机上运行客户和服务器，因为这是试验客户/服务器应用程序的最简单方法。既然我们是在同一个主机上运行客户和服务器，`netstat`给出了对应所建立TCP连接的两行额外的输出。

~~~(空)
linux % netetat -a
Active Internet connections (servers and established)
Proto 	Recv-Q 	Send-Q 	Local Address 		Foreign Address 	State
tcp			0		0 	localhost:9877		localhost:42758		ESTABLISHED
tcp			0		0 	localhost:42758		localhost:9877		ESTABLISHED
tcp			0		0 	*:9877				*：*					LISTEN
~~~

第一个`ESTABLISHED`行对应于服务器子进程的套接字，因为它的本地端口号是`9877`；第二个`ESTABLISHED`行对应于客户进程的套接字，因为它的本地端口号是`42758`。要是我们在不同的主机上运行客户和服务器，那么客户主机就只输出客户进程的套接字，服务器主机也只输出两个服务器进程（一个父进程一个子进程）的套接字。我们也可用ps命令来检查这些进程的状态和关系。

~~~(空)
linux & ps -t pte/6 -o pid,ppid,tty,stat,arge,wchan
PID		PPID 	TT		STAT 	COMMAND				WCHAN
22038 	22036 	pts/6	S		-bash				wait4
17870 	22038 	pts/6	S		./tcpserv01			wait_for_connect
19315 	17870 	pts/6	S		./tcpserv01			tcp_data_wait
19314 	22038 	pts/6	S		./tcpcli01 127.0 	read_chan
~~~

（我们已使用ps相当特定的命令行参数限定它只输出与本讨论相关的信息。）从输出中可见，客户和服务器运行在同一个窗口中（即`pts/6`，表示伪终端号6)。PID和PPID列给出了进程间的父子关系。由于子进程的PPID是父进程的PID，我们可以看出，第一个`tcpserv01`行是父进程，第二个`tcpserv01`行是子进程。而父进程的PPID是shell（bash)。我们所有三个网络进程的STAT列都是“`S`”，表明进程在为等待某些资源而睡眠。进程处于睡眠状态时`WCHAN`列指出相应的条件。Linux在进程阻塞于`accept`或`connect`时，输出`wait_for_connect`；在进程阻塞于套接字输入或输出时，输出`tcp_data_wait`；在进程阻塞于终端`I/O`时，输出`read_chan`。这里我们的三个网络进程的`WCHAN`值所表示的意义一目了然。



## 5.7 正常终止

至此连接已经建立，不论我们在客户端的标准输入中键入什么，都会回射到它的标准输出中。

~~~(空)
linux & tcpcli01 127.0.0.1			我们已经给出过本行
hello,world							现在键入这一行
hello,world							这一行被回射回来
good bye
good bye
^D									<CtrI+D>是我们的终端EOF字符
~~~

我们键入两行，每行都得到回射，我们接着键入终端`EOF`字符（`Control-D`）以终止客户。此时如果立即执行`netstat`命令，我们将看到如下结果：

~~~(空)
linux & netstat -a l grep 9877
tcp		0		0 	*:9877				*:*				LISTEN
tcp		0		0 	localhost:42758		localhost:9877	TIME WAIT
~~~

当前连接的客户端（它的本地端口号为42758）进入了`TIME_WAIT`状态（2.7节)，而监听服务器仍在等待另一个客户连接。（这回我们让命令`netstat`的输出通过管道作为`grep`的输入，从而只输出与服务器的众所周知端口相关的文本行。这样做也删掉了标题行。）

我们可以总结出正常终止客户和服务器的步骤。
（1）当我们键入`EOF`字符时，`fgets`返回一个空指针，于是`str_cli`函数返回。

（2）当`str_cli`返回到客户的`main`函数时，`main`通过调用`exit`终止。

（3）进程终止处理的部分工作是关闭所有打开的描述符，因此客户打开的套接字由内核关闭。这导致客户TCP发送一个`FIN`给服务器，服务器TCP则以`ACK`响应，这就是TCP连接终止序列的前半部分。至此，服务器套接字处于`CLOSEWAIT`状态，客户套接字则处于`FIN_WAIT_2`状态。

（4）当服务器TCP接收`FIN`时，服务器子进程阻塞于`readline`调用，于是`readline`返回0。这导致`str_echo`函数返回服务器子进程的`main`函数。

（5）服务器子进程通过调用`exit`来终止。

（6）服务器子进程中打开的所有描述符随之关闭。由子进程来关闭已连接套接字会引发TCP连接终止序列的最后两个分节：一个从服务器到客户的`FIN`和一个从客户到服务器的`ACK`。至此，连接完全终止，客户套接字进入`TIME_WAIT`状态。

（7）进程终止处理的另一部分内容是：在服务器子进程终止时，给父进程发送一个`SIGCHLD`信号。这一点在本例中发生了，但是我们没有在代码中捕获该信号，而该信号的默认行为是被忽略。既然父进程未加处理，子进程于是进入僵死状态。我们可以使用`ps`命令验证这一点。

~~~(空)
linux & ps -t pts/6 -o pid,ppid,tty.stat,arge,wchan
PID		PPID 	TT		STAT 	COMMAND				WCHAN
22038 	22036 	pts/6	S		-bash				read chan
17870 	22038 	pts/6	S		./tcpserv01			wait_for_connect
19315 	17870 	pts/6	Z		[tcpserv0l <defu 	do_exit
~~~

子进程的状态现在是`Z`（表示僵死）。我们必须清理僵死进程，这就涉及Unix信号的处理。我们将在下一节概述信号处理，在下一节继续我们的例子。



## 5.8 POSIX 信号处理

信号（`signal`）就是告知某个进程发生了某个事件的通知，有时也称为软件中断（software interrupt)。信号通常是异步发生的，也就是说进程预先不知道信号的准确发生时刻。信号可以：

* 由一个进程发给另一个进程（或自身）；
* 由内核发给某个进程。

上一节结尾提到的`SIGCHLD`信号就是由内核在任何一个进程终止时发给它的父进程的一个信号。每个信号都有一个与之关联的处置（disposition)，也称为行为（action)。我们通过调用`sigaction`函数（稍后讨论）来设定一个信号的处置，并有三种选择。







































