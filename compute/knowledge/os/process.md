---
计算机知识: 操作系统 进程
---

# 操作系统 进程
在Linux 中，每个进程都有父进程，而所有的进程都已init进程为根，形成一个树状的结构，我们在这里讲解进程组和会话，以便以更丰富的方式管理进程


# 进程组
每个进程都会属于一个进程组（process group）每个进程组中可以包含多个进程。进程组会有一个进程组领导进程（process group leader ）领导进程的PID 成为进程组的ID (PGID)以识别进程组

PID 为进程自身的ID ， PGID为进程所在的进程组的ID PPID为进程的父进程ID

# 引用
[http://www.cnblogs.com/vamei/archive/2012/10/07/2713023.html](http://www.cnblogs.com/vamei/archive/2012/10/07/2713023.html)



