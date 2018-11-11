---
description: docker namespace
---

# namespace

Linux Namespace 是Linux提供的内核级别环境隔离的方法，很早以前unix 有一个叫chroot的系统调用（通过修改根目录把用户jail到特定的目录下） chroot提供了一种简单的隔离模式， chroot 内部文件系统无法访问外部的内容。 Linux Namespace在此基础上，提供了对UTS IPC mount PID network User等的隔离机制



| 分类(namespace) | 系统调用 | 备注 |
| :--- | :--- | :--- |
|mount  | CLONE_NEWNS |  |
|uts| CLONE_NEWUTS |  |
|ipc  | CLONE_NEWIPC |  |
| pid | CLONE_NEWPID |  |
| network| CLONE_NEWNET |  |
| user|  | CLONE_NEWUSER |

主要是三个主要的系统调用

* clone() 实现线程的系统调用，用来创建新的进程，并通过设计上述参数达到隔离
* unshare() 使某个进程脱离某个namespace
* setns() 把某个进程加入到某个namespace

# clone 系统调用


# 未完待续 还需要继续研究操作系统原理

引用链接

[https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html](https://www.ibm.com/developerworks/cn/linux/1506_cgroup/index.html)

