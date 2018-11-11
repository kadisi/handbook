---
description: docker namespace 
---

# namespaces
Linux Namespace 是Linux提供的内核级别环境隔离的方法，很早以前unix 有一个叫chroot的系统调用（通过修改根目录把用户jail到特定的目录下） chroot提供了一种简单的隔离模式， chroot 内部文件系统无法访问外部的内容。 
Linux Namespace在此基础上，提供了对UTS IPC mount PID network User等的隔离机制



