---
layout: post
title:  "基本UDP套接字编程"
date:   2018-08-05 20:15:51 +0800
categories: unix_network_programming
tags: c++
description: unix network programming读书笔记
---

## recvfrom和sendto函数

{% highlight c++ %}

#include<sys/socket.h>
ssize_t recvfrom(int sockfd, void * buff, size_t nbytes, int flags,struct sockaddr* from, socklen_t *addrlen);

ssize_t sendto(int sockfd, const void* buff, size_t nbytes, int flags, const struct sockaddr* to, socklen_t addrlen);
//若成功则为读或写的字节数，若出错则为-1

{% endhighlight %}

前三个参数sockfd，buff和nbytes等同于read和write函数的三个参数：描述符，指向读入或写出缓冲区的指针和读写字节数。

sendto的to参数指向一个含有数据报接收者的协议地址的套接字地址结构，其大小由addrlen参数指定。recvfrom的from参数指向一个将由该函数在返回时填写数据报发送者的协议地址的套接字地址结构，而在该套接字地址结构中填写的字节数则放在addrlen参数所指的整数中返回给调用者。

recvfrom和sendto可以用于TCP

## UDP回射服务器程序：main函数

{% highlight c++ %}

void dg_echo(int fd, const struct *cliaddr, socklen_t clilen)
{
    int n;
    socklen_t len;
    char mesg[MAXLEN];

    for(;;)
    {
        len = clilen;
        n = recvfrom(sockfd, mesg, MAXLEN, 0, cliaddr, &len);

        sendto(sockfd, mesg, MAXLEN, 0, cliaddr, len);
    }
}

int main(int argc, char* argv[])
{
    int sockfd;
    struct sockaddr_in servaddr, cliaddr;

    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serveraddr.sin_port = htons(SERV_PORT);

    bind(sockfd, (const struct sockaddr*) &servaddr, sizeof(servaddr));

    dg_echo(sockfd, (struct sockaddr*) &cliaddr, sizeof(cliaddr));
}
{% endhighlight %}


## UDP回射客户程序：main函数

{% highlight c++ %}

void dh_cli(FILE* fp, int fd,const struct sockaddr* pservaddr, socklen_t servlen)
{
    int n;
    char sendline[MAXLEN], recvline[MAXLEN + 1];
    while(fgets(sendline, MAXLEN, fp) != NULL)
    {
        sendto(fd, sendline, strlen(sendline), 0, &servaddr, servlen);
        n = recvfrom(fd, recvline, MAXLEN, 0, NULL, NULL);

        recvline[n] = 0;
        fputs(recvline, stdout);
    }

}

int main(int argc, char *argv[])
{
    struct servaddr;
    int sockfd;
    socklen_t socklen;

    if(argc != 2)
    {
        perror("useage:udpcli <IPaddress>");
        exit(-1);
    }

    sockfd = socket(AF_INET, SO_DGRAM, 0);
    bzero(&servaddr, sizeof(servaddr));

    //点分十进制转二进制
    inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);

    dg_cli(stdin,sockfd, &servaddr, sizeof(servaddr));

    return 0;
}

{% endhighlight %}

## 数据报的丢失

UDP客户/服务器是不可靠的，如果一个客户数据报丢失，客户将永远阻塞于dg_cli函数中的recvfrom调用，等待一个永远不会到达的服务器应答。

## 验证接收到的响应

{% highlight c++ %}

void dh_cli(FILE* fp, int fd,const struct sockaddr* pservaddr, socklen_t servlen)
{
    int n;
    struct sockaddr preply_addr; 
    char sendline[MAXLEN], recvline[MAXLEN + 1];
    socklen_t len;
    while(fgets(sendline, MAXLEN, fp) != NULL)
    {
        sendto(fd, sendline, strlen(sendline), 0, &servaddr, servlen);
        //n = recvfrom(fd, recvline, MAXLEN, 0, NULL, NULL);
        len = socklen;
        n = recvfrom(fd, recvline, MAXLEN, 0, &preply_addr, &len);
        if(len != servlen || memcmp(&preply_addr, pservaddr, len))
        {
            printf("reply from %s (ignored)\n", sock_ntop(&preply_addr, len));
            continue;
        }
        recvline[n] = 0;
        fputs(recvline, stdout);
    }
}

{% endhighlight %}

服务器没有在其套接字上绑定一个实际的IP地址，因此内核将为这些应答的IP数据报选择源地址。选为源地址的是外出接口的主IP地址。

一个解决办法是：

* 得到由recvfrom返回的IP地址后，客户通过在DNS中查找服务器主机的名字来验证该主机的域名。
* UDP服务器给服务器主机上配置的每个IP地址上创建一个套接字，用bind捆绑每个IP地址到各自的套接字，然后在这些所有的套接字上使用select，再从可读的套接字给出应答。

## 服务器进程未运行

对于UDP套接字，由它引发的异步错误却并不返回给它，除非它已连接。因为recvfrom可以返回的信息只有error值，无法接收引起错误数据报的IP首部和UDP首部。

## UDP程序例子小结

客户必须给sendto调用指定的IP地址和端口号。一般来说，客户的IP地址和端口号都由内核自动选择，客户也可以通过调用bind指定。客户的临时端口是在第一次调用sendto时一次性选定，不能改变；然而客户的IP地址却可以随客户发送的每个UDP数据报而变动。

如果客户绑定了一个IP地址到其套接字上，但是内核决定外出的数据报必须从另一个数据链路发出，这种情况下，IP数据报将包含一个不同于外出链路IP地址的源IP地址。

## UDP的connect函数

UDP套接字调用connect，没有三次握手过程。内核只是检查是否存在立即可知的错误，记录对端的IP地址和端口号，然后立即返回到调用进程。

对于已连接的UDP套接字，与默认的未连接UDP套接字相比，发生了三个变化：

* 不能给输出操作指定目的IP地址和端口号。不在使用sendto，而改用write和send。写到已连接套接字上的任何内容都自动发送到由connect指定的协议地址。
* 不必使用recvfrom以获悉数据报的发送者，而改用read、recv或recvmsg。在一个已连接UDP套接字上，由内核为输入操作返回的数据报只有那些来自connect所指定协议地址的数据报。目的地为这个已连接UDP套接字的本地协议地址，发源地却不是该套接字早先connect到的协议地址的数据报，不会投递到该套接字。
* 由已连接UDP套接字引发的异步错误会返回给他们所在的进程，而未连接的UDP套接字不会收到任何异步错误

### 给一个UDP套接字多次调用connect

拥有一个已连接套接字UDP套接字的进程可出于下列两个目的之一再次调用connect：

* 指定新的IP地址和端口号
* 断开套接字，再次调用connect时把套接字地址结构地址族成员设置成AF——UNSPEC，这么做可能会返回一个EAFNOSUPPORT错误。

### 性能

在一个未连接的UDP套接字上给两个数据报调用sendto涉及内核执行下列6个步骤：

* 连接套接字
* 输出第一个数据报
* 断开套接字连接
* 连接套接字
* 输出第二个数据报
* 断开套接字连接

当应用进程知道自己要给同一目的地址发送多个数据报时，显示连接套接字的效率更高。调用connect后调用两次write涉及内核执行如下步骤：

* 连接套接字
* 输出第一个套接字
* 输出第二个套接字

## dg_cli函数（修订版）

{% highlight c++ %}

void dh_cli(FILE* fp, int fd,const struct sockaddr* pservaddr, socklen_t servlen)
{
    int n;
    struct sockaddr preply_addr; 
    char sendline[MAXLEN], recvline[MAXLEN + 1];
    socklen_t len;
    
    connect(fd, pservaddr, servlen);
    while(fgets(sendline, MAXLEN, fp) != NULL)
    {
        write(fd, sendline, strlen(sendline));
        n = read(fd, recvline, MAXLEN);
        recvline[n] = 0;
        fputs(recvline, stdout);
    }
}

{% endhighlight %}

所做的修改是调用cennect，并以write和read调用替代sendto和recvfrom调用。

如果服务器程序没有运行，在第一次发送数据时将返回错误。

## UDP缺乏流量控制

客户端发送2000个1400字节大小的UDP数据报给服务器

{% highlight c++ %}

#define NDG 2000
#define DGLEN 1400

void dg_cli(FILE *fp, int fd, const struct sockaddr *pservaddr,socklen_t servlen)
{
    int i;
    char snedline[DGLEN];
    for(i = 0;i < NDG; i++)
      snedto(sockfd, sendline, DGLEN, 0, pservaddr, servlen);
}

{% endhighlight %}

服务器接收数据报并对其计数

{% highlight c++ %}

static void recvfrom_init(int);
static int count;

void dg_echo(int sockfd,const struct *pcliaddr, socklen_t clilen)
{
    socklen_t len;
    char mesg[MAXLEN];
    
    signal(SIGINT, recvfrom_init);
    for(;;)
    {
        len = clilen;
        recvfrom(sockfd, mesg, MAXLEN, 0, pcliaddr, &len);
        count++;    
    }
}

static void recvfrom_init(int signo)
{
    printf("\nrecived %n datagrams\n", count);
    exit(0);
}

{% endhighlight %}

因为没有流量控制，发送端淹没其接收端是轻而易举之事。可以通过修改接收缓冲区大小来改善此问题。

## UDP中的外出接口确定

已连接的UDP套接字还可以用来确定用于某个特定的目的地的外出接口。这是由connect函数应用到UDP套接字时的一个副作用造成的:内核选择本地IP地址。这个本地地址通过为目的IP地址搜索路由表得到外出接口，然后选用该接口的主IP地址而选定