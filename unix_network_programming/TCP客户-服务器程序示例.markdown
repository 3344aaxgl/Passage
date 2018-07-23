---
layout: post
title:  "TCP客户/服务器程序示例"
date:   2018-07-17 13:39:51 +0800
categories: unix_network_programming
tags: c++
description: unix network programming读书笔记
---

## TCP回射服务器程序：main函数

{% highlight c++ %}

#include "unp.h"

#define SERV_PORT 54321
#define LISTENQ 256

int main(int argc, char*argv[])
{
    struct sockaddr_in serv_sockaddr,client_sockaddr;
    socklen_t clilen;
    int serv_fd,conn_fd;
    pid_t childpid;

    serv_fd = socket(AF_INET, SOCK_STREAM, 0);

    memset(&serv_sockaddr, 0, sizeof(serv_sockaddr));
    serv_sockaddr.sin_family = AF_INET;
    serv_sockaddr.sin_port = htons(SERV_PORT);
    //通用地址
    serv_sockaddr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(serv_fd, (const struct sockaddr*) &serv_sockaddr, sizeof(serv_sockaddr));
    listen(serv_fd, LISTENQ);

    for(;;)
    {
        //阻塞与accept，等待连接到来
        conn_fd = accept(serv_fd, (struct sockaddr*) &client_sockaddr, &clilen);
        //fork子进程处理新连接
        if((childpid = fork()) == 0)
        {
            close(serv_fd);
            str_echo(conn_fd);
            exit(0);
        }
        close(conn_fd);
    }
    return 0;
}

{% endhighlihgt %}

## TCP回射服务器程序str_echo函数

{% highlight c++ %}

int str_echo(int fd)
{
    char buf[MAXLEN];
    int num;
    while(1)
    {
        if((num = read(fd, buf, sizeof(buf))) <0)
        {
            if(errno = EINTR)
              ;           
            else
            {
                perror("read error");
                exit(-1);    
            };            
        }
        else if( num == 0)
          break;
        else
          writen(fd,buf,num);
    }
}
{% endhighlight %}

## TCP回射客户程序：main函数

{% highlig c++ %}

int main(int argc, char* argv[])
{
    int sock_fd;
    struct sockaddr_in sin;
    inet_pton(AF_INET, argv[1], &sin.sin_addr);
    sin.sin_port = htons(atoi(argv[2]));
    sin.sin_family = AF_INET;

    sock_fd = socket(AF_INET, SOCK_STREAM, 0);

    connect(sock_fd, (const struct sockaddr*) &sin, sizeof(sin));

    str_cli(stdin, sock_fd);
    exit(0);

    return 0;
}

{% endhighlight %}

## TCP回射客户程序:str_cli函数

{% highlight c++ %}

void str_cli(FILE* fp, int sock_fd)
{
    char sendline[256], recvline[256];
    while(fgets(sendline, MAXLEN, fp) !=NULL)
    {
        writen(sock_fd, sendline, strlen(sendline));
        if(readline(sock_fd, recvline, MAXLEN) == 0)
        {
            perror("str_cli:server terminated prematurely");
        }
        fputs(recvline, stdout);
    }
}
    
{% endhighlight %}

## 正常启动

服务器启动后，它调用socket，bind，listen和accpet，并阻塞与accept。在启动客户之前，运行netstat程序来检查服务器监听套接字的状态。此时服务器监听套接字处理LISTEN状态。

客户调用socket和connect，后者引起TCP的三路握手，当三路握手完成后，客户中的connect和accpet均返回。于是连接建立。接着发生如下步骤：

* 当客户调用str_cli，该函数将阻塞于fgets调用，因为我们还未曾键入过一行文本。
* 当服务器中的accept返回时，服务器调用fork，再由子进程调用str_echo。该函数调用readline，readline调用read，而read在等待客户送入一行文件期间阻塞
* 另一方面，服务器父进程再次调用accept并阻塞，等待下一个客户连接。

通过netstat可以看到客户和服务器子进程的已连接套接字均处于ESTABLISHED状态。服务器父进程的监听套接字处于LISTEN状态。

## 正常终止

* 当我们键入EOF字符时，fgets返回一个空指针，于是str_cli函数返回。
* 当str_cli返回到客户main函数时，main通过exit终止
* 进程终止处理的部分工作是关闭所有打开的描述符，因此客户打开的套接字由内核关闭。这导致客户TCP发送一个FIN给服务器，服务器则以ACK响应，这就是TCP连接终止的前半部分。至此服务器套接字处于CLOSE_WAIT，客户则处于FIN_WAIT2。
* 当服务器TCP收到FIN时，服务器子进程阻塞于readline调用，于是readline返回0。这导致str_echo返回子进程的main函数。
* 服务器子进程通过调用exit来终止
* 服务器子进程中打开的描述符随之关闭，由子进程来关闭套接字会引发TCP连接终止序列的最后两个分节：一个从服务器到客户的FIN和一个从客户到服务器的ACK。至此，连接完全终止，客户套接字进入TIME_WAIT状态。
* 进程终止处理的另一部分内容是：在服务器子进程终止时，给父进程发送了一个SIGCHLD信号。父进程没有处理此信号，子进程进入僵死状态。

## POSIX信号处理

信号就是告知某个进程发生了某件事情的通知，有时也称为软件中断。信号通常是异步发生的，也就是说进程预先不知道信号的何时发生。

信号可以：

* 由一个进程发给另一个进程（或自身）
* 由内核发给某个进程

每个信号都有一个关联的处置，共有三种选择：

* 可以提供一个函数，只要有特定的信号发生时它就被调用。这样的函数成为信号处理函数，这种行为成为捕获信号。有两个信号不能被捕获，SIGKILL和SIGSTOP。信号处理函数由信号值这个单一的整数参数来调用，且没有返回值。
* 可以把某个信号的处置设置为SIG_IGN来忽略它，SIGKILL和SIGSTOP不能忽略。
* 可以把某个信号的处置设定为SIG_DFL来启用它的默认处置。默认处置通常是在收到信号后终止进程，其中某些信号还在当前的工作目录产生一个进程的核心映像。另有个别信号的默认处置是忽略SIGCHLD和SIGURG

**signal函数**

{% highlight c++ %}

typedef void Sigfunc(int);

Sigfunc * signal(int signo, Sigfunc* func)
{
    strcut sigaction act,oact;
    act.sa_handler = func;
    sigemptyset(&act.sa_mask);
    if(signo == SIGALARM)
    {
        #ifdef SA_INTERRUPT
          act.sa_flags |= SA_INTERRUPT;
        #endif
    }
    else
    {
        #ifdef SA_RESTART
          act.flags |= SA_RESTART;
    }
    if(sigaction(signo, &act, &oact) < 0)
      return SIG_ERR;
    return oact.sa_handler;
}

{% endhighlight %}

**POSIX信号语义**

* 一旦安装了信号处理函数，它便一直安装着
* 在一个信号处理函数运行期间，正被递交的函数是被阻塞的。而且，安装处理函数时在传递给sigaction函数的sa_mask信号集中指定的任何信号也被阻塞。
* 如果一个信号再被阻塞过程中产生了一次或多次，那么该信号被解阻塞之后通常只被递交一次，也就是说unix信号默认是不排队的。

## 处理SIGCHLD信号

如果一个进程终止，而该进程有子进程处于处于僵死状态，那么它的所有僵死子进程的父进程ID将被重置为1（init进程），继承这些子进程的init进程将清理他们。

**处理僵死进程**

当SIGCHLD信号递交时，父进程阻塞于accept调用。sig_chld函数执行，其wait调用取到子进程的PID和终止状态，随后是printf调用，最后返回。

{% highlight c++ %}

void sig_chld(int signo)
{
    pid_t pid;
    int stat;

    pid = wait(&stat);
    printf("child %d terminated\n", pid);
    return ;
}
//注册处理SIGCHLD信号的函数，必须在fork之前注册
signale(SIGCHLD, sig_chld);

{% endhighlight %}

**处理被中断的系统调用**

慢系统调用适用于那些可能永远阻塞的系统调用。永远阻塞的系统调用是指调用永远无法返回，多数网络支持函数都属于这一类。

适用于慢系统调用的基本规则是：当阻塞于某个慢系统调用一个进程捕获某个信号且相应的信号处理函数返回时，该系统调用可能返回一个EINTR错误。有些内核自动重启某些被中断的系统调用。

对于accpet以及read、write、open、select之类的函数来说，在返回EINTR时，可以再次调用它。当connetc被一个捕获中断而且不自动重启时，我们必须调用select来等待连接完成。

## wait和waitpid函数

{% highlight c++ %}

pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
//成功均返回进程ID，若出错则为0或-1
{% endhighlight %}

wait和waitpid均返回两个值，已终止子进程的进程ID，以及通过statloc指针返回的子进程终止状态。我们可以调用三个宏来检查终止状态，并辨别子进程是正常终止，由某个信号杀死还是仅仅由作业控制停止而已。

如果调用wait进程没有已终止的子进程，但有一个或多个子进程正在执行，那么wait将阻塞直到现有子进程第一个终止为止。

waitpid就等待那个子进程和是否阻塞给我我们更多的控制，pid允许我们指定想等待的进程，-1表示等待第一个终止的子进程。其次options参数允许我们指定附加选项，最长用的是WNOHANG，它告知内核在没有已终止子进程时不要阻塞。

**函数wait和waitpid的区别**

客户端建立5个与服务器的连接

{% highlight c++ %}

int main(int argc, char* argv[])
{
    int i,sock_fd[5];
    struct sockaddr_in sin;
    inet_pton(AF_INET, argv[1], &sin.sin_addr);
    sin.sin_port = htons(atoi(argv[2]));
    sin.sin_family = AF_INET;

    for(i = 0; i < 5; i++)
    {
        sock_fd = socket(AF_INET, SOCK_STREAM, 0);
        connect(sock_fd, (const struct sockaddr*) &sin, sizeof(sin));
    }

    str_cli(stdin, sock_fd[0]);
    exit(0);
}

{% endhighlight %}

当客户终止时，基本同时发送了5个FIN，这导致服务器的5个子进程终止，子进程发送5个SIGCHLD给父进程。


问题在于信号都在信号处理函数执行之前产生，而信号处理函数只执行一次。所以在信号处理函数中循环调用waitpid，并且必须制定WNOHANG选项，它告知waitpid在有尚未终止的子进程在运行时不要阻塞。不能使用wait，因为wait无法在尚有未终止子进程的情况下不阻塞。

{% highlight c++ %}

void sig_chld(int signo)
{
    pid_t pid;
    int stat;

    while((pid = waitpid(-1, &stat, WNOHANG)) > 0);
    printf("child %d terminated\n", pid);
    return ;
}

{% endhighlight %}

服务器代码如下：

* 当fork子进程时，必须捕获SIGCHLD
* 当捕获信号时，必须处理被中断的系统调用
* SIGCHLD的信号处理函数必须正确编写，应使用waitpid函数避免产生僵尸进程

{% highlight c++ %}

int main(int argc, char*argv[])
{
    struct sockaddr_in serv_sockaddr,client_sockaddr;
    socklen_t clilen;
    int serv_fd,conn_fd;
    pid_t childpid;

    serv_fd = socket(AF_INET, SOCK_STREAM, 0);

    memset(&serv_sockaddr, 0, sizeof(serv_sockaddr));
    serv_sockaddr.sin_family = AF_INET;
    serv_sockaddr.sin_port = htons(SERV_PORT);
    //通用地址
    serv_sockaddr.sin_addr.s_addr = htonl(INADDR_ANY);

    bind(serv_fd, (const struct sockaddr*) &serv_sockaddr, sizeof(serv_sockaddr));
    listen(serv_fd, LISTENQ);
    Signal(SIGCHLD, sig_chld);
    for(;;)
    {
        //阻塞与accept，等待连接到来
        conn_fd = accept(serv_fd, (struct sockaddr*) &client_sockaddr, &clilen);
        //fork子进程处理新连接
        if((childpid = fork()) == 0)
        {
            close(serv_fd);
            str_echo(conn_fd);
            exit(0);
        }
        close(conn_fd);
    }
    return 0;
}

{% endhighlight %}

## accept返回前连接中止

三次握手完成之后，客户TCP发送一个RST。在服务器端看来，就在该连接已有TCP排队，等着服务器进程调用accept的时候RST到达。稍后服务器调用accept。

模拟这种情形可以设置SO_LINGER套接字选项，打开这个开关，并等待0秒。

不同的系统有不同的处理方式，POSIX指出返回的errno必须是ECONNABORTED。服务器可以忽略这个错误并在次调用accept。

## 服务器进程终止

步骤如下所述

* 在同一个主机上启动服务器和客户，并在客户上键入一行文本，以验证一切正常。
* 找到服务器子进程的进程ID，并执行kill命令杀死它，作为进程处理的部分工作，子进程中所有被打开着的描述符都被关闭。这就导致向客户发送一个FIN，而客户则响应以ACK。
* SIGCHLD被发送给服务器父进程，并得到正确处理
* 客户上没有发生任何特殊的事情，客户TCP接收到来自服务器TCP发送的FIN，并响应以ACK
* 此时监听套接字处于LISTEN状态，服务器已连接套接字处于FIN_WAIT2状态，客户服务器处于CLOSE_WAIT状态
* 在主机上在键入一行，当服务器接收到来自客户的数据时，响应以RST
* 客户进程看不到这个RST，因为它在调用writen后立即调用readline，并且由于第二步中接收的FIN，所调用的readline立即返回0。

问题在于当FIN到达时，客户正阻塞在fgets调用上。客户实际在应对两个描述符——套接字和用户输入，它不能单纯阻塞在这两个源中其中某个特定输入源中。而是应该阻塞在其中任何一个源的输入上，这正是select和poll这两个函数的目的之一。

## SIGPIPE信号

当一个进程向某个已收到RST的套接字执行写操作时，内核向该进程发送一个SIGPIPE信号。该信号的默认行为是终止进程，因此进程必须捕获它以免不情愿的被终止。

## 服务器主机崩溃

步骤如下所述：

* 当服务器主机崩溃时，已有的网络连接不发出任何东西。这里的假设是主机崩溃，而不是由操作人员执行关机命令
* 我们在客户上键入一行文本，它由writen写入内核，再由客户TCP作为一个分节送出。客户随后阻塞于readline，等待回射应答
* 如果我们用tcpdump观察网络就会发现，客户TCP持续重传数据分节，试图从服务器上接收一个ACK分节。当客户TCP最终放弃时，给客户进程返回一个错误。错误为ETIMEOUT、EHOSTUNREACH或ENETUNREACH。

## 服务器主机崩溃后重启

步骤如下：

* 启动服务器和客户，并在客户键入一行文本以确认连接建立
* 服务器主机崩溃并重启
* 
* 在客户上键入一行文本，它将作为一个数据分节发送到服务器主机
* 当服务器主机崩溃后重启时，它的TCP丢失了崩溃前的所有连接信息，因此服务器TCP对于所有收到的来自客户的数据分节响应以一个RST。
* 当客户收到该RST时，客户正阻塞于readline调用，导致该调用返回ECONNREST错误。

如果对客户而言服务器主机崩溃与否很重要，即使客户不主动发送数据也能检测出来，就需要采用其他技术，SO_KEEPLIVE套接字选项或心跳包。

## 服务器主机关机

Unix系统关机时，init进程通常先给所有进程发送SIGTERM信号（改信号可以被捕获），等待一段固定的时间，然后给所有仍在运行的进程发送SIGKILL信号。这么做留给所有运行的程序一小段时间来清除和终止。如果不捕获SIGTERM信号并终止，服务器将由SIGKILL信号终止。

## TCP程序例子小节

在TCP客户和服务器可以彼此通信之前，每一端都得指定连接的套接字：本地IP地址、本地端口、外地IP地址、外地端口。外地地址和外地端口必须在客户调用connect时指定，而两个本地值通常就由内核作为connect的一部分来选定。客户可以在调用connect之前，通过调用bind来指定其中一部分或全部两个本地值，不过这种做法并不常见。

服务器本地端口由bind指定。bind调用中服务器指定的本地IP地址通常是通配IP地址。如果服务器在一个多宿主机上绑定通配IP地址，那么可以在连接后通过getsockname来确定本地IP地址。两个外地值则由accept调用返回给服务器，可以通过getpeername来获取客户的IP地址和端口号。

## 数据格式

**在客户与服务器之间传递文本串**

{% highlight c++ %}

void str_echo(int sockfd)
{
    long arg1, arg2;
    sszie_t n;
    char line[MAXLEN];

    for(;;)
    {
        if((n = readline(scokfd,line, MAXLEN)) == 0)
          return;
        if(sscanf(line, "%ld%ld", &arg1,&arg2) == 2)
          snprintf(line, sizeof(line), "%ld\n", arg1 + arg2);
        else
          snprintf(line, sizeof(line), "input error\n");
        n = strlen(line);
        writen(sockfd,line, n);
    }
}

{% endhighlight %}

不论客户和服务器的字节序如何，客户和服务器都工作的很好


**在客户与服务器之间传递二进制结构**

{% highlight c++ %}

struct args
{
    long arg1;
    long arg2;
};

struct result
{
    long sum;
};

void str_cli(FILE* fp, int sockfd)
{
    char sendline[MAXLEN];
    struct args args;
    struct result result;

    while(fgets(sendline, MAXLEN, stdin))
    {
        if(sscanf(sendline,"%ld%ld", &args.arg1, &args.arg2) != 2)
        {
            printf("invalid input: %s", sendline);
            continue;
        }
        writen(sockfd, &args, sizeof(args));
        if(readn(sockfd, &result, sizeof(result)) == 0)
        {
            perror("str_cli:server terminated prematurely");
            exit(-1);
        }
        printf("%ld\n", result.sum);
    }
}

void str_echo(int sockfd)
{
    sszie_t n;
    struct args args;
    struct result result;

    for(;;)
    {
        if((n = readn(sock_fd, &args, sizeof(args))) == 0)
          return;
        result.sum = args.arg1 + args.arg2;
        writen(sockfd, &result, sizoef(result));
    }
}

{% endhoghlight %}

本例子存在三个问题：

* 不同的实现以不同的格式存储二进制
* 不同的实现在存储相同的C数据类型上可能存在差异
* 不同的实现给结构打包的方式存在差异，取决于各种数据类型所用的位数以及机器的对齐限制

解决这种数据格式问题有两个常用的方法

* 把所有数值作为文本串来传递
* 显示定义所支持数据的二进制格式并以这样的格式在客户与服务器之间传递所有数据。