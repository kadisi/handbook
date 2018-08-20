---
description: controller manager 中worke queue 的工作逻辑
---

# worke queue

## 简介

workqueue 是k8s 缓存涉及的一部分，主要处理类似controller manager 的场景，例如： depoy controller 会同时并发处理N个deploy更新事件，假如说同时并发处理20个deploy 的更新时间，这时候很可能会存在多个协程同时处理一个deploy 对象的情况，那么如何规避这种情况呢？k8s 设计了workqueue 的queue对象

## 说明

k8s 通过infromerfactory 机制 监控apiserver 资源对象的更新，通过addEventer 机制，将获得更新的资源对象分发到不同的缓存对象中，这里主要写当从informerFactory 获得更新后，要处理外部缓存的情况，即ADD，UPDATE， DELETE workqueue的对象

workqueue 里有一个深层的结构体： vendor/[k8s.io/client-go/util/workqueue/queue.go](http://k8s.io/client-go/util/workqueue/queue.go)， 它主要提供了ADD，GET ，DONE 的方法

## type 结构体 

主要有几个核心字段： 

```text
queue []t 数组，专门顺序存加入的数据，
dirty map对象，专门存放那些需要被处理的数据元素，相当于一个索引
processing map 对象，专门存放那些正在处理的数据元素，相当于一个索引
```



### Type ADD 逻辑 

当增加一个数据时候，（整个过程加锁）

```text
如果 发现dirty中已经有了，直接返回
如果dirty 中没有，在dirty中添加
如果发现 processing 已经有了，代表了这个元素正在处理中，直接返回
如果前面这几个情况都不满足，最后才将数据append 到queue中，queue代表了 这个元素还没有被处理，并且已经要准备处理

```

### Type Get 逻辑

 整个过程加锁， GET 代表了取出元素，并去处理

```text
如果queue 长度为空，并且这个type 没有shutdown，wait 等待
如果长度为空，并且type已经shutdown，直接返回空
如果前面情况都不满足，获得queue的第一个元素，取出来，请求queue 重新赋值为queue[1:], 这个最后会返回。
在process 中加入这个取出来的元素
在dirty 中删除这个元素
```

### Type Done 逻辑

整个过程加锁，DONE代表了，GET出来的数据，处理完成了

```text
首先从process 中删掉这个元素/*iterm A  ->  ADDiterm A  ->  GETiterm A  ->  ADD在ADD -> GET -> ADD 后，queue 没有A，但是dirty 里有A，同时processing 里也有A根据ADD的逻辑，这样即使你在 iterm A ADD操作后，因为dirty里有，所以直接返回iterm A -> Done*/之后 判断如果dirty中，还有这个元素，则需要重新把这个元素，append 到queue中。代表了，我在处理过程中，这个元素又添加进来了，需要重新处理。
```



