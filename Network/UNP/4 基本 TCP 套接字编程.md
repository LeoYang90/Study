# 基本 TCP 套接字编程

## 1、`socket` 函数中 `family`、`type`、`protocol` 等参数的选项有哪些？

```
int socket(int family, int type, int protocal)

```

- `family`
	- `AF_INET` IPV4 协议
	- `AF_INET6` IPV6 协议
	- `AF_LOCAL` Unix 域
	- `AF_ROUTE` 路由套接字
	- `AF_KEY` 秘钥套接字
- `type`
	- `SOCK_STREAM` 字节流套接字
	- `SOCK_DGRAM` 数据报套接字
	- `SOCK_SEQPACKET` 有序分组套接字
	- `SOCK_RAW` 原始套接字
- 	`protocol`
   - `IPPROTO_TCP` TCP 传输协议
   - `IPPROTO_UDP` UDP 传输协议
   - `IPPROTO_SCTP` SCTP 传输协议

## 2、调用 `connect` 函数出错返回有哪几种情况？

```
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)

```

- 若TCP客户没有收到 `SYN` 分节的响应, 返回 `ETIMEDOUT` 错误。
- 客户发送 `SYN` 分节后，收到 `RST` 响应，表明在指定的端口上没有进程在等待与之连接, 返回 `ECONNREFUSES` 错误。
- 若客户发送的 `SYN` 在中间某个路由器上引起了一个 `destination unreachable` `ICMP`错误. 返回 `EHOSTUNREACH` 或 `ENETUNTREACH` 错误

> 产生RST的三个条件：

> - 目的地为某端口的SYN到达，然而该端口上没有正在监听的服务器；
> - TCP想取消一个已有连接；
> - TCP接收到一个根本不存在的连接上的分节。

## 3、`bind` 函数地址、端口绑定的原则？

```
int bind(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)

```

- 对于 TCP 服务端来说，bind 函数会绑定一个众所周知的端口，还会绑定一个 IP 地址，这样该套接字就会只接受那些目的地为这个 IP 的客户连接。
- 服务器和客户端都可以bind，bind并不是服务器的专利。
客户端进程bind端口：  由进程选择一个端口去连服务器，（如果默认情况下，调用bind函数时，内核指定的端口是同一个，那么调用多个调用了bind（）的client程序，会出现端口被占用的错误）注意这里的端口是客户端的端口。如果不分配就表示交给内核去选择一个可用端口。
客户端进程bind IP地址：相当于为发送出去的IP数据报分配了源IP地址，但交给进程分配IP地址的时候（就是这样写明了bind IP地址的时候）这个IP地址必须是主机的一个接口，不能分配一个不存在的IP。如果不分配就表示由内核根据所用的输出接口来选择源IP地址。一般情况下客户端是不用调用bind函数的，一切都交给内核搞定

## 4、`listen` 函数中 `backlog` 参数的意义？

```
int listen(int sockfd, int backlog)

```

backlog: 内核应该为相应套接字排队的最大连接个数。
内核为每个监听的套接字维护两个队列:

- 未完成连接队列：由某个客户端发出并到达服务器，而服务器正在等待完成相应的TCP三路握手过程，这些套接字处于 `SYN_RCD`状态
- 已完成连接队列：每个已完成三次握手过程的客户对应其中一项。这些套接字处于ESTABLISHED状态。

当来自客户的 `SYN` 到达时，`TCP` 将会在未完成队列中创建一个新项，然后响应三次握手的第二个分节。这一项一直保留在未完成连接队列中，直到三次握手的第三个分节到达或超时。

如果三次握手正常完成，该项就从未完成队列移到已完成连接队列的队尾。当进程调用 `accecp` 时，已完成连接队列中的对头项将会返回给进程，或者如果该队列为空，那么进程将会被投入睡眠。

## accept

```
int accept(int sockfd,struct sockaddr *cliaddr, socklen_t *addrlen)

```
该函数传入一个监听套接字 sockfd

如果成功，该函数会返回一个新的 socket 套接字，可以称之为连接套接字，客户端的地址会放到 cliaddr 中

## 5、`getsockname` 函数的用途有哪些？

- `TCP` 客户端：`connect` 成功后，`getsockname` 用于返回由内核赋予的该连接的本地 IP 地址和本地端口号
- `TCP` 服务端：
    - 以端口号 0 调用 `bind` 函数后，`getsockname` 函数用于返回由内核赋予的本地端口号
    - 以通配 IP 地址调用 `bind` 函数的服务器，与某个客户的连接一旦建立， `getsockname` 就可以用于返回由内核赋予该连接的本地 IP 地址。但是，套接字描述符参数必须是已连接套接字的描述符，而不是监听套接字的描述符。 

## 6、`getpeername` 函数的用途有哪些？

可以在TCP的服务器端accept成功后，通过 getpeername() 函数来获取当前连接的客户端的IP地址和端口号.

getsockname和getpeername调度时机很重要，如果调用时机不对，则无法正确获得地址和端口。

- 毫无疑问，getpeername 只能用于已经建立连接的套接字
- 如果想要获取端口，bind 函数之后就可以调用 getsockname，如果想要获取 IP 地址，那么还是需要连接套接字。

## API

```
int socket(int family, int type, int protocol); 

int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen); 

int bind(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);

int listen (int sockfd, int backlog)

int accept(int sockfd,struct sockaddr *cliaddr, socklen_t *addrlen)

int close(int sockfd)

int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen)
int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen)

```