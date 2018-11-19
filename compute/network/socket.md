---
网络: socket 套接字
---

# socket 套接字

## 概述

主要讲解一下一个完整的TCP客户/服务器端 程序所需要的基本套接字函数。

## socket 函数

为了执行网络IO 一个进程必须做的第一件事就是调用socket函数。指定期望的通信协议类型（TCP UDP Unix域字节流协议等等）

```text
int socket(int family, int type, int protocal)

返回 若成功则为非负描述符， 若出错， 则为-1
```

其中family 参数指定协议族， 该参数也往往被称为协议域。 type参数指明套接字类型， protocal 应该设置为某个协议类型常值。并非所有family 与type的组合都是有效的。

![](../../.gitbook/assets/image%20%286%29.png)

socket 函数在成功返回一个小的非负整数值，他与文件描述符类似，我们把它称为套接字描述符。 简称sockfd。 为了得到这个套接字描述符， 我们只是指定了协议族（IPV4 IPV6 UNIX）和套接字类型 （字节流， 数据报文， 或者原始套接字）我们并没有指定本地协议地址或者远程协议地址

## connect 函数
TCP 客户端用connect 函数来建立与TCP服务器的链接
```
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
返回: 若成功则为0， 若出错则为-1
```

sockfd 是有socket 函数返回的套接字描述符， 第二个 第三个参数分别是一个指向套接字地址结构的指针和改结构大小。 套接字地址结构必须包含服务器IP地址和端口号

客户端在调用函数connect前不必非得调用bind函数， 因为如果需要的话， 内核会确定源IP地址， 并选择一个临时端口作为源端口。

如果是TCP套接字， 调用connect 函数将激发TCP 的三次握手过程， 而且仅仅在连接建立成功或者出错时候才返回， 其中出错返回可能有以下几种情况：

* 若TCP 客户没有收到SYN分节的相应， 则返回ETIMEOUT错误， 举例来说，调用connect函数时候， 4.4BSD内核发送一个SYN，若无相应则等待6s再发送一个， 若扔无响应则等待24s后在发送一个， 若总共等了75s后仍未收到相应则返回本错误

* 若对客户的相应是RST（表示复位或者重新连接），则表明该服务器主机在我们指定的端口上没有进程在等待与之链接（也许服务器进程压根没有 在运行）这是一种硬错误，客户一收到RST 就马上返回ECONNREFUSED（连接拒绝）错误.

* 若客户端发出的SYN在中间的某个路由器上引发了一个“destination unreachable” (目的地不可达) 的ICMP错误， 则认为是一种软错误。 客户主机内核保存该消息， 并按照第一种情况锁描述的时间间隔，继续发送SYN。 若在规定的时间后仍为收到相应， 则把保存的消息（即 ICMP错误）作为EHOSTUNREACH 或者ENETUNREACH（网络或者主机不可达）返回给进程

RST 是TCP在发生错误时候发送的一种TCP分节， 产生RST的三个条件是： 目的地位某个端口的SYN到达， 然而该端口上没有正在监听的服务器； TCP 想取消一个已有的连接； TCP 接受到一个根本不存在的连接上的分节。


## bind 函数

bind 函数把一个本地协议地址赋予一个套接字。 对于网际网协议， 协议地址是32位的IPV4 地址或者128位的IPV6 地址与16位的TCP或者UDP端口号的组合。

```
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen)
返回： 若成功则为0  若出错则为-1
```

第二个参数 是一个指向特定于协议的地址结构的指针， 第三个参数是该地址结构的长度。 对于TCP， 调用bind 函数可以指定一个端口号， 或者指定一个IP地址，也可以两者都指定， 还可以都不指定。

* sdf

* sdf











































































