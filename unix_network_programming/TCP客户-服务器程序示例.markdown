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