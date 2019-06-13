# 基本 TCP 套接字编程

## 1、`socket` 函数中 `family`、`type`、`protocol` 等参数的选项有哪些？

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

- 若TCP客户没有收到 `SYN` 分节的响应, 返回 `ETIMEDOUT` 错误。
- 客户发送 `SYN` 分节后，收到 `RST` 响应，表明在指定的端口上没有进程在等待与之连接, 返回 `ECONNREFUSES` 错误。
- 若客户发送的 `SYN` 在中间某个路由器上引起了一个 `destination unreachable` `ICMP`错误. 返回 `EHOSTUNREACH` 或 `ENETUNTREACH` 错误

> 产生RST的三个条件：

> - 目的地为某端口的SYN到达，然而该端口上没有正在监听的服务器；
> - TCP想取消一个已有连接；
> - TCP接收到一个根本不存在的连接上的分节。

## 3、`bind` 函数地址、端口绑定的原则？

![](http://owql68l6p.bkt.clouddn.com/bind%20%E5%87%BD%E6%95%B0.png)

## 4、`listen` 函数中 `backlog` 参数的意义？

backlog: 内核应该为相应套接字排队的最大连接个数。
内核为每个监听的套接字维护两个队列:

- 未完成连接队列：由某个客户端发出并到达服务器，而服务器正在等待完成相应的TCP三路握手过程，这些套接字处于 `SYN_RCD`状态
- 已完成连接队列：每个已完成三次握手过程的客户对应其中一项。这些套接字处于ESTABLISHED状态。

![](http://owql68l6p.bkt.clouddn.com/tcp%20%E4%B8%89%E6%AC%A1%E6%8F%A1%E6%89%8B%E5%92%8C%E7%9B%91%E5%90%AC%E5%A5%97%E6%8E%A5%E5%AD%97%E7%9A%84%E4%B8%A4%E4%B8%AA%E9%98%9F%E5%88%97.png)

当来自客户的 `SYN` 到达时，`TCP` 将会在未完成队列中创建一个新项，然后响应三次握手的第二个分节。这一项一直保留在未完成连接队列中，直到三次握手的第三个分节到达或超时。

如果三次握手正常完成，该项就从未完成队列移到已完成连接队列的队尾。当进程调用 `accecp` 时，已完成连接队列中的对头项将会返回给进程，或者如果该队列为空，那么进程将会被投入睡眠。

## 5、`getsockname` 函数的用途有哪些？

- `TCP` 客户端：`connect` 成功后，`getsockname` 用于返回由内核赋予的该连接的本地 IP 地址和本地端口号
- `TCP` 服务端：
    - 以端口号 0 调用 `bind` 函数后，`getsockname` 函数用于返回由内核赋予的本地端口号
    - 以通配 IP 地址调用 `bind` 函数的服务器，与某个客户的连接一旦建立， `getsockname` 就可以用于返回由内核赋予该连接的本地 IP 地址。但是，套接字描述符参数必须是已连接套接字的描述符，而不是监听套接字的描述符。 

## 6、`getpeername` 函数的用途有哪些？

`TCP` 服务器是由 `accept` 的某个进程通过调用 `exec` 执行程序时，它能够获取客户身份的唯一途径便是调用 `getpeername`.

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