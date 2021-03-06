# 快排

## 数组的快速排序

### case 1 从数组两侧

这种是通过一个基准哨兵，一般是从array\[start\] 开始， 之后设置两个下标， i,j 分别等于start 和end

对于基准是左侧的，p = array\[start\] 要首先

1 从右边开始查看 j--，知道找到比基准p 小的数

2 之后从左侧开始查看， i++, 直到找到比p 大的数

3 这样，如果i和j 不相等 则交换i 和j 下标的值

4 重新从1 开始， 直到i==j

5 最后i==j 的情况肯定是array\[i\] &lt;= p, 最后记得讲最开始的哨兵array\[start\] 与array\[i\] 互换

6 分治法处理 \[start ， i-1\]. \[i+1, start\]

```text
func QuickSort(array []int, start, end int) {
    if start >= end  {
        return
    }
    p := array[start]
    i,j := start, end
    for ;i<j; {
        for ;i<j ; j-- {
            if array[j] < p {
                break
            }
        }
        for ;i<j; i++{
            if array[i] > p {
                break
            }
        }
        if i != j {
            array[i], array[j] = array[j], array[i]
        }
    }
    array[start], array[i] = array[i], array[start]
    QuickSort(array, start, i-1)
    QuickSort(array, i+1, end)
}
```

### case 2 从数组的一侧

设置 i，j 都等于start，不断移动j，使得最终的i 左边的都小于基准值array【start】, i和j 中间的都大于基准值 array【start】

流程

1 设定基准值 p = array\[start\] , 设定下标i,j 都等于start

2 如果array【j】 &lt; p 的情况下，需要array【i】和array【j】的数据互换，但是要首先i++，这样才能保证将array（i,j\)里的数据与array【j】交换，这样能保证array\(i,j】的数据&gt;p

3 j ++

4 重复2,一直到j &gt; end

5 最后还是哨兵 与 i 下标置换，array\[i\],array\[start\] = arry\[start\], array\[i\]

6 分治法处理\[start ， i-1\]. \[i+1, start\]

```text
func QuickSort2(array []int, start, end int) {
   if start >= end {
      return
   }
   p := array[start]
   i, j := start, start
   for ;j<=end; j++{
      if array[j] < p {
         i++
         array[i], array[j] = array[j], array[i]
      }
   }

   array[i], array[start] = array[start], array[i]

   QuickSort2(array, start, i-1)
   QuickSort2(array, i +1, end)
}
```

 

### case2 随机数优化快排

核心思想是在start，end 区间随机找一个下标，然后让这个下标的值和start 下标的值互换， 之后才进行原来的快排逻辑

```text
func QuickSort1(array []int, start,end int) {
   if start >= end {
      return
   }
   ss := getRandom(start, end)
   array[ss], array[start] = array[start], array[ss]
   p,i,j := array[start], start, end
   for ;i<j; {
      for ;i<j; j-- {
         if array[j] < p {
            break
         }
      }
      for ;i<j; i++ {
         if array[i] > p {
            break
         }
      }
      array[i], array[j] = array[j], array[i]
   }

   array[start], array[i] = array[i], array[start]
   QuickSort1(array, start, i-1)
   QuickSort1(array, i+1, end)
}
```

 

## 2 单链表的快速排序

单链表的快速排序，可以参照case2，原理是一样的

## 快排的优化

1 随机数

2 小数量的用插入排序

3 减少递归层次

参照

{% embed url="https://www.cnblogs.com/vipchenwei/p/7460293.html" caption="" %}

## 快排案例

1 找到第K大的数据， 使用快排的思想，能达到O\(n\)的时间复杂度

2 快排是一种不稳定排序

3 归并排序是一种稳定排序， 主要看merge 的书写

