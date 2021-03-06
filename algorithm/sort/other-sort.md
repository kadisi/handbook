# 其他

## 冒泡排序（稳定排序）

最简单， 属于稳定排序，意思是之前两个数相等的话，排序完后， 原来两个数的前后位置 不变

对相邻的元素进行两两比较，顺序相反则进行交换，这样，每一趟会将最小或最大的元素“浮”到顶端，最终达到完全有序。就好像一串气泡一样，最终从小到大或从大到小依次排下来。





```go
func MapPaoSort(a []int) {
   for i :=0; i<=len(a)-1; i ++ {
      for j:=0; j <= len(a)-2 - i; j ++ {
         if a[j] > a[j+1] {
            a[j],a[j+1]=a[j+1],a[j]
         }
      }
   }
}
```

 

## 插入排序（稳定排序）

基本思想 ：插入排序（Insertion Sort）算法通过对未排序的数据执行逐个插入至合适的位置而完成排序工作。



1. 首先对数组的前两个数据进行从小到大排序
2. 然后将第三个数据与前面排好的数据进行比较，把第三个数插入合适的位置
3. 然后将第四个数据插入到前三个数据中
4. 重复此步骤，直到最后一个数插入合适的位置为止，到此排序完成



```text
func InsertSort(a []int) {
   for i:=0; i<=len(a)-2; i ++ {
      for j:=i+1; j>0; j-- {
         if a[j] < a [j-1] {
            a[j],a[j-1]= a[j-1],a[j]
         }
      }
   }
}
```

 

## 选择排序（不稳定排序）

选择排序（Selection sort）是一种简单直观的[排序算法](https://link.jianshu.com/?t=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E6%258E%2592%25E5%25BA%258F%25E7%25AE%2597%25E6%25B3%2595)。它的工作原理是每一次从待排序的[数据元素](https://link.jianshu.com/?t=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%25E6%2595%25B0%25E6%258D%25AE%25E5%2585%2583%25E7%25B4%25A0)中选出最小（或最大）的一个元素，存放在序列的起始位置，直到全部待排序的数据元素排完

```text
/*
直接选择排序是不稳定的.

算法的稳定性定义为,对于待排序列中相同项的原来次序不能被算法改变则称该算法稳定.

比如待排序列为:(2) 3 6 [2] 4 5 ,,,序列中的(2)排在[2]前面,不能因为算法把[2]排到(2)前面.

直接选择排序算法,不稳定性,举个简单的例子,就知道它是否稳定..例如:(7) 2 5 9 3 4 [7] 1...当我们利用直接选择排序算法进行排序时候,(7)和1调换,(7)就跑到了[7]的后面了,原来的次序改变了,这样就不稳定了.
---------------------
作者：houyanjun
来源：CSDN
原文：https://blog.csdn.net/houyanjun/article/details/2446074
版权声明：本文为博主原创文章，转载请附上博文链接！
*/
func SelectSort(a []int) {
	for i:=0; i<=len(a)-1; i ++ {
		minIndex := i
		for j := i ; j <= len(a)-1; j ++ {
			if a[j] < a[minIndex] {
				minIndex = j
			}
		}
		a[minIndex],a[i] = a[i],a[minIndex]
	}
}
```



## 算法选择

同样的三种排序， 最终一般选的是插入排序， 因为插入排序比冒泡 比较的少， 交换次数小，

对于选择排序，因为是不稳定的排序， 一般不会使用它

所以golang 源码库中，sort。sort 方法使用的快排的思想，但是对于数据量比较少时候，使用的是插入排序、



```text
func quickSort(data Interface, a, b, maxDepth int) {
   for b-a > 12 { // Use ShellSort for slices <= 12 elements
      if maxDepth == 0 {
         heapSort(data, a, b)
         return
      }
      maxDepth--
      mlo, mhi := doPivot(data, a, b)
      // Avoiding recursion on the larger subproblem guarantees
      // a stack depth of at most lg(b-a).
      if mlo-a < b-mhi {
         quickSort(data, a, mlo, maxDepth)
         a = mhi // i.e., quickSort(data, mhi, b)
      } else {
         quickSort(data, mhi, b, maxDepth)
         b = mlo // i.e., quickSort(data, a, mlo)
      }
   }
   if b-a > 1 {
      // Do ShellSort pass with gap 6
      // It could be written in this simplified form cause b-a <= 12
      for i := a + 6; i < b; i++ {
         if data.Less(i, i-6) {
            data.Swap(i, i-6)
         }
      }
      insertionSort(data, a, b)
   }
}
```

 

