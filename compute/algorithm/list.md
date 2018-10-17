---
description: 链表
---

# list

链表分为单链表，双向链表，单向循环链表，双向循环链表

链表相关的题目：

1 LRU 缓存

2 单向链表中找到链表的中间的元素（使用快慢指针）

3 链表的翻转， 注意 链表的翻转最好用哨兵链表，就是带有头的链表，这样声明三个变量， back，now，forward

back 刚开始初始化为nil，now 为list 的第一个实际节点，即l.root.next, forward 为now 的next 节点，这里需要判断如果now.next 为nil 或者now 为nil 直接返回，因为代表了只有一个节点和没有节点，剩下的就是back，now，forward 的逐渐移动

