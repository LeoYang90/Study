# IO 复用：select 和 poll 函数

## 1、什么是 IO 复用？

内核一旦发现进程所指定的一个或多个 IO 条件就绪（输入已经准备好，或者描述符可以承接更多的输出），内核就通知进程，这个能力被称为 IO 复用。

## 3、信号驱动 IO 与 异步 IO 的区别是什么？

- 信号驱动式 IO 是由内核告诉我们，何时进行 IO 操作
- 异步 IO 是由内核告诉我们， IO 操作何时完成

## 4、select 函数返回后如何获取已就绪的描述符？

- `select` 函数会返回所有描述符集的已就绪的总数。如果定时器到时，则返回 0.出错返回 -1
- `select` 函数返回后，我们可以使用 `FD_ISSET` 来测试 `fd_set` 描述符。描述符集内任何与未就绪描述符对应的位返回时均清成 0。因此，每次重新调用 `select` 函数，我们都得重新把所有描述符集内所关心的位均置为 1

## 5、描述符就绪的条件是什么？

- 读就绪
    - 套接字接受缓存区中的数据字节大于等于套接字接受缓冲区低水位大小
    - 连接的读半部关闭（接受了 FIN 的 TCP 连接）
    - 套接字是一个监听套接字并且已完成的连接数不为 0 （accept 不会阻塞）
    - 有一个套接字错误要处理

- 写就绪
    - 套接字的发送缓冲区中可用空间字节数大于套接字发送缓冲区低水位
    - 连接的写半部已关闭。对这样的套接字写操作会造成 `SIGPIPE` 信号
    - 使用非阻塞 connect 的套接字已建立连接，或者 connect 已经失败
    - 套接字错误 

## 6、select 函数的缺陷有哪些？

- 最大并发数限制：`select` 对打开的文件描述符是有限制的,由FD_SETSIZE(1024)设置, 如果要改变FD_SIZE的大小需要重新编译内核
- 效率低：调用 `select` 时，需要将 `readfds`, `writefds`, `exceptfds` 从用户层拷贝到内核。`select` 返回后，需要线性遍历套接字集合（`readfds, writefds, exceptfds`）。
- 每次都要重新初始化 `fd_set`
- 如果其中某个描述符被关闭了，`select` 并不会告诉我们是哪个描述符，我们需要在描述符上进行 IO 操作并检查错误来判断

## 7、`str_cli` 基础程序中，为什么不能用 `Readline`、`Fgets` 等函数？

- `Fgets` 函数在客户端输入 `EOF` 后，立刻返回到 `main` 函数，随即进程终止。然而，标准输入中的 `EOF` 并不意味着我们完成了套接字的读入：可能仍有请求在去往服务器的路上，或者仍有应答在返回客户的路上。（需要 `shutdown` 函数告诉服务端已完成了数据传送）
- `Readline`、`Fgets` 等函数存在缓存，用户很可能有不完整的输入行，也可能有一个或多个完整的输入行。（需要替换为 `read` 无缓存函数）


## 8、`pselect` 函数与 `select` 函数有什么区别？

- 超时值使用的结构不一样，`pselect` 使用的是 `timespec` 结构

- `pselect`的超时值被声明为 `const`，保证了调用 `pselect` 不会改变此值

- `pselect` 可使用可选信号屏蔽字，在调用 `pselect` 时，以原子操作的方式安装该信号屏蔽字。在返回时，恢复以前的信号屏蔽字。

## 9、`poll` 函数的 `events/revents` 参数选项有哪些？

![](http://owql68l6p.bkt.clouddn.com/poll_events.png) 

## 10、`poll` 函数描述符就绪之后，`revents` 会返回什么状态？

- 非阻塞式 `connect` 的完成被认为是套接字可写
- 监听套接字上有新的连接可用既可认为是普通数据，也可认为是优先级数据
- 正规 TCP 数据和 UDP 数据都被认为是普通数据
- TCP 带外数据被认为是优先级带数据
- TCP 读半部关闭时，被认为是普通数据
- TCP 连接错误既可认为是普通数据，也可认为是错误

## 11、select 和 poll 的缺点

- 每次调用 `select` 和 `poll`，内核都要检查所有的文件描述符
- 需要将 `readfds`, `writefds`, `exceptfds` 从用户层拷贝到内核。`select` 返回后，需要线性遍历套接字集合（`readfds, writefds, exceptfds`）。
- 函数返回后，程序必须检查返回的数据结构中的每个元素

## 12、如何为套接字的 IO 操作设置超时？

- 调用 alarm
- 在 select 中阻塞等待 IO
- 使用 `SO_RCVTIMEO` 和 `SO_SNDTIMEO` 套接字选项

以上三种办法均可适用于输入与输出操作，`select` 函数适用于非阻塞的 `connect` 操作，对于阻塞式，只要第一种方法适用，而且

- alarm 设置超时只能指定一个小于 75s 的数值
- 注意信号处理的自动重启功能


## 13、什么是水平触发和边缘触发？在 IO 处理中有什么不同？

- 水平触发：如果文件描述符上可以无阻塞地进行 IO 系统调用，此时认为它已经就绪
- 边缘触发：如果文件描述符自上次状态检查以来发生了新的 IO 活动，此时需要触发通知

对于水平触发：

- 我们可以在任何时刻来检查文件描述符的就绪状态，这表明当我们确定文件描述符处于就绪态时，就可以执行一些 IO 操作，然后重复检查，看看是否仍然处于就绪态，没有必要尽可能多的执行 IO

对于边缘触发：

- 在接收到一个 IO 就绪的事件通知后，程序在某个时刻应该在相应的文件描述符上尽可能多地执行 IO（比如尽可能多读取字节）
- 每个被检查的文件描述符都应该设置为非阻塞模式

`epoll` 的帮助中指出，使用 `ET` 模式，可以便捷的处理 `EPOLLOUT` 事件，省去打开与关闭 `EPOLLOUT` 的`epoll_ctl`（`EPOLL_CTL_MOD`）调用。从而有可能让你的性能得到一定的提升。  例如你需要写出1M的数据，写出到`socket` 256k时，返回了`EAGAIN`，`ET` 模式下，当再次返回 `EPOLLOUT` 时，继续写出待写出的数据，当没有数据需要写出时，不处理直接略过即可。而 `LT` 模式则需要先打开 `EPOLLOUT`，当没有数据需要写出时，再关闭 `EPOLLOUT`（否则会一直会返回 `EPOLLOUT` 事件）  总体来说，`ET` 处理 `EPOLLOUT` 方便高效些，`LT` 不容易遗漏事件、不易产生 `bug`

## 14、epoll 的优点？

- 支持一个进程打开大数目的socket描述符
-  IO效率不随FD数目增加而线性下降. 传统select/poll的另一个致命弱点就是当你拥有一个很大的socket集合，由于网络得延时，使得任一时间只有部分的socket是"活跃" 的，而select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进 行操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。于是，只有"活跃"的socket才会主动去调用 callback函数，其他idle状态的socket则不会，在这点上，epoll实现了一个"伪"AIO，因为这时候推动力在os内核。在一些 benchmark中，如果所有的socket基本上都是活跃的---比如一个高速LAN环境，epoll也不比select/poll低多少效率，但若 过多使用的调用epoll_ctl，效率稍微有些下降。然而一旦使用idle connections模拟WAN环境，那么epoll的效率就远在select/poll之上了。

- 使用mmap加速内核与用户空间的消息传递。这点实际上涉及到epoll的具体实现。无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就显 得很重要，在这点上，epoll是通过内核于用户空间mmap同一块内存实现的。而如果你像我一样从2.5内核就开始关注epoll的话，一定不会忘记手 工mmap这一步的。

## 15、`epoll_ctl` 函数中 `op` 参数的选项有哪些？

- `EPOLL_CTL_ADD` 将描述符 fd 添加到 epfd 中的兴趣列表中
- `EPOLL_CTL_MOD` 修改描述符 fd 上设定的事件
- `EPOLL_CTL_DEL` 将 fd 从 epfd 的兴趣列表中删除

## 16、`epoll_event` 中的 `events` 选项有哪些？

- EPOLLIN：表示对应的文件描述符可以读；
- EPOLLOUT：表示对应的文件描述符可以写；
- EPOLLPRI：表示对应的文件描述符有紧急的数据可读；
- EPOLLERR：表示对应的文件描述符发生错误；
- EPOLLHUP：表示对应的文件描述符被挂断；
- EPOLLRDHUP：表示对应的文件描述符对端关闭；

- EPOLLET: 采用边缘触发通知模式。
- EPOLLONESHOT: 在完成事件通知之后禁用检测，即我们希望在某个特定的文件描述符上只得到一次通知


## 17、`epoll_event` 中的 `epoll_data_t` 选项有哪些？

```
struct union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
}

```

## 18、采用边缘触发模式时如何避免文件描述符饥饿现象？

当我们尝试通过非阻塞式的读操作将所有的输入都读取的时候，其他的文件描述符将会有处于饥饿状态的风险存在。解决方案是让应用程序维护一个列表，存放着已经被通知为就绪态的文件描述符：

- 调用 `epoll_wait` 监视文件描述符，并将处于就绪态的文件描述符添加到程序维护的列表中去。如果这个文件描述符已经存在，那么这次监视操作的超时时间应该设置为较小值或者是 0。这样如果没有新的文件描述符就绪，应用程序就可以迅速进行到下一步，去处理那些已经处于就绪态的文件描述符
- 在应用程序维护的列表中，在那些列表中的文件描述符上进行一定限度的 IO 操作，例如轮转调度。当文件描述符出现 EAGAIN 或者 EWOULDBLOCK 错误时，就可以将其从列表中移除

## `str_cli` 函数基础程序

```

void str_cli(FILE *fp, int sockfd)
{
    int            maxfdp1;
    fd_set        rset;
    char        sendline[MAXLINE], recvline[MAXLINE];

    FD_ZERO(&rset);
    for ( ; ; ) {
        FD_SET(fileno(fp), &rset);
        FD_SET(sockfd, &rset);
        maxfdp1 = max(fileno(fp), sockfd) + 1;
        Select(maxfdp1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(sockfd, &rset)) {    /* socket is readable */
            if (Readline(sockfd, recvline, MAXLINE) == 0)
                err_quit("str_cli: server terminated prematurely");
            Fputs(recvline, stdout);
        }

        if (FD_ISSET(fileno(fp), &rset)) {  /* input is readable */
            if (Fgets(sendline, MAXLINE, fp) == NULL)
                return;        /* all done */
            Writen(sockfd, sendline, strlen(sendline));
        }
    }
}

```

## `str_cli` 函数改进版

```
#include    "unp.h"

void
str_cli(FILE *fp, int sockfd)
{
    int            maxfdp1, stdineof;
    fd_set        rset;
    char        buf[MAXLINE];
    int        n;

    stdineof = 0;
    FD_ZERO(&rset);
    for ( ; ; ) {
        if (stdineof == 0)
            FD_SET(fileno(fp), &rset);
        FD_SET(sockfd, &rset);
        maxfdp1 = max(fileno(fp), sockfd) + 1;
        Select(maxfdp1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(sockfd, &rset)) {    /* socket is readable */
            if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
                if (stdineof == 1)
                    return;        /* normal termination */
                else
                    err_quit("str_cli: server terminated prematurely");
            }

            Write(fileno(stdout), buf, n);
        }

        if (FD_ISSET(fileno(fp), &rset)) {  /* input is readable */
            if ( (n = Read(fileno(fp), buf, MAXLINE)) == 0) {
                stdineof = 1;
                Shutdown(sockfd, SHUT_WR);    /* send FIN */
                FD_CLR(fileno(fp), &rset);
                continue;
            }

            Writen(sockfd, buf, n);
        }
    }
}

```

## 回射服务端程序

```
int main(int argc, char **argv)
{
    int                    i, maxi, maxfd, listenfd, connfd, sockfd;
    int                    nready, client[FD_SETSIZE];
    ssize_t                n;
    fd_set                rset, allset;
    char                buf[MAXLINE];
    socklen_t            clilen;
    struct sockaddr_in    cliaddr, servaddr;

    listenfd = Socket(AF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family      = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port        = htons(SERV_PORT);

    Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

    Listen(listenfd, LISTENQ);

    maxfd = listenfd;            /* initialize */
    maxi = -1;                    /* index into client[] array */
    for (i = 0; i < FD_SETSIZE; i++)
        client[i] = -1;            /* -1 indicates available entry */
    FD_ZERO(&allset);
    FD_SET(listenfd, &allset);
/* end fig01 */

/* include fig02 */
    for ( ; ; ) {
        rset = allset;        /* structure assignment */
        nready = Select(maxfd+1, &rset, NULL, NULL, NULL);

        if (FD_ISSET(listenfd, &rset)) {    /* new client connection */
            clilen = sizeof(cliaddr);
            connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
#ifdef    NOTDEF
            printf("new client: %s, port %d\n",
                    Inet_ntop(AF_INET, &cliaddr.sin_addr, 4, NULL),
                    ntohs(cliaddr.sin_port));
#endif

            for (i = 0; i < FD_SETSIZE; i++)
                if (client[i] < 0) {
                    client[i] = connfd;    /* save descriptor */
                    break;
                }
            if (i == FD_SETSIZE)
                err_quit("too many clients");

            FD_SET(connfd, &allset);    /* add new descriptor to set */
            if (connfd > maxfd)
                maxfd = connfd;            /* for select */
            if (i > maxi)
                maxi = i;                /* max index in client[] array */

            if (--nready <= 0)
                continue;                /* no more readable descriptors */
        }

        for (i = 0; i <= maxi; i++) {    /* check all clients for data */
            if ( (sockfd = client[i]) < 0)
                continue;
            if (FD_ISSET(sockfd, &rset)) {
                if ( (n = Read(sockfd, buf, MAXLINE)) == 0) {
                        /*4connection closed by client */
                    Close(sockfd);
                    FD_CLR(sockfd, &allset);
                    client[i] = -1;
                } else
                    Writen(sockfd, buf, n);

                if (--nready <= 0)
                    break;                /* no more readable descriptors */
            }
        }
    }
}
/* end fig02 */

```

## `poll` 改进版服务端程序

```
int main(int argc, char **argv)
{
    int                    i, maxi, listenfd, connfd, sockfd;
    int                    nready;
    ssize_t                n;
    char                buf[MAXLINE];
    socklen_t            clilen;
    struct pollfd        client[OPEN_MAX];
    struct sockaddr_in    cliaddr, servaddr;

    listenfd = Socket(AF_INET, SOCK_STREAM, 0);

    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family      = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port        = htons(SERV_PORT);

    Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));

    Listen(listenfd, LISTENQ);

    client[0].fd = listenfd;
    client[0].events = POLLRDNORM;
    for (i = 1; i < OPEN_MAX; i++)
        client[i].fd = -1;        /* -1 indicates available entry */
    maxi = 0;                    /* max index into client[] array */
/* end fig01 */

/* include fig02 */
    for ( ; ; ) {
        nready = Poll(client, maxi+1, INFTIM);

        if (client[0].revents & POLLRDNORM) {    /* new client connection */
            clilen = sizeof(cliaddr);
            connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
#ifdef    NOTDEF
            printf("new client: %s\n", Sock_ntop((SA *) &cliaddr, clilen));
#endif

            for (i = 1; i < OPEN_MAX; i++)
                if (client[i].fd < 0) {
                    client[i].fd = connfd;    /* save descriptor */
                    break;
                }
            if (i == OPEN_MAX)
                err_quit("too many clients");

            client[i].events = POLLRDNORM;
            if (i > maxi)
                maxi = i;                /* max index in client[] array */

            if (--nready <= 0)
                continue;                /* no more readable descriptors */
        }

        for (i = 1; i <= maxi; i++) {    /* check all clients for data */
            if ( (sockfd = client[i].fd) < 0)
                continue;
            if (client[i].revents & (POLLRDNORM | POLLERR)) {
                if ( (n = read(sockfd, buf, MAXLINE)) < 0) {
                    if (errno == ECONNRESET) {
                            /*4connection reset by client */
#ifdef    NOTDEF
                        printf("client[%d] aborted connection\n", i);
#endif
                        Close(sockfd);
                        client[i].fd = -1;
                    } else
                        err_sys("read error");
                } else if (n == 0) {
                        /*4connection closed by client */
#ifdef    NOTDEF
                    printf("client[%d] closed connection\n", i);
#endif
                    Close(sockfd);
                    client[i].fd = -1;
                } else
                    Writen(sockfd, buf, n);

                if (--nready <= 0)
                    break;                /* no more readable descriptors */
            }
        }
    }
}
/* end fig02 */

```

## UNIX 下的 IO 模型有哪些？

- 阻塞式 IO

![](http://owql68l6p.bkt.clouddn.com/%E9%98%BB%E5%A1%9E%E5%BC%8F.png)

- 非阻塞式 IO
![](http://owql68l6p.bkt.clouddn.com/%E9%9D%9E%E9%98%BB%E5%A1%9E%E5%BC%8F.png)

- IO 复用
![](http://owql68l6p.bkt.clouddn.com/io%20%E5%A4%8D%E7%94%A8.png)

- 信号驱动式 IO
![](http://owql68l6p.bkt.clouddn.com/%E4%BF%A1%E5%8F%B7%E9%A9%B1%E5%8A%A8.png)

- 异步 IO
![](http://owql68l6p.bkt.clouddn.com/%E5%BC%82%E6%AD%A5%20io.png)

## API

```
int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout)

void FD_ZERO(fd_set *fdset)
void FD_SET(fd_set *fdset)
void FD_CLR(fd_set *fdset)
void FD_ISSET(fd_set *fdset)

int shutdown(int sockfd,int howto)//SHUT_RD、SHUT_WD

int pselect(int maxfdp1,fd_set *restrict readfds,fd_set *restrict writefds,fd_set *restrict exceptfds,const struct timespec *restrict tsptr,const sigset_t *restrict sigmask);

int poll(struct pollfd fdarray[],nfds_t nfds,int timeout);
struct pollfd {
        int   fd;         /* file descriptor */
        short events;     /* requested events */
        short revents;    /* returned events */
};

int epoll_create(int size)
int epoll_ctr(int epfd, int op, int fd, struct epoll_event *ev)
int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout)
struct union epoll_data {
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
}
``` 