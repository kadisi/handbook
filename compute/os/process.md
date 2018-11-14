---
计算机知识: 操作系统 进程
---

# 进程

## 操作系统 进程

在Linux 中，每个进程都有父进程，而所有的进程都已init进程为根，形成一个树状的结构，我们在这里讲解进程组和会话，以便以更丰富的方式管理进程

## 进程组

每个进程都会属于一个进程组（process group）每个进程组中可以包含多个进程。进程组会有一个进程组领导进程（process group leader ）领导进程的PID 成为进程组的ID \(PGID\)以识别进程组

PID 为进程自身的ID ， PGID为进程所在的进程组的ID PPID为进程的父进程ID

如下图

![](../../.gitbook/assets/image%20%2818%29.png)

  
图中箭头表示父进程通过[fork和exec机制](http://www.cnblogs.com/vamei/archive/2012/09/20/2694466.html)产生子进程。ps和cat都是bash的子进程。进程组的领导进程的PID成为进程组ID。领导进程可以先终结。此时进程组依然存在，并持有相同的PGID，直到进程组中最后一个进程终结。

我们将一些进程归为进程组的一个重要原因是我们可以将[信号](http://www.cnblogs.com/vamei/archive/2012/10/04/2711818.html)发送给一个进程组。进程组中的所有进程都会收到该信号。我们会在下一部分深入讨论这一点。

## 引用

{% embed url="http://www.cnblogs.com/vamei/archive/2012/10/07/2713023.html" %}

