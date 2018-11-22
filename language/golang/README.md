---
description: golang 语言
---

# 概述 

golang  高效的核心在于减少了系统调用

例如 golang 的MPG 模型 就是减少了线程的上下文切换

golang 的内存分配 也是首先使用mmap 申请一大块内存，自足管理，使用tcmalloc 模式，减少系统调用

