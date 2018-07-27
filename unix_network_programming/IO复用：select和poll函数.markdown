---
layout: post
title:  "I/O复用：select和poll函数"
date:   2018-07-23 13:39:51 +0800
categories: unix_network_programming
tags: c++
description: unix network programming读书笔记
---

I/O复用典型使用在下列网络应用场合

* 当客户处理多个描述符时，必须使用I/O复用。
* 一个客户同时处理多个套接字
* 如果一个TCP服务器既要处理监听套接字，又要处理已连接套接字
* 如果一个服务器既要处理TCP套接字，又要处理UDP
* 如果一个服务器要处理多个服务或者多个协议

## I/O模型

UNIX下可用的5中I/O模型的基本区别：

* 阻塞式I/O
* 非阻塞式I/O
* I/O复用
* 信号驱动式I/O
* 异步I/O

一个输入操作通常包括两个不同的阶段：

* 等待数据准备好
* 从内核向进程复制数据

对于一个套接字上的输入操作，第一步涉及等待数据从网络中到达。当所等待网络分组到达时，它被复制到内核的某个缓冲区中。第二步就是把数据从内核缓冲区复制到应用进程缓冲区

### 阻塞式I/O模型

进程调用recvfrom，其系统调用直到数据报到达且被复制到进程缓冲区中或者发生错误才返回。最常见的错误是系统调用被信号中断，进程在调用recvfrom开始到它返回的整段时间内是被阻塞的。recvfrom成功返回后，应用进程开始处理数据报。

![6-1.png](6-1.png)


### 非阻塞式I/O模型

进程把一个套接字设置成非阻塞是在通知内核：当所请求的I/O操作非得把本进程投入睡眠才能完成时，不要把本进程投入睡眠，而是返回一个错误。

![6-2.png](6-2.png)

当一个应用进程像这样对一个非阻塞描述符玄幻调用recvfrom时，我们称之为轮询。应用程序持续轮询内核，以查看某个操作是否就绪。

### I/O复用模型

有了I/O复用，我们就可以调用select或者poll，阻塞在这两个系统调用中的某一个之上。而不是阻塞在真正的I/O系统调用之上。

![6-3.png](6-3.png)

我们阻塞于select调用，等待数据报套接字变为可读。当select返回套接字可读这一条件时，我们调用recvfrom把所读数据报复制到应用进程缓冲区。

和阻塞式I/O相比，I/O复用 优势在于可以等待多个描述符就绪。

### 信号驱动式I/O模型

我么也可以用信号，让内核在描述符就绪时发送SIGIO信号通知我们。

![6-4.png](6-4.png)

我们首先开启套接字的信号驱动I/O功能，并通过sigaction系统调用安装一个信号处理函数。改系统调用立即返回，我们的进程继续工作，也就是说它没有被阻塞。当数据报准备好被读取时，内核就为该进程产生一个SIGIO信号。我们随后既可以在信号处理函数中调用recvfrom读取数据，并通知主循环数据已准备好待处理。也可以立即通知主循环，让他读取数据报。

优势在于等待数据报到达期间进程不阻塞，主循环可以继续执行，只要等待来自信号处理函数的通知：既可以是数据已准备好被处理，也可以是数据报已准备好被处理。

### 异步I/O模型

告知内核启动某个操作，并让内核在在整个操作完成后通知我们。这种模型与信号模型的区别在于信号驱动式I/O是由内核通知我们何时可以启动一个I/O操作，而异步I/O模型是由内核通知我们I/O操作何时完成。

![6-5.png](6-5.png)

我们调用aio_read函数，给内核传递描述符、缓冲区指针、缓冲区大小和文件偏移，并告诉内核当整个操作完成时如何通知我们。改系统调用立即返回，而且在等待I/O完成期间，进程不被阻塞。该信号知道数据已复制到应用进程的缓冲区才产生，这一点不同于信号I/O。

### 各种I/O模型的比较

前4种模型的主要区别在于第一阶段，因为他们的第二阶段是一致的：在数据从内核到进程缓冲区期间，进程阻塞于recvfrom调用。相反异步I/O模型在这两个阶段都要处理。从而不同于其他4种模型。

### 同步I/O和异步I/O对比

* 同步I/O操作导致请求进程阻塞，直到I/O操作完成
* 异步I/O操作不导致请求进程阻塞

![6-6.png](6-6.png)

前4种模型——阻塞式I/O模型、非阻塞式I/O模型、I/O复用模型和信号I/O模型都是同步I/O模型，因为其中真正的I/O操作（recvfrom）将阻塞进程。只有异步I/O* 模型与POSIX定义的异步I/O相匹配。


## select函数

该函数允许进程指示内核等待多个事件中的任何一个发生，并且只有在一个或多个事件发生或经理一段指定时间后才唤醒它。

{% highlight c++ %}

#include<sys/select.h>
#inlcude<sys/time.h>

int select(int maxfdp1,fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);

//若有就绪描述符返回数目，超时则为0，出错返回-1
{%  endhighlight %}

最后一个参数timeout，它告知内核所等待描述符中的任何一个就绪可以花多长时间。其timeval结构用于指定这段时间的秒数和微秒数。

{% highlight c++ %}

struct timeval
{
  long tv_sec;
  long tv_usec;
}

{% endhighlight %}

这个参数有三种可能：

* 永远等待下去，仅在有一个描述符准备好时，为此，我们把该参数设置为空指针
* 等待一段固定时间，在有一个描述符准备好I/O时返回，但是不超过由该参数所指向的timeval结构中指定的秒数和微秒
* 根本不等待，检查描述符后立即返回，这称为轮询。为此，该参数必须指向一个timeval结构，而且其中的定时器值必须为0.

中间的三个参数readset，writeset和exceptset指定我们要让内核测试读、写和异常条件的描述符。目前支持的异常条件只有两个：

* 某个套接字的带外数据到达
* 某个已设置为分组模式的伪终端存在可从其主端读取的控制状态信息

描述符集通常是一个整数数组，其中每个整数中的每一位对应一个描述符。

{% highlight c++ %}

void FD_ZERO(fd_set *fdset);
void FD_SET(int fd, fd_set *fdset);
void FD_CLR(int fd, fd_set *fdset);
void FD_ISSET(int fd, fd_set *fdset);

{% endhighlight %}

分配一个fd_set数据类型的描述符集，并用宏设置或测试该集合中的每一位。描述符集的初始化十分重要，因为作为一个自动变量分配的一个描述符集如果没有初始化，那么可能发生不可预期的结果。

select函数中间的三个参数readset，writeset和exceptset中，如果我们对某个条件不感兴趣，就可以把他设为空指针。

maxfdp1指定待测试描述符个数，它的值是待测试的最大描述符加1.描述符0,1,2,3...一直到maxfdp1-1均将被测试。

select函数修改由指针readset、writeset和exceptset所指向的描述符集。因而这三个参数都是值——结果参数。调用该函数时，指定我们关心的描述符值，该函数返回时，结果将指示哪些描述符已就绪。该函数返回时，使用FD_ISSET宏来测试fd_set数据类型中的描述符。描述符集内任何与未就绪描述符对应的位返回时均清成0.为此，每次调用select函数时，我们都得再次把所有描述符集内所关心的位均置为1.

### 描述符就绪条件

满足下列四个条件中的任何一个时，一个套接字准备好读

* 该套接字接收缓冲区中的数据字节数大于等于套接字接收缓冲区低水平标记的当前大小。对这样的套接字执行读操作不会阻塞并返回一个大于0的值。我们可以使用SO_RCVLOWAT套接字选项设置该套接字的低水位标记
* 该连接的读半部关闭。对这样的套接字的读操作将不阻塞并返回0
* 该套接字是一个监听套接字且已完成的连接数不为0。对这样的套接字accept通常不会阻塞。
* 其上有一个套接字错误待处理。对这样套接字的读操作将不阻塞并返回-1.

下列四个条件中的任何一个满足时，一个套接字准备好写

* 该套接字发送缓冲区中的可用空间字节数大于等于套接字发送缓冲区低水平位的当前大小，并且该套接字已连接或不需要连接。如果我们将这样的套接字设置成非阻塞，写操作将不阻塞并返回一个正值。我们可以使用SO_SNDLOWAT套接选项来设置该套接字的低水平位标记。对于TCP和UDP套接字而言，其默认值通常为2048
* 该连接的写半部关闭。对这样的套接字的写操作将产生SIGPIPE信号
* 使用非阻塞式connec的套接字已建立连接，或者connect以失败告终
* 其上有一个套接字错误待处理

如果一个套接字存在带外数据或者仍处于带外标记，那么它由异常条件待处理。

当套接字发生错误时，它将由select标记为既可读又可写。

接收低水平位标记和发送低水平位标记的目的在于：允许应用进程控制在select返回可读或可写条件之前有多少数据可读或多大空间可写。

![6-7.png](6-7.png)

### select的最大描述符

仅仅修改\<sys/types.h\>中的FD_SETSIZE不起作用，需要重新编译内核

## str_cli函数（修订版）

{% highlight c++ %}

void str_cli(FILE *fp, int sockfd)
{
    fd_set readset;
    int count;
    char sendline[MAXLEN];
    char recvline[MAXLEN];

    for (;;)
    {
        //每次调用select前设置关心的套接字描述符
        FD_ZERO(&readset);
        FD_SET(sock_fd, &readset);
        FD_SET(fileno(fp), &readset);
        //取得关心最大套接字 + 1
        int maxfd = (fileno(fp) > sock_fd ? fileno(fp) : sockfd) + 1;
        count = select(maxfd, &readset, NULL, NULL, NULL);
        //是否被设置
        if (FD_ISSET(fileno(fp), &readset))
        {
            if (fgets(sendline, MAXLEN, fp) == NULL)
              return;
            write(sockfd, sendline, strlen(sendline));
        }
        if (FD_ISSET(sockfd, &readset))
        {
            if(readline(sockfd, recvline, MAXLEN) == 0)
            {
                perror("recive FIN");
                exit(0);
            }
            fputs(recvline, stdout);
        }
    }
}

{% endhighlight  %}

程序不会阻塞fgets而没有及时收到FIN

## 批量输入

select函数只从系统调用角度关心读写缓冲区，如果混用了使用了自有缓冲区的函数，可能出现大量数据已经存入自有缓冲区，而函数只返回一部分给程序。但是select并不关心函数自有缓冲区的内容，不会有再有读写状态返回。

## shutdown函数

终止网络连接的的通常方法是调用close函数。不过close有两个限制，却可以使用shutdown来避免。

* close把描述符的引用计数减1，仅在该计数变为0时才关闭套接字。使用shutdown可以不管引用计数就激发TCP的正常连接终止序列。
* close终止读和写两个方向的数据传送。

{% highlight c++ %}

#include<sys/socket.h>
int shutdown(int sockfd, int howto);

{% endhighlight %}

该函数的行为依赖于howto参数的值：

* SHUT_RD关闭连接读的这一半——套接字中不在有数据可以接收，而且套接字缓冲区中的现有数据都被丢弃。
* SHUT_WR关闭连接写的这一半——对于TCP套接字，这称为半关闭。当前留在套接字发送缓冲区的数据将被发送掉，后跟TCP的正常终止序列。
* SHUWRDWR连接的读半部和写半部都关闭——这与调用shutdown两次等效：第一次调用指定SHUT_RD，第二次调用指定SHUT_WR

## str_cli函数（再修订版）

{% highlight c++ %}

void str_cli(FILE *fp, int sockfd)
{
    fd_set readset;
    int count, maxfd, stdineof;
    char sendline[MAXLEN];
    char recvline[MAXLEN];
    FD_ZERO(&readset);
    stdineof = 0;
    for (;;)
    {
        //每次调用select前设置关心的套接字描述符
        FD_SET(sock_fd, &readset);
        FD_SET(fileno(fp), &readset);
        //取得关心最大套接字 + 1
        maxfd = (fileno(fp) > sock_fd ? fileno(fp) : sockfd) + 1;
        count = select(maxfd, &readset, NULL, NULL, NULL);
        //是否被设置
        if (FD_ISSET(fileno(fp), &readset))
        {   //不在使用自有缓冲区的函数
            if (read(fielno(fp),sendline, MAXLEN) == 0)
            {
                stdineof = 1;
                shutdown(sockfd, SHUT_WR);
                FD_CLR(fileno(fp), &readset);
                continue;
            }
            write(sockfd, sendline, strlen(sendline));
        }
        if (FD_ISSET(sockfd, &readset))
        {
            if(read(sockfd, recvline, MAXLEN) == 0)
            {
                if(stdineof == 1)
                  return ;
                else
                {
                    perror("server terminate prematurely");
                    exit(-1);
                }
            }
            write(fileno(stdout),recvline, strlen(recvline));
        }
    }
}

{% endhighlight %}

## TCP回射服务器程序（修订版）

{% highlight c++ %}

int main()
{
    int listenfd,maxfd,clifd,maxi,i,n;
    struct sockaddr_in servaddr,cliaddr;
    socklen_t len;
    fd_set readset,allset;
    int count,client[FD_SETSIZE];
    char buf[MAXLEN];

    memset(servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);

    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    bind(listenfd, (const struct sockaddr*) &servaddr, sizeof(servaddr));
    listen(listenfd, LISTENQ);

    maxfd = listenfd;
    maxi = -1;
    FD_ZERO(&allset);
    FD_SET(listenfd, &allset);

    for(;;)
    {
        readset = allset
        count = select(maxfd + 1,&readset, NULL,NULL,NULL);
        if(FD_ISSET(listenfd, &readset))
        {
            len = sizeof(cliaddr);
            clifd = accept(listenfd,(struct sockaddr*) &cliaddr, &len);
            for(i = 0; i< FD_SETSIZE; i++)
            {
                //找到第一个未被使用的用来存储已连接套接字
                if(client[i] < 0)
                {
                  client[i] = clifd;
                  break;
                }
            }
            if (i == FD_SETSIZE)
            {
                perror("too many client");
                exit(-1);
            }
            FD_SET(clifd, &allset);
            if(clifd > maxfd)
              maxfd = clifd;
            if(maxi < i)
              maxi = i;
            //只有监听套接字有事件发生
            if(--count <= 0 )
                continue;
        }
        for(i = 0; i <= maxi; i++)
        {
            if(client[i] < 0)
              continue;
            if(FD_ISSET(sockfd,&readset))
            {
                //改成不使用自有缓存的函数
                if((n = readn(client[i], buf, MAXLEN)) == 0)
                {
                    close(client[i]);
                    FD_CLR(sockfd,&allset);
                    client[i] = -1;
                }
                else
                {
                    writen(client[i], buf, n);
                }

                if(--count <= 0)
                  break;
            }
        }
    }
    return 0;
}

{% endhighlight %}

**拒绝服务型攻击**

当一个服务器在处理多个客户时，它绝对不能阻塞于只与单个客户相关的某个函数调用。否则可能导致服务器被挂起，拒绝为所有其他客户提供服务。这就是所谓的拒绝服务型攻击。可能的解决办法包括：

* 使用非阻塞式I/O
* 让每个客户由单独的线程提供服务
* 对I/O操作设置一个超时

## pselect函数

{% highlight c++ %}

#include<sys/select.h>
#include<signal.h>
#include<time.h>

int pselect(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set exceptset, const struct timespec *timeout, const sigset_t *sigmask);
//如有就绪描述符则为其数目，若超时则为0，否则为-1

{% endhighlight %}

pselect相对于select有两个变化：

* pselect使用timespec结构，而不是用timeval结构。
* pselect函数增加了第六个参数：一个指向信号掩码的指针。该参数允许程序先禁止递交某些信号，再测试由这些当前被禁止信号的处理函数设置的全局变量，然后调用pselect，告诉他重新设置信号掩码。

## poll函数

{% highlight c++ %}

#include<poll.h>
int poll(struct pollfd *fdarray, unsigned long nfds, int timeout);
//若有就绪描述符则为其数目，若超时则为0，若出错则为-1

struct pollfd
{
  int fd;
  short events;
  short revents;
};
{% endhighlight %}

第一个参数是指向一个结构数组第一个元素的指针，每个数组元素都是一个pollfd结构，用于指定测试某个描述符的条件。

![6-8.png](6-8.png)

poll识别三类数据：普通，优先级带和高优先级

POSIX在其poll的定义中留了许多空洞

* 所有正规TCP数据和所有UDP数据都被认为是普通数据
* TCP的带外数据被认为是优先级带数据
* 当TCP连接的读半部关闭时，也被认为是普通数据，随后的读操作返回0
* TCP连接存在错误既可认为是普通数据，也可认为是错误。无论那种情况，随后的读操作将返回-1，并把errno设置成合适的值
* 在监听套接字上有新的连接可用既可认为是普通数据，也可以认为是优先级数据。大多数实现视之为普通数据。
* 非阻塞式connect的完成被认为是使相应的套接字可写


timeout参数指定poll函数返回前等待多长时间，

![6-9.png](6-9.png)

当发生错误时，poll函数返回值-1，若定时器到时之前没有任何描述符就绪，这返回0。否则返回就绪描述符的个数。

{% highlight c++ %}


int main()
{
    int i, maxi, listenfd, connfd, sockfd;
    int nready;
    ssize_t n;
    char buf[MAXLEN];
    socklen_t clilen;
    struct pollfd client[OPEN_MAX];
    struct sockaddr_in cliaddr, servaddr;

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));

    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);

    bind(listenfd, (const struct sockaddr*) & servaddr, sizeof(servaddr));
    listen(listen, LISTENQ);

    client[0].fd = listenfd;
    client[0].events = POLLRDNORM;
    for(i = 0;i < OPEN_MAXl; i++)
        client[i].fd = -1;
    maxi = 0;

    for(;;)
    {
        nready = poll(client, maxi + 1, INFTIME);

        if(client[0].revents & POLLRDNORM)
        {
            clilen = sizeof(cliaddr);
            connfd = accept(listenfd, (struct sockaddr*) &cliaddr, &clilen);

            for(i = 0; i < OPEN_MAX; i++)
              if(client[i].fd < 0)
              {
                client[i].fd = connfd;
                break;
              }
            if(i == OPEN_MAX)
            {
                perror("too many clients");
                exit(-1);
            }
            client[i].events = POLLRDNORM;
            if(maxi < i)
              maxi = i;
            if(--nready <= 0)
              continue;
        }
        for(i = 1; i < maxi; i++)
        {
            if((sockfd = client[i].fd) < 0)
              continue;
            if(client[i].revents & (POLLRDNORM | POLLERR))
            {
                if((n = read(sockfd, buf, MAXLEN)) < 0)
                {
                    if(errno == ECONNREST)
                    {
                        close(sockfd);
                        client[i].fd = -1
                    }
                    else
                    {
                        perror(read error);
                        exit(-1);
                    }
                }
                else if(n == 0)
                {
                    close(sockfd);
                    client[i].fd = -1;
                }
                else
                  writen(sockfd, buf, n);
                if(--nready <= 0)
                  break;
            }
        }
    }
    return 0;
}

{% endighlight %}