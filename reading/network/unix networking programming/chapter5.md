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

POSIX信号语义

我们把符合POSIX的系统上的信号处理总结为一下几点

* 一旦安装了信号处理函数，它便一直安装着（较早期的系统是每执行一次就将其拆除）。

* 在一个信号处理函数运行期间，正被递交的信号是阻塞的。而且，安装处理函数时在传递给sigaction函数的sa_mask信号集中指定的任何额外信号也被阻塞。在图中，我们将sa_mask置为空集，意味着除了被捕获的信号外，没有额外的信号被阻塞。

* 如果一个信号在被阻塞期间产生了一次或多次，那么该信号解阻塞之后通常只递交一次，也就是说Unix信号默认是不排队的。我们将在下一节查看这一行的一个例子。POSIX实时标准一些排队的可靠信号，不过本书中我们不使用。

* 利用sigprocmask函数选择性地阻塞和解阻塞一组信号是可能的。这使得我们可以做到在一段临时去代码执行期间，防止捕获某些信号，以此保护这段代码。

## 5.9 处理SIGCHLD信号

设置将死（zombie）状态的目的是维护子进程的信息，以便父进程在以后某个时候获取。这些信息包括子进程的进程ID、最终状态以及资源利用信息（CPU时间、内存使用量等等）。如果一个进程终止，而该进程有子进程处于将死状态，那么它的所有僵死子进程的父进程ID将被置为1.进程这些子进程的init进程将清理它们（也就是说init进程将wait它们，从而去除它们的僵死状态）。有些unix系统在ps命令输出的command栏以<defunct>指明僵死进程。

***处理僵死进程***

我们显然不愿意留存僵死进程。它们占用内核中的空间，最终可能导致我们消耗尽进程资源。无论何时我们fork子进程都得wait它们，以防它们成为僵死进程。为此我们建立一个俘获SIGCHLD信号的信号处理函数，在函数体重我们调用wait。通过在图所示代码的listen调用之后增加如下函数调用：

```
Signal(SIGCHLD, sig_chld);
```

我们就建立了该信号处理函数。这必须在fork第一个子进程前完成，且只做一次。我们接着定义名为sig_chld的这个信号处理函数，如下:
```
#include "unp.h"

void sig_child(int signo)
{
	pid_t pid;
	int   stat;

	pid = wait(&stat);
	printf("child %d terminated\n", pid);
	return;
}

***在System V和Unix 98标准下，如果一个进程把SIGCHLD的处置设为SIG_IGN，它的子进程就不会变为僵死进程。不幸的是，这种做法仅仅适用于System V和Unix98，而POSIX明确表示没有规定这样做。处理僵死进程的可移植方法就是捕获SIGCHLD，并且调动wait或waitpid。 ****

```

在solaris9下如此编译本程序：以图5-2中代码为基础，加上对Signal的调用以及我们的sig_chld信息处理函数，而且所用的signal函数来自系统自带的函数库。

具体的各个步骤如下：

1）我们键入EOF字符来终止客户。客户TCP发送一个FIN给服务器，服务器响应一个ACK。

2)收到客户的FIN导致服务器TCP递送一个EOF给子进程阻塞中的readline，从而子进程终止。

3）当SIGCHLD信号递交时，父进程阻塞于accept调用。sig_chld函数执行，其wait调用取到子进程的PID和终止状态，随后是printf调用，最后返回。

4)既然该信号是在父进程阻塞与慢系统调用（accept）时父进程捕获的，内核就会使accept返回一个EINTR错误（被中断的系统调用）。而父进程不处理该错误，于是中止。

这个例子是为了说明，在编写捕获信号的网络程序中，我们必须认清被中断的系统调用且处理他们。在这个运行在solaris9环境下的特殊例子，标准c函数库中提供的signal函数不会使内核自动重启被中断的系统调用。

***处理被中断的系统调用***

我们用术语慢系统调用（slow system call）描述过accept函数，该术语也适用于那些可能永远阻塞的系统调用。永远阻塞的系统调用是指调用有可能永远无法返回.举例来说，如果没有客户端连接到服务器上，那么服务器的accept调用就没有返回的保证。类似地，在图5.3中如果客户从未发送过一行要求服务器回射的文本，那么服务器的read调用将永不返回。其他满系统调用的例子是对管道和终端设备的读和写。一个值得注意的例外是磁盘I/O，它们一般都会返回到调用者。

适用于慢系统调用的基本规则是：当阻塞于某个慢系统调用的一个进程捕获某个信号且相应信号处理函数返回时，该系统调用可能返回一个EINTR错误。有些内核自动启动某些被中断的系统调用。不过为了便于移植，当我们编写捕获信号的程序时(多数并发服务器捕获SIGCHLD),我们必须对慢系统调用返回EINTR有所准备。移植性问题是由早起使用的修饰词“可能”、“有些" 和对POSIX的SA_RESTART标志的支持是可选的这一事实造成的。
即使某个实现支持SA_RESTART标志，也并非所有被中断系统调用都可以自动重启。举例来说，大多数BSD的实现从不自动重启select，其中有些实现从不重启accept和recvfrom.

为了处理被中断的accept，我们把图5-2中对accept的调用从for循环开始改起，如下所示：

```
for (;;) {
	clilen = sizeof(cliaddr);
	if ((connfd = accpet(listenfd, (SA *) &cliaddr, &clilen)) < 0) {
		if (errno == EINTR)
			continue; /*back to for() */		
		else
			err_sys("accept error");
	}
}
```

注意，我们调用的是accept函数本身而不是它的包裹函数Accept，因为我们必须自己处理该函数的失败情况。

这段代码所做的事情就是自己重启被中断的系统调用。对于accpet以及诸如read、write、select和open之类函数来说，这是合适的。不过有一个函数我们不能重启：connect。如果该函数返回EINTR,我们就不能在次调用它，否则将立即返回一个错误。当connect被一个捕获的信号中断而且不自动重启时，我们必须调用select来等待连接完成。

## 5.10 wait和waitpid函数

在图5-7中，我们调用了函数wait来处理已终止的子进程。

```
#include <sys/wait.h>
pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options); / * 均返回：若成功则为进程ID，若出错则为0或者-1 */
```

函数wait和waitpid均返回两个值：已终止子进程的进程id号，以及通过statloc指针返回的子进程终止状态。我们可以调用三个宏来检查终止状态，并辨别子进程是正常终止、由某个信号杀死或停止子进程的作业控制信号值。在图15-10中，我们将为此目的使用宏WIFEXITED和WEXITSTATUS。

如果调用wait的进程没有已终止的子进程，不过有一个或多个子进程仍在执行，那么wait将阻塞现有子进程第一个终止为止。

waitpid函数就等待哪个进程以及是否阻塞给了我们更多的控制。首先，pid参数允许我们指定想等待的进程id，值-1表示等待第一个终止的子进程。（另有一些处理进程组id的可选值，不过本书中用不上）。其次，options参数允许我们指定附加选项。最常用的选项时WNOHANG,它告知内核在没有已终止子进程时不要阻塞.

***函数wait和waitpid的区别***

我们现在图示出函数wait和waitpid在用来清理已终止子进程时的区别。为此，我们把tcp客户端程序修改为如图5-9所示。客户端建立5个与服务器的连接，随后在调用str_cli函数时仅用第一个连接。建立多个连接的目的是从并发服务器上派生多个子进程。

```
#include "unp.h"

int main(int argc, char **argv)
{
	int i, sockfd[5];
	struct sockaddr_in servaddr;

	if (argc != 2)
		err_quit("usage: tcpcli <IPaddress>>");
	for (i = 0; i < 5; i++) {
		sockfd[i] = Socket(AF_INET, SOCK_STREAM, 0);
		bzero(&servaddr, sizeof(servaddr));
		servaddr.sin_family = AF_INET;
		servaddr.sin_port = htons(SERV_PORT);
		Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
		Connect(sockfd[i], (SA *) &servaddr, sizeof(servaddr));
	}
}
```

当客户端终止时，所有打开的描述符由内核由内核自动关闭（我们不调用close，进调用exit），且所有5个连接基本在同一时刻终止。这就引发了5个FIN，每个连接一个，它们反过来使用服务器的5个子进程基本在同一时刻终止。这又导致差不多在同一时刻有5个SIGCHLD信号递交给父进程，如图。

正是这种同一信号多个实例的递交造成了我们即将查看的问题。

我们首先在后台运行服务器，接着运行新的客户端。我们的服务器程序由图5-2修改而来，它调用signal函数，把图5-7中的函数建立为SIGCHLD的信号处理函数。

```
tcpserv03 &
[1] 20419
tcpcli04 127.0.0.1
hello
hello
^D
child 20426 terminated
```	

我们注意到的第一件事是只有一个printf输出，而当时我们预期所有5个子进程都终止了。如果运行ps，我们将发现其他4个子进程仍作为僵死进程存在着。

建立一个信号处理函数并在其中调用wait并不足以防止出现僵死进程。本问题在于：所有5个信号都在信号处理函数执行前产生，而信号处理函数只执行一次，因为Unix信号一般是不排队的。更严重的是，本问题是不确定的。在我们刚刚运行的例子中，客户端与服务器端在同一主机上，信号处理函数执行一次，留下4个僵死金出的那个。但是如果我们在不同的主机上运行客户端和服务器，那么信号处理函数一般执行2次：一次是第一个产生的信号引起的，由于另外4个信号在信号处理函数第一次执行时发生，因此该处理函数仅仅被再调用一次，留下3个僵死进程。不过有时候，依赖FIN到达服务器主机的时机，信号处理函数可能会执行3次甚至4次。

正确的解决办法是调用waitpid而不是wait，图中给出了正确处理SIGCHLD的sig_chld函数。这个版本管用的原因在于：我们在一个循环内调用waitpid，以获取所有已终止子进程的状态。我们必须制定WNOHANG选项，告知waitpid在有尚未终止的子进程在运行时不要阻塞。我们在图5-7中不能在循环内调用wait，因为没有办法防止wait在正运行的子进程上有为终止时阻塞。

```
#include "unp.h"

void sig_chld(int signo)
{
	pid_t pid;
	int stat;
	while ((pid = waitpid(-1, &stat, WNOHANG)) > 0)
		printf("child %d terminated\n", pid);
	return;
}
```

图5-12给出了我们的服务器程序的最终版本。它正确处理accpet返回的EINTR，并建立一个给所有已终止子进程调用waitpid的信号处理函数

本节的目的是示范我们在网络编程时可能遇到的三种情况：

1）当fork子进程时，必须捕获SIGCHLD信号；

2）当捕获信号时，必须处理被中断的系统调用；

3 SIGCHLD的信号处理函数必须正确编写，应使用waitpid函数以避免留下僵死进程。

```
#Include "unp.h"

int main(int argc, char **argv)
{
	int listenfd, connfd;
    pid_t childpid;
    socklen_t clilen;
    struct sockaddr_in cliaddr, serveraddr;
    void sig_child(int);
    listenfd = Socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = htonl(INADR_ANY);
    serveraddr.sin_port = host(SERV_PORT);
    Bind(listenfd, (SA *)&serveraddr, sizeof(serveraddr));
    Listen(listenfd, LISTENQ);
    Signal(SIGCHLD, sig_chld); /* must call waitpid() */
    for (;;) {
		chilen = sizeof(cliaddr);
        if ((connfd = accept(listenfd, (SA *)&cliaddr, &clilen)) < 0) {
			if (errno = EINTR)
				continue; /* back to for() */
			else
				err_sys("accept error");
		}
		if ((childpid = Fork()) == 0) { /* child process */
			Close(listenfd); /* close listening socket */
			str_echo(connfd); /* process the request */
			exit(0);
		}
		Close(connfd); /* parent closes connected socket */
	}
}

我们的TCP服务器程序最终版本加上SIGCHLD函数处理函数能够处理上述三种情况。
```

### 5.11 accept 返回前连接终止

类似于前一节中介绍的被中断系统调用的例子，另有一种情形也能够导致accept返回一个非致命的错误，在这种情况下，只需要在此调用accept。图5-13所示的分组序列在较忙的服务器上已经出现过。

这里，三路握手完成从而连接建立之后，客户端TCP却发送了一个RST（复位）。在服务器端看来，就在该连接已由TCP排队，等着服务器进程调用accept的时候RST到达。稍后，服务器进程调用accept。

```
模拟这种情形的一个简单方法就是：启动服务器，让它第阿勇socket、bind和listen，然后再调用accept之前睡眠一小段时间。在服务器进程睡眠的时候，启动客户端，让它调用socket和connect。一旦connect返回，就设置SO_LINGER套接字选项以产生这个RST，然后终止。
```

但是，如何处理这种终止的连接依赖于不同的实现。BSD的实现完全在内核中处理终止的连接，服务器进程根本看不到。然而大多数SVR4实现返回一个错误给服务器进程，作为accept的返回结果，不过错误本身取决于实现。这些SVR4实现返回一个EPROTO error值，而POSIX指出返回的errno值必须是ECONNABORTED。POSIX做出修改的理由在于：流子系统发生某些致命的协议相关事件时，也会返回EPROTO。要是对于客户单引起的一个已建立连接的非致命中止也同样的错误，那么服务器就不知道该再次调用accept还是不该了。换成ECONNABORTED错误，服务器就可以忽略它，再次调用accept就行。

```
BSD的内核不把该错误传递给进程的做法所涉及的步骤在TCPv2中得到阐述。引发该错误的RST在第964页得到处理，导致tcp_close被调用。该函数在第897页调用in_pcdbetach,它又转而在第719页第阿勇sofree。sofree函数发现待中止的连接仍在监听套接字的已完成连接队列中，于是从该队列中删除该连接，并释放相应的已连接套接字。当服务器最终调用accpect函数时，它根本不知道曾经有一个已经完成的连接稍后被从已完成队列中删除了。
```

在16.6节我们将再次回到这些中止的连接，查看与select函数和正常阻塞模式下的监听套接字组合时它们是如何成为问题的。

### 5.12 服务器进程终止

现在启动我们的客户端/服务器对，然后杀死服务器子进程。这是在模拟服务器进程崩溃的情形，我们可从中查看客户端将发生什么。（我们必须小心区别即将讨论的服务器进程崩溃与将在5.14节讨论的服务器主机崩溃。）所发生的步骤如下所述。




