---
网络: socat
---

# socat

## socat

Socat 基本语法

```text
socat [options] <address> <address>
```

其中这 2 个 address 就是关键了，address 类似于一个文件描述符，Socat 所做的工作就是在 2 个 address 指定的描述符间建立一个 pipe 用于发送和接收数据。几个常用的 address 描述方式如下：

-,STDIN,STDOUT 表示标准输入输出，可以就用一个横杠代替。 /var/log/syslog 打开一个文件作为数据流，可以是任意路径。 TCP:: 建立一个 TCP 连接作为数据流，TCP 也可以替换为 UDP 。 TCP-LISTEN: 建立 一个 TCP 监听端口，TCP 也可以替换为 UDP。 EXEC: 执行一个程序作为数据流。 以上规则中前面的 TCP 等都可以小写，在这些描述后可以附加一些选项，用逗号隔开。如 fork，reuseaddr，stdin，stdout，ctty 等。

## 使用实例

### 文件操作

读写文件

```text
# 从绝对路径读取
$ socat - /var/www/html/flag.php 

# 从相对路径读取
$ socat - ./flag.php


# 写文件
$ echo "This is Test" | socat - /tmp/hello.html
```

### 网络管理

连接远程端口

```text
socat - TCP:192.168.1.252:3306
```

监听端口

```text
socat TCP-LISTEN:7000 -
```

### 端口转发

在实际生产中我们经常会遇到到一个场景就是，用一台机器作为转发服务器，连接 AB 两个网段，将转发服务器的某个端口上的流量转发到 B 网段的某台机器的某个端口，这样 A 网段的服务器就可以通过访问转发服务器上的端口访问到 B 网段的服务器端口。

这样的场景一般在和客户建立专线的连接时候经常用到，一般也可以采用 iptables 做转发，但是比较复杂。Socat 可以很轻松的完成这个功能，但是 Socat 不支持端口段转发，只适用于单端口或者少量端口。

转发TCP

监听 192.168.1.252 网卡的 15672 端口，并将请求转发至 172.17.0.15 的 15672 端口。

```text
socat  -d -d -lf /var/log/socat.log TCP4-LISTEN:15672,bind=192.168.1.252,reuseaddr,fork TCP4:172.17.0.15:15672
```

参数说明

```text
-d -d  前面两个连续的 -d -d 代表调试信息的输出级别。
-lf /var/log/socat.log 指定输出信息的文件保存位置。 
TCP4-LISTEN:15672 在本地建立一个 TCP IPv4 协议的监听端口，也就是转发端口。这里是 15672，请根据实际情况改成你自己需要转发的端口。
bind 指定监听绑定的 IP 地址，不绑定的话将监听服务器上可用的全部 IP。
reuseaddr 绑定一个本地端口。
fork TCP4:172.17.0.15:15672 指的是要转发到的服务器 IP 和端口，这里是 172.17.0.15 的 15672 端口。
```

转发udp

转发 UDP 和 TCP 类似，只要把 TCP4 改成 UDP4 就行了。

### 文件传递

将客户端文件a.txt 转发到96 上， 保存为lisi.txt

服务器端96 首先执行

```text
socat -u TCP4-LISTEN:9999 open:./lisi.txt,create
```

客户端执行：

```text
socat -u open:./a.txt tcp:172.20.7.96:9999,reuseaddr
```

```text
-u 表示数据传输模式为单向，从左面参数到右面参数。
-U 表示数据传输模式为单向，从右面参数到左面参数。
```

## 引用

[https://www.hi-linux.com/posts/61543.html](https://www.hi-linux.com/posts/61543.html)

