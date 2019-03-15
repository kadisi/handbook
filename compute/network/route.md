---
网络: 网络 路由
---

# 路由

## 路由

### 简单路由表

```text
[root@172.20.7.100 ~]# netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         172.20.7.254    0.0.0.0         UG        0 0          0 fake
172.20.7.0      0.0.0.0         255.255.255.0   U         0 0          0 fake
```

对于一个给定的路由器， 可以打印出5种不同的标志

* U 代表改路由可以使用
* G 该路由是一个网关（路由器）如果没有设置标志， 说明目的地址是直接相连的
* H 该路由是一个主机， 也就是说 目的地址是一个完整的主机地址， 如果没有设置该标志， 说明该路由是 到一个网络， 而目的地址是一个网络地址： 一个网络号 或者网络号和子网号的组合
* D 该路由是由重定向报文创建的
* M 该路由已被重定向报文修改

标志G 非常重要， 因为由他区分了间接路由和直接路由（对于直接路由来说是不设置标志G的） .

其区别在于发往直接路由的分组中， 不但具有指明目的端的IP地址， 还具有其链路层地址。

当分组被发往一个间接路由时，IP地址指明的是最终的目的地， 但是链路层地址指明的是网关。

### IP 转发

默认情况， 主机一般不转发IP数据包， 除非对他们进行特殊配置作为路由器使用， 大多数伯克利派生出来的系统都有一个内核变量 ipforwarding， 只有在该变量值不为0 的情况下才转发数据报。

serverfault.com 上对IP转发有个很好的回答： [https://serverfault.com/questions/248841/ip-forwarding-when-and-why-is-this-required/248845](https://serverfault.com/questions/248841/ip-forwarding-when-and-why-is-this-required/248845)

```text
IP forwarding should be enabled when you want the system to act as a router, that is transfer IP packets from one network to another.

In the simplest case, consider a server with two physical ethernet ports which is meant to connect to two different networks (say your internal network and the outside world as provided by a DSL modem). If you just connect and configure those two interfaces, the system can communicate on either network. However, packets from one network cannot travel to the other network, because forwarding is not enabled.

Consider the specific example of 'route add'. If you have two network interfaces, you will add a minimum of two routes, one for each interface. When the kernel considers where to send a network packet, it will pick the most specific applicable route and then send it along to that interface.

However, if forwarding is turned off, the kernel will first check to see which interface the packet came from. If it didn't come from the same interface, the kernel will discard it.

EDIT: First note that you can use a router without having two physical network interfaces. For example if you are using VLANs, your server can transfer IP packets between vlans but only have one physical network interface. This is called a one-armed router. However for the simplest case yes you can say that if you only have one physical network interface then you don't need to enable IP forwarding.

IP forwarding involves transferring packets between network interfaces (real or virtual) so I think that if you had two interfaces on the same network, you would have to enable ip forwarding to allow packets to move between the interfaces. However since the interfaces are already on the same network, it doesn't seem to make a lot of sense to transfer packets between them.
```

大意： ip forwarding 能够让你的系统看起来像个路由器， 让数据报从一个网络转发到另一个网络

简单的例子， 假设服务器有两个物理网卡， 两个物理网卡连接不同的网络， 如果你仅仅连接和配置两个物理网卡， 系统可以跟每个网络通信， 然而数据包从一个网络不能够传到另一个网络， 因为forwarding not enabled

考虑到`route add` 的例子，如果你有两个网络接口， 你需要添加两个路由， 每个路由对应一个网卡， 当kernerl 考虑到要把网络packet 发到哪里时候， 它将要选择这个最合适的路由， 然后通过这个网卡把数据包扔出去。

然而 如果forwarding 是管的， kernel 将要首先检查这个packet 来自于哪里， 如果它没有来自于同一个网卡， kernel 将要丢弃它

注意： 如果你单纯只有一个物理网卡， 你压根不需要开启ip 转发

IP forward 包括了 数据包在网络接口之间的传输， 因此我认为如果你有两个物理接口 但是具有相同的网络， 你不得不开启ip forward 来允许数据包在这接口中的传递， 然而 因为接口是在相同的网络下， 因此他们之间的数据包传送没啥意义

## ip route 命令

在linux 系统中， 内核会为路由策略数据库配置三条缺省的规则：

* 0 匹配任何条件， 查询路由表 local 路由表local 是一个特殊的路由表， 包含本地和广播地址的高级优先级路由
* 32766 匹配任何条件 查询路由表 main， 路由表main 是一个通常的表，正常我们通过ip route show 命令， 查看的都是看的这张表， 包含所有无策略路由， 系统关系源可以删除或者使用另外规则覆盖
* 32767 匹配任何条件， 查询路由表 default， 路由表default 是一个空表， 为后续的操作保留的。

举例， 查询local 路由表规则

```text
[root@172.20.7.99 ~]# ip route show table local
broadcast 127.0.0.0 dev lo proto kernel scope link src 127.0.0.1
local 127.0.0.0/8 dev lo proto kernel scope host src 127.0.0.1
local 127.0.0.1 dev lo proto kernel scope host src 127.0.0.1
broadcast 127.255.255.255 dev lo proto kernel scope link src 127.0.0.1
broadcast 172.20.7.0 dev fake proto kernel scope link src 172.20.7.99
local 172.20.7.99 dev fake proto kernel scope host src 172.20.7.99
broadcast 172.20.7.255 dev fake proto kernel scope link src 172.20.7.99
```

默认 查看main 路由表

```text
[root@172.20.7.99 ~]# ip route
default via 172.20.7.254 dev fake
172.20.7.0/24 dev fake proto kernel scope link src 172.20.7.99
```

或者

```text
[root@172.20.7.99 ~]# ip route show table main
default via 172.20.7.254 dev fake
172.20.7.0/24 dev fake proto kernel scope link src 172.20.7.99
```

### ip route 的scope link 含义

scope 不写时候， 默认是 global

```text
scope value 
      the scope of the destinations covered by the route prefix.  SCOPE_VAL may be a number or a
      string from the file /etc/iproute2/rt_scopes.  
      If this parameter is omitted, ip assumes scope global for all gatewayed unicast routes
      scope link for direct unicast and broadcast routes
      scope host for local routes.


onlink pretend that the nexthop is directly attached to this link, even if it does not match any interface prefix.
```

简单点 scope link 可以代表是单播或者广播路由， 就是能够arp 到目标地址的， 意思就是说 目标地址 属于跟本地直连的二层链路上， 而不是垮三层。

### ip route 的 proto kernel 含义

proto 不写的时候， 默认是boot

```text
 protocol RTPROTO
                     the routing protocol identifier of this route.  RTPROTO may be a number or a string from the
                     file /etc/iproute2/rt_protos.  If the routing protocol ID is not given, ip assumes protocol
                     boot (i.e. it assumes the route was added by someone who doesn't understand what they are
                     doing). Several protocol values have a fixed interpretation.  Namely:

                             redirect - the route was installed due to an ICMP redirect.

                             kernel - the route was installed by the kernel during autoconfiguration.

                             boot - the route was installed during the bootup sequence.  If a routing daemon starts,
                             it will purge all of them.

                             static - the route was installed by the administrator to override dynamic routing.
                             Routing daemon will respect them and, probably, even advertise them to its peers.

                             ra - the route was installed by Router Discovery protocol.

                     The rest of the values are not reserved and the administrator is free to assign (or not to
                     assign) protocol tags.
```

## linux 系统配置rp\_filter

rp\_filter reserve path filter, 参数用于控制系统是否开启对数据包源地址校验。

```text
0：不开启源地址校验。
1：开启严格的反向路径校验。对每个进来的数据包，校验其反向路径是否是最佳路径。如果反向路径不是最佳路径，则直接丢弃该数据包。

2：开启松散的反向路径校验。对每个进来的数据包，校验其源地址是否可达，即反向路径是否能通（通过任意网口），如果反向路径不同，则直接丢弃该数据包。
```

举例

```text
[root@172.20.7.99 ~]# sysctl net.ipv4 |grep -v arp_filter |grep rp_filter
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.eth0.rp_filter = 1
net.ipv4.conf.fake.rp_filter = 1
net.ipv4.conf.lo.rp_filter = 0
```

## linux 系统配置 arp\_filter

arp\_filter 的表兄弟就是 rp\_filter

```text
arp_filter - BOOLEAN
    1 - Allows you to have multiple network interfaces on the same
    subnet, and have the ARPs for each interface be answered
    based on whether or not the kernel would route a packet from
    the ARP'd IP out that interface (therefore you must use source
    based routing for this to work). In other words it allows control
    of which cards (usually 1) will respond to an arp request.

    0 - (default) The kernel can respond to arp requests with addresses
    from other interfaces. This may seem wrong but it usually makes
    sense, because it increases the chance of successful communication.
    IP addresses are owned by the complete host on Linux, not by
    particular interfaces. Only for more complex setups like load-
    balancing, does this behaviour cause problems.

    arp_filter for the interface will be enabled if at least one of
    conf/{all,interface}/arp_filter is set to TRUE,
    it will be disabled otherwise
```

当一台机器有多个位于同一个网段中的网卡， 每个网卡都有各自IP， 是否允许把一个网卡的mac 地址作为对另一个网卡的arp 请求回应

默认为0 表示允许， 这样即使一块网卡故障了， 报文可以被另一个网卡接受

