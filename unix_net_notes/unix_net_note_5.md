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

（1）我们可以提供一个函数，只要有特定信号发生它就被调用。这样的函数称为信号处理函数（signal handler)，这种行为称为捕获（catching)信号。有两个信号不能被捕获，它们是SIGKILL和SIGSTOP。信号处理函数由信号值这个单一的整数参数来调用，且没有返回值，其函数原型因此如下：

~~~c
void int_handler(int signo)
~~~

对于大多数信号来说，调用`sigaction`函数并指定信号发生时所调用的函数就是捕获信号所需做的全部工作。不过我们稍后将看到，SIGIO、SIGPOLL和SIGURG这些个别信号还要求捕获它们的进程做些额外工作。

(2)我们可以把某个信号的处置设定为SIG_IGN来忽略（ignore）它。SIGKILL和SIGSTOP这两个信号不能被忽略。

(3)我们可以把某个信号的处置设定为SIG_DFL来启用它的默认（default）处置。默认处置通常是在收到信号后终止进程，其中某些信号还在当前工作目录产生一个进程的核心映像（core image，也称为内存影像）。另有个别信号的默认处置是忽略，SIGCHLD和SIGURG（带外数据到达时发送，见第24章）就是本书中出现的默认处置为忽略的两个信号。



### signal 函数

建立信号处置的POSIX方法就是调用sigaction函数。不会这有点复杂，因为该函数的参数之一是我们必须分配并填写的结构。简单些的方法就是调用signal函数，其第一个参数是信号名，第二个参数或为指向函数的指针，或为常值SIG_IGN或SIG_DFL。然而signal是早于POSIX出现的历史悠久的函数。调用它时，不同的实现提供不同的信号语义以达成向后兼容，而POSIX则明确规定了调用sigaction时的信号语义。我们的解决办法时定义自己的signal函数，它只是调用POSIX的sigaction函数。这就以所期望的POSIX语义提供了一个简单那的接口。我们把该函数以及早先讲过的`err_XXX`函数和包裹函数等一道包含在自己的函数库中，而这个函数库在我们构造这本书中的程序时指定。

~~~c
/* include signal */
#include	"unp.h"

Sigfunc *
signal(int signo, Sigfunc *func)
{
	struct sigaction	act, oact;

	act.sa_handler = func;
	sigemptyset(&act.sa_mask);
	act.sa_flags = 0;
	if (signo == SIGALRM) {
#ifdef	SA_INTERRUPT
		act.sa_flags |= SA_INTERRUPT;	/* SunOS 4.x */
#endif
	} else {
#ifdef	SA_RESTART
		act.sa_flags |= SA_RESTART;		/* SVR4, 44BSD */
#endif
	}
	if (sigaction(signo, &act, &oact) < 0)
		return(SIG_ERR);
	return(oact.sa_handler);
}
/* end signal */

Sigfunc *
Signal(int signo, Sigfunc *func)	/* for our signal() function */
{
	Sigfunc	*sigfunc;

	if ( (sigfunc = signal(signo, func)) == SIG_ERR)
		err_sys("signal error");
	return(sigfunc);
}
~~~

**用typedef简化函数原型**：函数signal的正常函数原型因层次太多而变得很复杂：

~~~c
void (*signal(int signo, void (*func)(int)))(int);
// 为了简化起见，我们在头文件unp.h中定义了如下的Sigfunc原型：
typedef void Sigfunc(int);
// 它说明信号处理函数是仅有一个整数且不返回值的函数。signal的函数原型于是变为：
Sigfunc *signal(int signo, Sigfunc *func);
// 该函数的第二个参数和返回值都是指向信号处理函数指针。
~~~

现代的Linux系统中已经有了标准化的实现，我们进行调用即可，部分说明如下：

~~~c
void (*signal(int signum, void (*fun)(int)))(int);
SYNOPSIS
       #include <signal.h>

       typedef void (*sighandler_t)(int);

       sighandler_t signal(int signum, sighandler_t handler);
~~~

**设置处理函数**：sigaction结构的sa_handler成员被置为**func**函数。

**设置处理函数的信号掩码**：POSIX允许我们指定这样一组信号，它们在信号处理函数被调用时阻塞。任何阻塞的信号都不能递交（delivering）给进程。我们把`sa_mask`成员设置为空集，意味着在该信号处理函数运行期间，不阻塞额外的信号。POSIX保证被捕获的信号在其信号处理函数运行期间总是阻塞的。

**设置SA_RESTART标志**：`SA_RESTART`标志是可选的。如果设置，由相应信号中断的系统调用将由内核自动重启。（我们将在下一节继续上一节的例子时详细讨论被中断的系统调用。）如果被捕获的信号不是`SIGALRM`且`SA_RESTART`有定义，我们就设置该标志。（对`SIGALRM`进行特殊处理的原因在于：产生该信号的目的正如14.2节将讨论的那样，通常是为`I/O`操作设置超时，这种情况下我们希望受阻塞的系统调用被该信号中断掉。）一些较早期的系统（如SunOS4.x）默认设置成自动重启被中断的系统调用，并定义了与`SA_RESTART`互补的`SA_INTERRUPT`标志。如果定义了该标志，我们就在被捕获的信号是`SIGALRM`时设置它。

**调用sigaction函数**：调用sigaction函数，并将相应信号的旧行为作为signal函数的返回值。



### POSIX 信号语义

我们把符合POSIX的系统上的信号处理总结为一下几点。

* 一旦安装了信号处理函数，它便一直安装着（较早期的系统是每执行一次就将其拆除)。
* 在一个信号处理函数运行期间，正被递交的信号是阻塞的。而且，安装处理函数时在传递给sigaction函数的`sa_mask`信号集中指定的任何额外信号也被阻塞。如果我们将`sa_mask`置为空集，意味着除了被捕获的信号外，没有额外信号被阻塞。
* 如果一个信号在被阻塞期间产生了一次或多次，那么该信号被解阻塞之后通常只递交一次，也就是说Unix信号默认是不排队的。我们将在下一节查看这样的一个例子。POSIX 实时标准 `1003.1b` 定义了一下排队的可靠信号，不过书中我们不使用。
* 利用sigprocmask函数选择性地阻塞或解阻塞一组信号是可能的。这使得我们可以做到在一段临界区代码执行期间，防止捕获某些信号，以此保护这段代码。



## 5.9 处理 SIGCHLD 信号

设置僵死（zombie）状态的目的是维护子进程的信息，以便父进程在以后某个时候获取。这些信息包括子进程的进程ID、终止状态以及资源利用信息（CPU时间、内存使用量等等）。如果·个进程终止，而该进程有子进程处于僵死状态，那么它的所有僵死子进程的父进程ID将被重置为l（init进程)。继承这些子进程的init进程将清理它们（也就是说init进程将wait它们，从而去除它们的僵死状态）。有些Unix系统在ps命令输出的`COMMAND`栏以`<defunct>`指明僵死进程。

### 处理僵死进程

我们显然不愿意留存僵死进程。它们占用内核中的空间，最终可能导致我们耗尽进程资源。无论何时我们`fork`子进程都得`wait`它们，以防它们变成僵死进程。为此我们建立一个俘获`SIGCHLD`信号的信号处理函数，在函数体中我们调用`wait`。（我们将在5.10节介绍`wait`和`waitpid`函数。）通过在图5-2所示代码的`listen`调用之后增加如下函数调用：

~~~c
Signal(SIGCHLD, sig_chld);
~~~

我们就建立了该信号处理函数。（这必须在fork第一个子进程之前完成，且只做一次。）我们接着定义名为`sig_chld`的这个信号处理函数。

~~~c
#include	"unp.h"

void
sig_chld(int signo)
{
	pid_t	pid;
	int		stat;

	while ( (pid = waitpid(-1, &stat, WNOHANG)) > 0) {
		printf("child %d terminated\n", pid);
	}
	return;
}

~~~

这个例子是为了说明，在编写捕获信号的网络程序时，我们必须认清被中断的系统调用且处理它们。在这个运行在Solaris9环境下特定例子中，标准C函数库中提供的`signal`函数不会使内核自动重启被中断的系统调用。也就是说，我们设置的`SA_RESTART`标志在系统函数库的signal函数中并没有设置。另有些系统自动重启被中断的系统调用。如果我们在4.4BSD环境下照样使用系统函数库版本的`signal`函数运行上述例子，那么内核将重启被中断的系统调用，于是`accept`不会返回错误。我们定义自己的`signal`函数并在贯穿全书使用的理由之一就是应对不同操作系统之间的这个潜在问题。

作为本书使用的编程约定之一，我们总是在信号处理函数中显式给出`return`语句，即使对于返回值类型为`void`的信号处理函数而言，从函数结尾处掉出和执行`return`语句效果是一样的，我们也还是为其使用`return`语句。这么一来，当某个系统调用被我们编写的某个信号处理函数中断时，我们就可以得知该系统调用具体是被哪个信号处理函数的哪个`return`语句中断的。

### 处理被中断的系统调用

我们用术语慢系统调用（slow system call）描述过`accept`函数，该术语也适用于那些可能永远阻塞的系统调用。永远阻塞的系统调用是指调用有可能永远无法返回，多数网络支持函数都属于这一类。举例来说，如果没有客户连接到服务器上，那么服务器的`accept`调用就没有返回的保证。类似地，在图5-3中，如果客户从未发送过一行要求服务器回射的文本，那么服务器的`read`调用将永不返回。其他慢系统调用的例子是对管道和终端设备的读和写。一个值得注意的例外是磁盘`I/O`，它们一般都会返回到调用者（假设没有灾难性的硬件故障）。

适用于慢系统调用的基本规则是：当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个`EINTR`错误。有些内核自动重启某些被中断的系统调用。不过为了便于移植，当我们编写捕获信号的程序时（多数并发服务器捕获`SIGCHLD`)，我们必须对慢系统调用返回`EINTR`有所准备。移植性问题是由早期使用的修饰词“可能”、“有些”和对POSIX的`SA_RESTART`标志的支持是可选的这一事实造成的。即使某个实现支持`SA_RESTART`标志，也并非所有被中断系统调用都可以自动重启。举例来说，大多数源自Berkeley的实现从不自动重启`select`，其中有些实现从不重启`accept`和`recvfrom`。为了处理被中断的`accept`，我们对`accept`的调用从`for`循环开始改起，如下所示：

~~~c
	for ( ; ; ) {
		clilen = sizeof(cliaddr);
		if ( (connfd = accept(listenfd, (SA *) &cliaddr, &clilen)) < 0) {
			if (errno == EINTR)
				continue;		/* back to for() */
			else
				err_sys("accept error");
		}

		if ( (childpid = Fork()) == 0) {	/* child process */
			Close(listenfd);	/* close listening socket */
			str_echo(connfd);	/* process the request */
			exit(0);
		}
		Close(connfd);			/* parent closes connected socket */
	}
~~~

注意，我们调用的是`accept`函数本身而不是它的包裹函数`Accept`，因为我们必须自己处理该函数的失败情况。

这段代码所做的事情就是自已重启被中断的系统调用。对于`accept`以及诸如`read`、`write`、`select`和`open`之类函数来说，这是合适的。不过有一个函数我们不能重启：`connect`。如果该函数返回`EINTR`，我们就不能再次调用它，否则将立即返回一个错误。当`connect`被一个捕获的信号中断而且不自动重启时，我们必须调用`select`来等待连接完成。



## 5.10 wait 和 waitpid 函数

Linux中的wait和waitpid函数：

~~~c
#include <sys/wait.h>

pid_t wait(int *wstatus);

pid_t waitpid(pid_t pid, int *wstatus, int options);

int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
/* This is the glibc and POSIX interface; see
   NOTES for information on the raw system call. */
~~~

### wait

~~~c
#include <sys/wait.h>
pid_t wait(int *wstatus);
~~~

**阻塞等待任意一个子进程结束**，并返回该子进程的 PID。

| 参数      | 含义                          |
| --------- | ----------------------------- |
| `wstatus` | 子进程的退出状态（可为 NULL） |

| 返回值 | 含义               |
| ------ | ------------------ |
| `> 0`  | 已结束子进程的 PID |
| `-1`   | 出错（如无子进程） |

~~~c
int status;
pid_t pid = wait(&status);

if (WIFEXITED(status)) {
    printf("exit code=%d\n", WEXITSTATUS(status));
}

~~~

**特点 & 限制**

* 只能等 **任意子进程**
* **一定阻塞**
* 不能和 `SIGCHLD` 灵活配合

现在基本只用于**教学或非常简单程序**。

### waitpid

~~~c
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *wstatus, int options);
~~~

**等待指定子进程（或一组子进程）的状态变化。**

**参数：**

① `pid`（核心）

| pid 值 | 含义                            |
| ------ | ------------------------------- |
| `> 0`  | 等待指定 PID                    |
| `-1`   | 等待任意子进程（等价 `wait()`） |
| `0`    | 等待同进程组的子进程            |
| `< -1` | 等待指定进程组                  |

------

② `wstatus`

和 `wait()` 一样，存放状态。

------

③ `options`（重点）

| 选项         | 含义                   |
| ------------ | ---------------------- |
| `0`          | 阻塞                   |
| `WNOHANG`    | **非阻塞**（立即返回） |
| `WUNTRACED`  | 监听 stop              |
| `WCONTINUED` | 监听 continue          |

----

**返回值：**

| 返回值 | 含义                    |
| ------ | ----------------------- |
| `> 0`  | 子进程 PID              |
| `0`    | 使用 `WNOHANG` 且无变化 |
| `-1`   | 出错                    |

~~~c
void sig_chld(int signo)
{
    int status;
    pid_t pid;

    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        // 回收子进程
    }
}

~~~

工程中 90% 用这个，因为：

 **可选阻塞**、可精确控制、**与 `SIGCHLD` 完美配合**、**不会卡死信号处理函数**。

### waitid

~~~c
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t id,
           siginfo_t *infop, int options);
~~~

**等待子进程状态变化，并以 `siginfo_t` 结构返回详细信息。**

**参数：**

① `idtype + id`（比 waitpid 更精细）

| idtype   | id 含义      |
| -------- | ------------ |
| `P_PID`  | 等待指定 PID |
| `P_PGID` | 等待进程组   |
| `P_ALL`  | 所有子进程   |

------

② `infop`（重点差异）

```c
siginfo_t info;
```

返回内容包括：

| 字段        | 含义          |
| ----------- | ------------- |
| `si_pid`    | 子进程 PID    |
| `si_code`   | 退出原因      |
| `si_status` | 退出码 / 信号 |
| `si_uid`    | 子进程用户    |

------

③ `options`

| 选项         | 含义             |
| ------------ | ---------------- |
| `WEXITED`    | 等待退出         |
| `WSTOPPED`   | 等待 stop        |
| `WCONTINUED` | 等待 continue    |
| `WNOHANG`    | 非阻塞           |
| `WNOWAIT`    | 不回收（只查询） |

------

**返回值：**

| 返回值 | 含义       |
| ------ | ---------- |
| `0`    | 成功       |
| `-1`   | 失败返回值 |

~~~c
siginfo_t info;

waitid(P_ALL, 0, &info, WEXITED | WNOHANG);

printf("pid=%d exit=%d\n",
       info.si_pid, info.si_status);
~~~

因为：**接口复杂、不直观、老代码兼容性差**，很少使用。在**glibc、内核工具、系统服务**更常用。

-------

### 对比

| 特性         | wait  | waitpid | waitid      |
| ------------ | ----- | ------- | ----------- |
| 等待对象     | 任意  | 精确    | 最精细      |
| 是否阻塞     | 是    | 可选    | 可选        |
| 状态返回     | `int` | `int`   | `siginfo_t` |
| SIGCHLD 推荐 | ❌     | ✅       | ⚠️           |
| 工程实用性   | 低    | ⭐⭐⭐⭐⭐   | 中          |

`wait()` 是历史接口；`waitpid()` 是工程主力；`waitid()` 是底层与高级控制接口。实际写守护进程、服务器、`SIGCHLD` 处理时，优先使用 `waitpid(-1, &status, WNOHANG)`。



## 5.11 accept 返回前连接中止

类似于前一节中介绍的被中断系统调用的例子，另有一种情形也能够导致`accept`返回一个非致命的错误，在这种情况下，只需要再次调用`accept`。分组序列在较忙的服务器（典型的是较忙的Web服务器）上已出现过。

这里，三路握手完成从而连接建立之后，客户TCP却发送了一个`RST`（复位)。在服务器端看来，就在该连接已由TCP排队，等着服务器进程调用`accept`的时候`RST`到达。稍后，服务器进程调用`accept`。

但是，如何处理这种中止的连接依赖于不同的实现。源自Berkeley的实现完全在内核中处理中止的连接，服务器进程根本看不到。然而大多数SVR4实现返回一个错误给服务器进程，作为`accept`的返回结果，不过错误本身取决于实现。这些SVR4实现返回一个`EPROTO`（“protocol error”，协议错误)`errno`值，而POSIX指出返回的`errno`值必须是ECONNABORTED(“sofware caused connectionabort”，软件引起的连接中止）。POsIx作出修改的理由在于：流子系统（streams subsystem）中发生某些致命的协议相关事件时，也会返回`EPROTO`。要是对于由客户引起的一个已建立连接的非致命中止也返回同样的错误，那么服务器就不知道该再次调用`accept`还是不该了。换成`ECONNABORTED`错误，服务器就可以忽略它，再次调用`accept`就行。



## 5.12 服务器进程终止

现在启动我们的客户/服务器对，然后杀死服务器子进程。这是在模拟服务器进程崩溃的情形，我们可从中查看客户将发生什么。（我们必须小心区别即将讨论的服务器进程崩溃与将在5.14节讨论的服务器主机崩溃。）所发生的步骤如下所述。

（1）我们在同一个主机上启动服务器和客户，并在客户上键入一行文本，以验证一切正常。正常情况下该行文本由服务器子进程回射给客户。

（2）找到服务器子进程的进程ID，并执行kill命令杀死它。作为进程终止处理的部分工作，子进程中所有打开着的描述符都被关闭。这就导致向客户发送一个`FIN`，而客户TCP则响应以一个`ACK`。这就是TCP连接终止工作的前半部分。

（3）`SIGCHLD`信号被发送给服务器父进程，并得到正确处理。

（4）客户上没有发生任何特殊之事。客户TCP接收来自服务器TCP的`FIN`并响应以一个`ACK`，然而问题是客户进程阻塞在fgets调用上，等待从终端接收一行文本。

（5）此时，在另外一个窗口上运行`netstat`命令，以观察套接字的状态。

~~~c
linux & netstat -a l grep 9877
tcp		0		0 	*:9877					*:*					LISTEN
tcp		0		0 	localhost:9877			localhost:43604		FIN_WAIT2
tcp		1		0 	localhost:43604			localhost:9877		CLOSE WAIT
~~~

参照上表，可以看到TCP连接终止序列的前半部分已经完成。

（6）我们可以在客户上再键入一行文本。以下是从第一步开始发生在客户之事：

~~~(空)
lirux & tcpcli01 127.0.0.1			启动客户
hello								键入第一行文本
hello								它被正确回射
									在这儿杀死服务器子进程
another line						然后键入下一行文本
str_cli: server terminatedprematurely
~~~

当我们键入“anotherline”时，`str_cli`调用`writen`，客户TCP接着把数据发送给服务器。TCP允许这么做，因为客户TCP接收到`FIN`只是表示服务器进程已关闭了连接的服务器端，从而不再往其中发送任何数据而已。`FIN`的接收并没有告知客户TCP服务器进程已经终止（本例子中它确实是终止了)。在6.6节讨论TCP的半关闭时我们将再次论述这一点。当服务器TCP接收到来自客户的数据时，既然先前打开那个套接字的进程已经终止，于是响应以一个`RST`。通过使用`tcpdump`来观察分组，我们可以验证该`RST`确实发送了。

（7）然而客户进程看不到这个`RST`，因为它在调用`writen`后立即调用`readline`，并且由于第2步中接收的`FIN`，所调用的`readline`立即返回 0（表示`EOF`)。我们的客户此时并未预期收到`EOF`，于是以出错信息“server terminated prematurely”（服务器过早终止）退出。

（8）当客户终止时（通过调用err_quit`)，它所有打开着的描述符都被关闭。

本例子的问题在于：当FIN到达套接字时，客户正阻塞在`fgets`调用上。客户实际上在应对两个描述符——套接字和用户输入，它不能单纯阻塞在这两个源中某个特定源的输入上（正如目前编写的`str_cl`i函数所为），而是应该阻塞在其中任何一个源的输入上。事实上这正是`select`和`poll`这两个函数的目的之一，我们将在第6章中讨论它们。我们在6.4节重新编写`str_cli`函数之后，一旦杀死服务器子进程，客户就会立即被告知已收到`FIN`。



## 5.13 SIGPIPE 信号

要是客户不理会`readline`函数返回的错误，反而写入更多的数据得到服务器上，那又会发生什么呢？这种情况是可能发生的，举例来说，客户可能在读回任何数据之前执行两次针对服务器的写操作，而`RST`是由其中第一次写操作引发的。适用于此的规则是：当一个进程向某个已收到`RST`的套接字执行写操作时，内核向该进程发送一个`SIGPIPE`信号。该信号的默认行为是终止进程，因此进程必须捕获它以免不情愿地被终止。不论该进程是捕获了该信号并从其信号处理函数返回，还是简单地忽略该信号，写操作都将返回`EPIPE`错误。

为了看清有了SIGPIPE信号会发生什么，我们把客户程序修改成如下：

~~~c
#include	"unp.h"

void
str_cli(FILE *fp, int sockfd)
{
	char	sendline[MAXLINE], recvline[MAXLINE];

	while (Fgets(sendline, MAXLINE, fp) != NULL) {

		Writen(sockfd, sendline, strlen(sendline));
        sleep(1);
		Writen(sockfd, sendline+1, strlen(sendline)-1);
		if (Readline(sockfd, recvline, MAXLINE) == 0)
			err_quit("str_cli: server terminated prematurely");

		Fputs(recvline, stdout);
	}
}

~~~

**10 - 12** ：我们所做的修改就是调用`writen`两次：第一次把文本行数据的第一个字节写入套接字，暂停一秒钟后，第二次把同一文本行中剩余字节写入套接字。目的是让第一次`writen`引发一个`RST`，再让第二个`writen`产生`SIGPIPE`。在我们的Linux主机上运行客户，我们得到如下结果：

~~~(空)
linux % tcpcli11 127.0.0.1
hi there					我们键入这行文本
hi there					它被服务器回射回来
							在这儿杀死服务器子进程
bye							然后键入这行文本
Broken pipe					本行由shell显示
~~~

我们启动客户，键入一行文本，看到它被正确回射后，在服务器主机上终止服务器子进程。我们接着键入另一行文本（“bye”)，结果是没有任何回射，而shell告诉我们客户进程因为`SIGPIPE`信号而死亡了。当前台进程未曾执行内存内容倾泻（core dumping）就死亡时，有些shell不显示任何信息，不过我们用于本例子的shell即bash却告知我们欲知的信息。处理`SIGPIPE`的建议方法取决于它发生时应用进程想做什么。如果没有特殊的事情要做，那么将信号处理办法直接设置为`SIG_IGN`，并假设后续的输出操作将捕捉EPIPE错误并终止。如果信号出现时需采取特殊措施（可能需在日志文件中登记)，那么就必须捕获该信号，以便在信号处理函数中执行所有期望的动作。但是必须意识到，如果使用了多个套接字，该信号的递交无法告诉我们是哪个套接字出的错。如果我们确实需要知道是哪个`write`出了错，那么必须要么不理会该信号，要么从信号处理函数返回后再处理来自`write`的`EPIPE`。



## 5.14 服务器主机崩溃

我们接着查看当服务器主机崩溃时会发生什么。为了模拟这种情形，我们必须在不同的主机上运行客户和服务器。我们先启动服务器，再启动客户，接着在客户上键入一行文本以确认连接工作正常，然后从网络上断开服务器主机，并在客户上键入另一行文本。这样同时也模拟了当客户发送数据时服务器主机不可达的情形（即建立连接后某些中间路由器不工作)。步骤如下所述:

（1）当服务器主机崩溃时，已有的网络连接上不发出任何东西。这里我们假设的是主机崩溃，而不是由操作员执行命令关机（我们将在5.16节讨论后者）。

（2）我们在客户上键入一行文本，它由`writen`写入内核，再由客户TCP作为一个数据分节送出。客户随后阻塞于`readline`调用，等待回射的应答。

（3）如果我们用`tcpdump`观察网络就会发现，客户TCP持续重传数据分节，试图从服务器上接收一个`ACK`。TCPv2的25.11节给出了TCP重传一个典型模式：源自Berkeley的实现重传该数据分节12次，共等待约9分钟才放弃重传。当客户TCP最后终于放弃时（假设在这段时间内，服务器主机没有重新启动，或者如果是服务器主机未崩溃但是从网络上不可达，那么假设主机仍然不可达），给客户进程返回一个错误。既然客户阻塞在`readline`调用上，该调用将返回一个错误。假设服务器主机已崩溃，从而对客户的数据分节根本没有响应，那么所返回的错误是`ETIMEDOUT`。然而如果某个中间路由器判定服务器主机已不可达，从而响应以一个“destination unreachable”（目的地不可达）`ICMP`消息，那么所返回的错误是`EHOSTUNREACH`或`ENETUNREACH`。尽管我们的客户最终还是会发现对端主机已崩溃或不可达，不过有时候我们需要比不得不等待9分钟更快地检测出这种情况。所用方法就是对`readline`调用设置一个超时，我们将在14.2节讨论这一点。

我们刚刚讨论的情形只有在我们向服务器主机发送数据时才能检测出它已经崩溃。如果我们不主动向它发送数据也想检测出服务器主机的崩溃，那么需要采用另外一个技术，也就是我们将在7.5节讨论的`SO_KEEPALIVE`套接字选项。



## 5.15 服务器主机崩溃后重启

在这种情形中，我们先在客户与服务器之间建立连接，然后假设服务器主机崩溃并重启。前一节中，当我们发送数据时，服务器主机仍然处于崩溃状态；本节中，我们将在发送数据前重新启动已经崩溃的服务器主机。模拟这种情形的最简单方法就是：先建立连接，再从网络上断开服务器主机，将它关机后再重新启动，最后把它重新连接到网络中。我们不想客户知道服务器主机的关机（我们将在5.16节讨论这一点）。

正如前一节所述，如果在服务器主机崩溃时客户不主动给服务器发送数据，那么客户将不会知道服务器主机已经崩溃。（这里假设我们没有使用`SO_KEEPALIVE`套接字选项)。所发生的步骤如下所述。

（1）我们启动服务器和客户，并在客户键入一行文本以确认连接已经建立。

（2）服务器主机崩溃并重启。

（3）在客户上键入一行文本，它将作为一个TCP数据分节发送到服务器主机。

（4）当服务器主机崩溃后重启时，它的TCP丢失了崩溃前的所有连接信息，因此服务器TCP对于所收到的来自客户的数据分节响应以一个`RST`。

（5）当客户TCP收到该`RST`时，客户正阻塞于`readline`调用，导致该调用返回`ECONNRESET`错误。

如果对客户而言检测服务器主机崩溃与否很重要，即使客户不主动发送数据也要能检测出来，就需要采用其他某种技术（诸恶SO_KEEPALIVE套接字选项或某些C/S心搏函数）。



## 5.16 服务器主机关机

前面两节讨论了服务器主机崩溃或无法通过网络到达的情形。本节考虑当我们的服务器进程正在运行时，服务器主机被操作员关机将会发生什么。

Unix系统关机时，init进程通常先给所有进程发送`SIGTERM`信号（该信号可被捕获)，等待一段固定的时间（往往在5到20秒之间），然后给所有仍在运行的进程发送`SIGKILL`信号（该信号不能被捕获）。这么做留给所有运行的进程一小段时间来清除和终止。如果我们不捕获`SIGTERM`信号并终止，我们的服务器将由`SIGKILL`信号终止。当服务器子进程终止时，它的所有打开着的描述符都被关闭，随后发生的步骤与5.12节中讨论过的一样。正如那一节所述，我们必须在客户中使用`select`或`poll`函数，使得服务器进程的终止一经发生，客户就能检测到。



## 5.17 TCP 程序例子 & 数据结构传输





















