
## slice 


```
1.求输出结果

func Change(a []int) {
	a[0] = 1
}

func Change1(a [3]int) {
	a[0] = 1
}

func main() {
	a := []int{9, 8, 7}
	b := [3]int{9, 8, 7}

	Change(a)
	fmt.Println(a)

	Change1(b)
	fmt.Println(b)
}

结果：
[1 8 7]
[9 8 7]
```



```go
2.求输出结果

func main() {
	A := make([]int, 0, 3)

	for i := 0; i < 3; i++ {
		A[i] = i
	}

	fmt.Println(A)
}

结果:
panic: runtime error: index out of range
```


```go
3.求输出结果

func main() {
	A := make([]int, 3, 3)

	for i := 0; i < 3; i++ {
		A[i] = i
	}

	B := A[0:2]
	fmt.Println(B)
}

结果:
[0 1]
```



```go
4.求输出结果

func Change(a []int) {
	a[0] = 9
}

func main() {
	A := make([]int, 3, 3)

	for i := 0; i < 3; i++ {
		A[i] = i
	}

	B := A[0:2]

	Change(B)
	fmt.Println(A)
}

结果：
[9 1 2]
```


```go
5.求输出结果

func Change(a []int) {
	a[0] = 9
}

func main() {
	A := make([]int, 3, 3)

	for i := 0; i < 3; i++ {
		A[i] = i
	}

	B := A[0:2]

	A = append(A, []int{3, 4, 5}...)

	Change(B)
	fmt.Println(A)
}

结果：
[0 1 2 3 4 5]
```

## defer



```go
6.求输出结果

func test(a int) (value int) {

	defer func() {
		value = a + 1
	}()

	return a
}

func main() {
	fmt.Println(test(10))
}

结果：
11
```



```go
7.求输出结果

func test(a int) (value int) {

	defer func(b int) {
		value = b + 1
	}(a)
	a = a + 10

	return a
}

func main() {
	fmt.Println(test(10))
}

结果：
11
```



```go
8.求输出结果

func test(a int) (value int) {

	a = a + 10
	defer func(b int) {
		value = b + 1
	}(a)

	return a
}

func main() {
	fmt.Println(test(10))
}

结果：
21
```

```go
9.求输出结果

func test(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
	a := 1
	b := 2
	defer test("1", a, test("10", a, b))
	a = 0
	defer test("2", a, test("20", a, b))
	b = 1
}

结果：
10 1 2 3
20 0 2 2
2 0 2 2
1 1 3 4
```

## interface



```go
10.求输出结果

type Animal interface {
	Eat()
}

type Cat struct {
	Name string
}

func (c *Cat) Eat() {
	fmt.Println("eat")
}

func Factory(a Animal) {
	a.Eat()
}

func main() {
	c := Cat{}

	Factory(c)
}

结果：
编译错误
cannot use c (type Cat) as type Animal in argument to Factory:
Cat does not implement Animal (Eat method has pointer receiver)
```



```go
11.求输出结果

type Animal interface {
}

type Cat struct {
}

func Factory(a Animal) {
	if a != nil {
		fmt.Println("not nil")
	} else {
		fmt.Println("nil")
	}
}

func main() {
	var c *Cat
	Factory(c)
}

结果：
not nil
```



```go
12.求输出结果

type Animal interface {
}

func Factory(a Animal) {
	if a != nil {
		fmt.Println("not nil")
	} else {
		fmt.Println("nil")
	}
}

func main() {
	Factory(nil)
}

结果：
nil
```


## 枚举 iota


```go
13.求输出结果

const (
    A, B = iota, iota + 1
    C, D
    E, F
)

func main() {
    fmt.Println(A, B, C, D, E, F)
}

结果：
0 1 1 2 2 3
```



```go
14.求输出结果

const (
	A, B = iota, iota + 1
	_, _
	C, D
)

func main() {
	fmt.Println(A, B, C, D)
}

结果：
0 1 2 3
```


```go

const (
	_, _ = iota, iota * 10
	a, b
	c, d
)
func main() {
	fmt.Println(a, b, c, d)

}

结果：
1 10 2 20
```


如果中断iota， 则必须显示恢复，切后续自增值按照行序递增

```go

const (
	 a = iota
	 b
	 c = 100
	 d
	 e
	 f = iota
	 g
)
func main() {
	fmt.Println(a, b, c, d, e, f, g)

}

结果:
0 1 100 100 100 5 6
```


```go

求输出结果
const (
	read byte = 1 << iota
	write
	exec
)
func main() {
	fmt.Printf("%b\n",read)
	fmt.Printf("%b\n",write)
	fmt.Printf("%b\n",exec)
}
结果：
1
10
100

```

## panic



```go
15.求输出结果

func main() {
    defer func(){fmt.Println("1")}()
    defer func(){fmt.Println("2")}()
    defer func(){fmt.Println("3")}()
    panic(4)
}

结果：
3
2
1
panic: 4
```

## foreach



```go
16.求输出结果

const (
	KEY = "wocloud"
)

func Get(cache map[string]int) *int {
	for key, v := range cache {
		if key == KEY {
			return &v
		}
	}
	return nil
}
func main() {
	cache := make(map[string]int)
	cache[KEY] = 2019

	value := Get(cache)
	if value != nil {
		*value = 2020
	}

	fmt.Println(cache[KEY])
}

结果：
2019
```

## map

```go
求输出结果

func main() {
	a := new(map[string]string)
	b := *a
	b["name"] = "zhangjie"

	fmt.Println(b)
}

结果
panic: assignment to entry in nil map

解释：
map 的初始化需要用make 去初始化， golang 中有三种引用类型 slice map chan ，这三种数据的初始化需要用make 操作

如果用new 关键字， 仅仅是分配了map 类型本身，所需要的内存， 但是并没有分配map底层键值存储的内存，所以会报错。

```

## waitGroup



```go
17.求输出结果

func main() {

	group := sync.WaitGroup{}

	for i := 0; i < 10; i++ {
		group.Add(1)
		go func() {
			defer func() {
				group.Done()
			}()
			fmt.Println(i)
		}()

	}

	group.Wait()
}

结果：
输出为0-10中随机整数
```

## select and chan



```go
18.求输出结果

func main() {

	c1 := make(chan int, 1)

	select {
		case a := <- c1:
			fmt.Println(a)
	}

	c1 <- 1
}

结果：
fatal error: all goroutines are asleep - deadlock!
```

## make



```go
19.求输出结果

func main() {

	cache := make([]int, 1)
	
	cache = append(cache, 2)
	
	fmt.Println(cache)
}

结果：
[0 2]
```


## 设计题

20.设计一个线程安全的map

## 算法题

21.算法题


假设联通员工总人数是26w，年底绩效考核，员工的最低考核成绩为30.00, 最高考核成绩为98.00，考核成绩精度为小数点后两位。请自行设计结构体，给员工进行绩效排序。  


