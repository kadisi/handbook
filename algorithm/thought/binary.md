---
计算机知识: 算法思想 二分法
---

# 二分

## 二分法

二分法最具有典型的代表是二分查找

## 二分查找

二分查找核心思想有点像分治思想，每次都通过区间中的中间元素进行匹配，将待查找的区间缩小为一半，直到查到到要查找的元素。或者区间缩小为0 也没有查到。 但是二分查找的代码实现容易比较写错，需要掌握它的三个容易出错的地方：

* 循环退出条件
* mid 的取值
* low 和high 的更新

关于mid 取值， 大部分人都是这么写 `mid := (high+low)/2` , 但是这么写容易造成溢出， 因此标准的写法应该是 `mid := low + (high-low)/2`, 但是相比于除法操作， 计算机对于位运算更快，因此 最快 最标准的写法应该是 `mid := low + (high-low) >> 1`

二分查找虽然比较优秀，但是二分查找的应用场景也比较局限

* 底层必须是数组
* 数组必须是有序的

对于数据比较小的规模，我们可以直接使用顺序遍历即可，二分查找的优势并不明显。

二分查找适合于

* 处理静态数据
* 没有频繁的数据插入，删除操作

## 二分查找的应用场景

* 求一个数的算术平方根（可以用二分思想，但是算术平方根还有个更快更优的解法 叫 牛顿迭代法）
