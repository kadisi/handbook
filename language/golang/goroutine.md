---
description: golang goroutine
---

# goroutine

golang 中有4个最重要的结构:

 **M**  内核级线程，一个M就是一个线程，goroutine 就是跑在一个线程之上，M是一个很大的结构，里面维护了一个小对象内存cache，（mcache）, 当前执行的goroutine，随机数发生器等好多信息  **P**  全称是Processor，处理器，它的主要用途是用来执行goroutine 的，它也维护了一个goroutine 队列，里面存储了它需要执行的goroutine队列，  **G**  G 就是goroutine, G 维护了goroutine需要的栈，所要运行的函数，程序计数器，以及它所在的M等信息  **sched** 

