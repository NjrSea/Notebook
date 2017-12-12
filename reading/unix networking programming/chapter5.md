# 第五章 TCP客户端/服务器程序示例

## 5.1 概述

我们将在本章使用前一章介绍的基本函数编写一个完整的TCP客户端/服务器程序。这是个简单的例子是执行如下步骤的一个回射服务器：

1）客户端从标准输入读入一行文本，并写给服务器；
2）服务器从网络输入读入一行文本，并回射给客户端；
3）客户从网络输入读入折行回射文本，并显示在标准输出上。

我们在客户端与服务器之间画了两个箭头，不过他们实际上构成一个全双工的TCP连接。fgets和fputs这两个函数来自标准I/O函数库，writen和readline。

除了以正常的方式运行本例子外，我们还会讨论他的许多边界条件：客户端和服务器启动时发生了什么？客户端正常终止时发生了什么？若服务器进程在客户端之前终止，则客户端会发生什么？若服务器主机崩溃，则客户端发生什么？如此等等。通过观察这些情况，弄清在网络层次发生了什么以及他们如何反应到套接字API，我们将更多地理解这些层次的工作原理，并体会如何编写应用程序代码来处理这些情形。

在所有这些例子中，我们把诸如地址端口之类特定于协议的常值硬编写到代码中。这么做有两个原因：一是我们必须确切了解特定于协议的地址结构中应存放什么内容；二是我们尚未讨论到可以使得代码更便于移植的函数库，这些库函数将在第11章中讨论。

我们现在就留意，随着学习越来越多的网络编程知识，我们将在后续各个章节中多次修改本章的客户端和服务器程序。

## 5.2 TCP回射服务器程序：main函数

我们的TCP客户端和服务器程序遵循图4-1所示的函数调用流程。图5-2给出了其中的并发服务器程序。

```
#include "unp.h"

int main(int argc, char **argv)
{
	int listenfd, connfd;
	pid_t childpid;
	socklen_t clilen;
	struct sockaddr_in cliaddr, servaddr;
	listenfd = Socket(AF_INET, SOCK_STREAM, 0);
	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(SERV_PORT);
    Bind(listenfd, (SA *)&servaddr, sizeof(servaddr));
	Listen(listenfd, LISTENQ);
	for (;;) {
		clilen = sizeof(cliaddr);
		connfd = Accept(listenfd, (SA *)&cliaddr, &clilen);
		if ((childpid = Fork()) == 0) { /* child process */
			Close(listenfd);	/* close listening socket */
			str_echo(connfd);	/* process the request */
			exit(0);
		}
		Close(connfd);		/* parent closes connected socket */
	}	
}
```

创建套接字，捆绑服务器的总所周知端口

9 ~ 15 创建一个TCP套接字。在捆绑该TCP套接字的网际网套接字地址结构中填入通配地址（INADDR_ANY)和服务器的众所周知端口（SERV_PORT,在头文件unp.h中其值定义为9877）。捆绑通配地址是告知系统：要是系统多事宿主机，我们将接受目的地址为任何本地接口的连接。我们对TCP端口的选择基于图2-10.它应该比1023大，比4000小，比49152小，而且不应该与任何已注册的端口冲突。listen把该套接字转换成一个监听套接字。

等待完成客户端连接

17 ~ 18 服务器阻塞于accept调用，等待客户端连接的完成。

并发服务器

19 ~ 24 fork为每个客户端派生一个处理它们的子进程。正如我们在4.8节讨论的那样，子进程关闭监听套接字，父进程关闭已连接套接字。

## 5.3 TCP回射服务器程序：str_echo函数

```
#include "unp.h"

void str_echo(int sockfd)
{
	ssize_t n;
	char buf[MAXLINE];
again:
	while ((n = read(sockfd, buf, MAXLINE)) > 0)
		Writen(sockfd, buf, n);

	if (n < 0 && errno == EINTR)
		goto again:
	else if (n < 0)
		err_sys("str_echo: read error");
}
```

读入缓冲区并回射其中内容

8~9 read函数从套接字读入数据，writen函数把其中内容太回射给客户端。如果关闭客户端连接，那么收到客户端的FIN将导致服务器子进程的read函数返回0，这又导致str_echo函数的返回，从而在图5-2终止子进程。

这一版的新作者在图3-17和图3-18中修正了第二版对应的图3-16和图3-17中的一个错误，也就是在读入一些数据后再碰到EOF的情况下，Stevens先生把读入字符数少减了1；不过他们却在图5-3中过早地使用了不易文本行为中心的代码，而本书以文本行为中心的回射服务讨论中，隐含假设服务器也是面向文本行从套接字读取数据，以便进一步处理，尽管纯粹的回射服务没有这个需要。从这个意义上看，第2班中对应的图5-3更为确切，而尽管新作者修改了str_echo函数，在随后的章节中却又不加修改地沿用了Stevens先生的解释，可能会让读者不知所云。为此译者建议读者仍然采用第2版中对应的图5-3。

## 5.4 TCP回射客户端程序：main函数

图5-4所示为TCP客户端的main函数.

```
int main(int argc, char **argv)
{
	int sockfd;
	struct sockaddr_in servaddr;
	
	if (argc != 2)
		err_quit("usage: tcpcli <IPAddress>");
	sockfd = Socket(AF_INET, SOCK_STREAM, 0);
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(SERV_PORT);
	Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
	Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));
	str_cli(stdin, sockfd);
	exit(0);
}

创建套接字，填装网际网套接字地址结构

9~13

连接到服务器

14~15



## 5.5 TCP回射客户端程序：str_cli函数

```
#include "unp.h"

void str_cli(FILE *fp, int sockfd)
{
	char sendline[MAXLINE], recvline[MAXLINE];
	while (Fgets(sendline, MAXLINE, fp) != NULL) {
		Writen(sockfd, snedline, strlen(sendline));
		if (Readline(sockfd, recvline, MAXLINE) == 0)
			err_quit("str_cli: server terminated permaturely");
		Fputs(recvline, stdout);
	}
}
```

读入一行，写到服务器

6~7 fgets读入一行文本，writen把该行发送给服务器。

服务器读入回射行，写到标准输出

8~10

返回到main函数

11~12

## 5.6 正常启动

尽管我们的TCP程序例子很小，然而对于我们弄清客户端和服务器如何启动，如何终止，更为重要的是当发生某些错误是将会发生什么，本例子却至关重要。只要高清这些边界条件以及它们与TCP/IP协议的相互作用，我们才能写出能够处理这些情况的程序。

首先，我们在主机linux上后台启动服务器

tcpserv01 &

服务器启动后，它调用socket、bind、listen和accept，并阻塞与accept调用。在我们启动客户端之前，我们运行netstat程序来检查服务器监听套接字的状态。

netstat -a

Active Internet connections (servers and established)

我们这里只给出了输出的第一行以及我们最关心的那一行。本命令列出系统中所有套接字的状态，可能有大量输出。我们必须指定-a标志以查看监听套接字。

这个输出正式我们所期望的：有一个套接字处于LISTEN状态，它有通配的本地IP地址，本地端口为9877.netstat用星号“*”来表示一个为0的IP地址（INADDR_ANY，通配地址）或为0的端口号。

我们接着在同一主机上启动客户端，并指定服务器主机的IP地址为127.0.0.1（环回地址）。当然我们也可以指定该地址为该主机的普通IP地址。

客户端调用socket和connect，后者引起TCP的三路握手过程。当三路握手完成后，客户端中的connect和服务器中的accept均返回，连接建立。接着发生的步骤如下：

1）客户端调用str_cli函数，该函数阻塞与fgets调用，因为我们还未曾键入过一行文本。

2）当服务器中的accept返回时，服务器调用fork，再由子进程调用str_echo。该函数调用readline，readline调用read，而read在等待客户端发送一行文本期间阻塞。

3）另一方面，服务器父进程在此调用accept并阻塞，等待下一个客户端连接。

至此，我们有3个都在睡眠的进程：客户端进程、服务器父进程和服务器子进程。

```
当三路握手完成时，我们特意首先列出客户端的步骤，接着列出服务器的步骤。从图中可知其原因：客户端接受到三路握手的第二个分节时，connect返回，而服务器要到收到三路握手的第三个分节才返回，即在connect返回之后再过一半RTT才返回。

我们特意在同一主机上运行客户端和服务器，因为这是实验客户端、服务器程序的最简单方式。既然我们在同一主机上运行客户端和服务器，netstat给出了对应所建立TCP的两行额外的输出。

地址个ESTABLISHED行对应于服务器子进程的套接字，因为它的本地套接字端口号是9877；第二个ESTABLISHED行对应于客户端进程的套接字。要是我们在不同的主机上运行客户端和服务器，那么客户端主机就只输出客户端进程的套接字，服务器主机也只输出两个服务器进程的套接字。

我们也可用ps命令来检查这些进程的状态和关系。

我们已使用ps相当特定的命令行参数限定它只输出与本讨论相关的信息。从输出中可见，客户端和服务器运行在同一个窗口中。pid和ppid列给出了进程间的斧子关系。由于子进程的ppid是父进程的pid，我们可以看见，第一个tcpserv01行是父进程

## 5.7 正常终止

至此连接已经建立。

当前连接的客户端进入了TIME_WAIT状态，而监听服务器仍在等待另一个客户端连接。

步骤：

1）当我们键入eof字符时，fgets返回一个空指针，于是str_cli函数返回。

2）当str_cli返回到客户端的main函数时，main通过调用exit终止。

3）进程终止处理的部分工作是关闭所有打开的描述符，因此客户端打开的套接字由内核关闭。这导致客户端tcp发送一个fin给服务器，服务器tcp以ack响应，这就是tcp连接终止序列的前半部分。至此，服务器套接字处于CLOSE_WAIT状态，客户端套接字处于FIN_WAIT_2状态。

4）当服务器tcp接受fin时，服务器阻塞于readline调用，于是readline返回0.这导致str_echo函数返回服务器子进程的main函数。

5) 服务器子进程通过调用exit来终止。

6）服务器子进程中打开的所有描述符随之关闭。由子进程来关闭已连接套接字会引发TCP连接终止序列的最后两个分节：一个从服务器到客户端的FIN和一个从客户端到服务器的ACK。至此，连接完全终止，客户端套接字进入TIME_WAIT状态。

7）进程终止处理的另一部分内容是：在服务器子进程终止时，父进程发送一个SIGCHLD信号。这一点在本例中发生了，但我们没有在代码中捕获该信号，而该信号的默认行为是被忽略。既然父进程未加处理，子进程于是进入僵死状态。我们可以使用ps命令验证这一点。

我们必须清理僵死进程，这就涉及Unix信号处理。我们将在下一节概述信号处理。


