---
计算机知识: 算法 其他
---

# 全排列 

全排列就是从第一个数字起每个数分别与它后面的数字交换

```

func Arrange1(s []int, begin int) {
	if begin == len(s) -1 {
		fmt.Printf("%v\n", s)
		arr1 ++
		return
	} 
	if begin >= len(s) {
		return
	}
	for i := begin ; i <= len(s)-1; i ++ {
		s[begin],s[i] = s[i],s[begin]
		Arrange1(s, begin +1)
		s[begin],s[i] = s[i],s[begin]
	}
}



```



