# 归并

归并排序的主旨思想是， 将两个有序序列合并成一个， 首先要将序列拆分， 不断拆分





```go
func MergeSort(array []int, s,  e int) {
   if s >= e {
      return
   }


   m := (e -s)/2 + s

   MergeSort(array, s, m)
   MergeSort(array, m+1, e)
   Merge(array, s, m, e)

   return
}

func Merge(a []int, s, m, e int)  {

   l := a[s:m+1]
   r := a[m+1:e+1]

   t := make([]int, 0, e-s+1)
   i,j := 0, 0
   for ;i < len(l) && j < len(r) ;  {
      var c int
      if l[i] < r[j] {
         c = l[i]
         i ++
      }else {
         c = r[j]
         j ++
      }
      t = append(t, c)
   }
   for ;j < len(r); j++{
      t = append(t, r[j])
   }
   for ;i<len(l); i++{
      t= append(t, l[i])
   }
   for i:=0; i<len(t); i ++ {
      a[s+i] = t[i]
   }
}
```

 



## 案例

找出两个集合相同的元素

1 首先将两个集合快排， 然后利用归并的思想，找相同的

