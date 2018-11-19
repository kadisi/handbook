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

