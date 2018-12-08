---
网络: bridge vs macvlan 
---
# bridge
bridge 是二层网络设备，用来连接 two Layer 2 (i.e. Ethernet) segments together. 
Frames between the two segments are forwarded based on the Layer 2 addresses (i.e. MAC addresses). 

bridge 实际上可以理解为二层交换机

一个bridge 根据mac 地址表来做转发， bridge 通过帧头中的地址来学习mac 地址。

bridge 可以是一个物理设备 也可以由纯软件实现。 从1999年后， linux kernel 能够在软件层面实现bridge 功能。

通过创建一个bridge， 你可以讲多个物理或者虚拟接口链接到一个二层segment。在一个linux主机中 讲两个物理设备连接起来的bridge 实际上讲这个host 转变成了物理二层交换机

![](../../.gitbook/assets/bridge1.png)

交换机此时已经变成了一个专门的物理设备，软bridge 已经失去了他的地位。 然而，随着虚拟化的发展， 运行在物理机上的虚拟机需要2层连接到物理网络和其他的虚拟机上。 linux bridge 提供了一个非常好的技术，使得bridge 重新开始复兴。

一个bridge 可以将虚拟网络接口彼此连接， 也可以将虚拟网络接口和物理接口彼此连接， 把他们都连接到一个二层设备中。


![](../../.gitbook/assets/bridge2.png)

你可以使用brctl 命令来配置linux host bridge， brctl 在bridge-utils包中

```
# brctl show
bridge name  bridge id          STP enabled  interfaces
br0          8000.080006ad34d1  no           eth0
                                             veth0
br1          8000.080021d2a187  no           veth1

```
bridge 有可能造成二层环路 ， 因此你可以设置STP选项 如果需要的话

## STP

生成树协议（英语：Spanning Tree Protocol，STP），是一种工作在OSI网络模型中的第二层(数据链路层)的通信协议，基本应用是防止交换机冗余链路产生的环路.用于确保以太网中无环路的逻辑拓扑结构.从而避免了广播风暴,大量占用交换机的资源.

生成树协议工作原理:任意一交换机中如果到达根网桥有两条或者两条以上的链路.生成树协议都根据算法把其中一条切断,仅保留一条.从而保证任意两个交换机之间只有一条单一的活动链路.因为这种生成的这种拓扑结构.很像是以根交换机为树干的树形结构.故为生成树协议


# macvlan 

macvlan 允许你配置多个二层地址在一个物理设备接口上。 macvlan 允许你给父接口物理设备(有时候称作为upper device)配置子接口（有时候也会称作slave device）,每个子接口都有自己独立mac 地址（自动生成）和独立的IP地址.

应用程序， vm 和容器 可以绑定到一个特定的子接口去直连物理网络，通过使用它们自己的mac 和ip地址。

子接口macvlan 不能改直接与它们的父接口通信，例如： vm 不能直接访问他的宿主机。 如果你需要vm 和宿主机互相通信，你应该增加另一个macvlan 子接口并且把主机的ip地址配置个这个子接口，物理接口实际上没有IP。 

macvlan 子接口用使用 mac0@etch0 的形式， 是为了清楚的定义子接口和父接口的关系， 子接口状态与父接口的状态做绑定， 如果父接口eth0 down 了， 子接口mac0@eth0 也down了

![](../../.gitbook/assets/macvlan.png)

# macvlan mode










































# 引用

{% embed url="https://hicu.be/bridge-vs-macvlan" %}



