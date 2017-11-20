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


