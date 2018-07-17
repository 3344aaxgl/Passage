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

{% highlight c++ %}


{% endhighlight %}