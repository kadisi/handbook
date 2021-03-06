---
description: 本文介绍linux 下的共享内存机制
---

# 共享内存

共享内存定义： 共享内存是最快的可用IPC（进程间通信）形式， 它允许多个不相关的进程去访问同一部分逻辑内存，。 共享内存是由IPC 为一个进程创建的一个特殊的地址范围， 它将出现在进程的地址空间中。 其他进程可以把同一段共享内存段 “ 链接到” 他们自己的地址空间中去。 所有进程都可以访问共享内存中的地址。 如果一个进程向这段共享内存写了数据， 所做的其他改动会立刻被有访问同一段共享内存的其他进程看到， 因此共享内存对于数据的传输是非常高效的。

共享内存存于进程的**数据段** 通过shmget 默认的SHMMAX的默认值是32M，

共享内存可以用shmget 和mmap 调用完成

通过shmget 可以创建或者获得共享内存的标识符，取得共享内存的标识符后， 要通过shmat 将内存区映射到本进程的虚拟机制空间。

## shmget 和mmap的区别

本质上mmap可以看到文件的实体， shmget 对应文件在交换分区的shm 文件系统内， 貌似无法read 和write

共享内存允许两个或者多个进程共享一给定的存储区， 因为数据不需要来回复制， 所以是最快的一种进程间通讯的机制， 共享内存可以通过mmap映射普通文件机制实现， 也可以通过系统V\(shm\)共享内存机制实现，

mmap保存到实际硬盘

shm保存到物理存储器， 优点 进程间访问速度会快。

## 共享内存互斥访问

两个进程使用共享内存，同时访问时候，必然会遇到同步和互斥的问题，

一般情况下， 有下面几种方法

1 互斥

2 信号量

3 文件锁 对共享内存的一部分区域进行加锁，

一般互斥锁或者信号量需要在共享内存这块空间里创建，这样两个进程能拿到同一个锁。初始化该锁的时候， 需要设置为进程间共享。

但是使用锁，有可能导致加锁后，进程挂掉的情况，出现死锁

