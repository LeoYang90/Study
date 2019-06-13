# TCP 客户/服务器程序示例

## 1、当遇到僵死子程序时，基本程序应该如何改进？

添加 `SIGCHLD` 信号处理函数。具体步骤为：

- 当 `fork` 子进程的时候，必须捕获 `SIGCHLD` 信号
- 当捕获信号时，必须处理被中断的系统调用
- `SIGCHLD` 的信号处理函数必须正确编写，应使用 `waitpid` 以免留下僵死进程。

```
...
Listen(listenfd, LISTENQ);

Signal(SIGCHLD, sig_chld);
...


...
connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
if (error == EINTR) 
    continue;
...

```

## 2、当服务端子进程终止时，会发生什么？

- 在同一个主机上启动服务器和客户端，找到服务器子进程的进程ID，并执行kill命令杀死它。
- `SIGCHLD` 信号被发送给服务器父进程，并得到正确处理
- 客户端接收来自服务器 `TCP` 的 `FIN` 并向服务端响应了一个 `ACK`, 但是此时客户端正阻塞在 `fgets` 调用上。
- 在客户上再键入一行文本，`str_cli` 调用 `writen`，客户 `TCP` 接着把数据发送给服务器。TCP允许这么做，因为客户TCP接收到FIN只是表示服务器进程已关闭了连接的服务器端，从而不再往其中发送任何数据而已。FIN的接收并没有告知客户TCP服务器进程已经终止。
- 当服务器TCP接收到来自客户的数据时，既然先前打开那个套接字的进程已经终止，于是相应以一个 `RST`
- 客户进程看不到这个 `RST`，因为它在调用 `writen` 后立即调用 `readlind`，并且由于第二步接收的 `FIN`，所调用的`readline` 立即返回 0
- 客户端以出错信息“server terminated prematurely”退出，客户端终止，关闭所有打开的描述符。

当FIN到达套接字时，客户正阻塞在 `fgets` 调用上。客户实际上在应对两个描述符--套接字和用户输入。事实上程序不应该阻塞到两个源中某个特定源的输入上，而是应该阻塞在其中任何一个源的输入上，这正是 `select` 和 `poll` 这两个函数的目的之一。

> 如果客户不理会 `readline` 返回的错误，继续向服务器写入更多数据会发生什么？
> 
> 当一个进程向某个已收到 `RST` 的套接字执行写操作（返回EPIPE错误）时，内核向该进程发送一个 `SIGPIPE` 信号。该信号的默认行为是终止进程，因此进程必须捕获它以免不情愿地被终止。

## 3、当服务端主进程终止了，会发生什么？

- 在不同主机上运行客户和服务器。先启动服务器，再启动客户（键入一行文本以确定连接工作正常），然后从网络上断开服务器主机，在客户上键入另一行文本。
- 当服务器主机崩溃时，已有的网络连接上不发出任何东西。
- 我们在客户上键入一行文本，它由 `writen` 写入内核，再由客户 `TCP` 作为一个数据分节送出。客户随后阻塞于 `readline` 调用。
- 客户 `TCP` 持续重传数据分节，试图从服务器上接收一个 `ACK`。如果在放弃重传前服务器主机没有重新启动，则客户进程返回一个错误。所返回的错误是 `ETIMEOUT`，然后如果某个中间路由器判断服务器主机已不可达，从而相应一个“destination unreachable”ICMP消息，那么所返回的错误是 `EHOSTUNREACH` 或 `ENETUNREACH`

我们发现这种问题只有在客户端向服务端主机非诉讼数据后才能检测到它已经崩溃，如果想要主动检测服务端的崩溃状态，我们需要使用 `SO_KEEPALIVE` 这个套接字选项。

## 4、服务端主机终止后重启，会发生什么？

- 当服务器主机崩溃后重启时，它的TCP丢失了崩溃前的所有连接信息，因此服务器TCP对于所收到的来自客户的数据分节相应一个RST

- 当客户TCP收到该RST时，客户正阻塞与readline调用，导致调用返回ECONNERESET错误


## TCP 服务端基本程序

```
int main(int argc, char **argv)
{
    int                    listenfd, connfd;
    pid_t                childpid;
    socklen_t            clilen;
    struct sockaddr_in    cliaddr, servaddr;

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

        if ( (childpid = Fork()) == 0) {    /* child process */
            Close(listenfd);    /* close listening socket */
            str_echo(connfd);    /* process the request */
            exit(0);
        }
        Close(connfd);            /* parent closes connected socket */
    }
}

void str_echo(int sockfd)
{
    ssize_t        n;
    char        buf[MAXLINE];

again:
    while ( (n = read(sockfd, buf, MAXLINE)) > 0)
        Writen(sockfd, buf, n);

    if (n < 0 && errno == EINTR)
        goto again;
    else if (n < 0)
        err_sys("str_echo: read error");
}
```

## 客户端基本程序

```

int main(int argc, char **argv)
{
    int                    sockfd;
    struct sockaddr_in    servaddr;

    if (argc != 2)
        err_quit("usage: tcpcli <IPaddress>");

    sockfd = Socket(AF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);
    Inet_pton(AF_INET, argv[1], &servaddr.sin_addr);

    Connect(sockfd, (SA *) &servaddr, sizeof(servaddr));

    str_cli(stdin, sockfd);        /* do it all */

    exit(0);
}

void str_cli(FILE *fp, int sockfd)
{
    char    sendline[MAXLINE], recvline[MAXLINE];

    while (Fgets(sendline, MAXLINE, fp) != NULL) {

        Writen(sockfd, sendline, strlen(sendline));

        if (Readline(sockfd, recvline, MAXLINE) == 0)
            err_quit("str_cli: server terminated prematurely");

        Fputs(recvline, stdout);
    }
}
```