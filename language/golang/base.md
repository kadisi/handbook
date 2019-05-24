# 基础
里面的结果可能会出现编译错误

## slice 

```
slice 1.求输出结果

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

解析： 数组和切片在函数中都是值传递， 只不过切片封装了底层数据结构， 实参的切片和形参赋值的切片底层指向相同的内存空间
```


```go
slice 2.求输出结果

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
slice 3.求输出结果

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
slice 4.求输出结果

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
slice 5.求输出结果

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



```go
slice 6.求输出结果

func main() {
	a := []int{1,2,3,4,5,6,7,8,9}

	fmt.Println(len(a), cap(a))

	b := a[2:4]

	fmt.Println(len(b), cap(b))

}






结果:
9 9
2 7

解析： cap 是指的这个切片的容量， a 和 b 都指向底层相同的一块内存区域， 只不过b 是从2 开始的， 长度是2 而cap 是总长度9 - 2
```

## defer


```go
defer 1.求输出结果

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
defer 2.求输出结果

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

解析： defer 后面的函数 是代表注册了将要延迟的调用， 参数值在注册的时候 会被复制并被缓存起来，因此在最后执行func 时候，参数b的值实际上是10

相比于直接调用， defer 会花费更大的代价, 因为这其中包括注册， 调用等操作，还有额外的缓存开销
```



```go
defer 3.求输出结果

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
defer 4.求输出结果

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


```go
defer 5.求输出结果:

func test() (z int) {
	defer func() {
		fmt.Println("defer", z)
		z += 100
	}()
	return 100
}
func main() {
	fmt.Println("test:", test())
}






结果:
defer 100
test: 200


解析： 首先会注册defer 后面的func，注意此时func里面没有参数
在最后retun 时候， 实际上是 z = 100
之后再执行注册的func， 这时候输出 defer 100
z 被再加了100， 这时候z 成了200 函数退出
```

## interface 接口


```go
interface 1.求输出结果

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
interface 2.求输出结果

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
interface 3.求输出结果

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


```go
interface 4.求输出结果

func main() {

	a := 10

	va, vp,vpe := reflect.ValueOf(a), reflect.ValueOf(&a), reflect.ValueOf(&a).Elem()
	fmt.Println(va.CanAddr(), va.CanSet())
	fmt.Println(vp.CanAddr(), vp.CanSet())
	fmt.Println(vpe.CanAddr(), vpe.CanSet())

}






结果：
false false
false false
true true

解析： valueof 专注于对象数据的读写， 接口变量会复制对象，并且是unaddresasble 的，所以想要修改对象，必须使用指针

但是就算是传入指针， 一样需要elem 获取目标对象， 因为被接口存储的指针本身是不能寻址和进行操作设置的。
```


```go
interface 5.求输出结果：

func main() {
	var a interface{} = nil
	var b interface{} = (*int)(nil)

	fmt.Println(a==nil)
	fmt.Println(b==nil, reflect.ValueOf(b).IsNil())

}






结果：
true
false true

解析： 接口有两种nil 状态， 一种是类型和值都为nil，就像a
另一种是类型不为nil，但是值为nil, 

接口的这两种nil状态一直是个潜在的麻烦， 解决方法是用isNil 看来判断值是否为nil

```

## 枚举 iota


```go
iota 1.求输出结果

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
iota 2.求输出结果

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
iota 3.求输出结果

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



```go
iota 4.求输出结果

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


解析: 如果中断iota， 则必须显示恢复，切后续自增值按照行序递增
如果 iota 中间出现插队的情况，那么后续的值会使用插队的值：直到重新遇到iota

```


```go

iota 5.求输出结果

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

```go

求输出结果：

const (
	a = iota
	b = iota
)
const (
	c    = iota
	d    = iota
)
func main() {
	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
	fmt.Println(d)
}




结果：
0
1
0
1


解析：

iota 是代表常量生成器， iota 代表了下一行的自增值， 默认情况下， 不打断的情况下iota 没增加一行， 会+1

iota 只能在常量表达式const 中使用。

iota 可以这么理解： iota 只能在const 表达式里起作用， 它代表了const 表达式里的行索引， 因此不管iota 在const 表达式里第几行出现 , 还是在同一个const 表达式里出现几次， 都代表了行索引

```



```go
求输出结果：

const (
	a = iota
	b = iota
)
const (
	te = "ttt"
	c = iota
	d = iota
)
func main() {
	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
	fmt.Println(d)
}




结果：
0
1
1
2

iota 在每个const 中起作用。 代表const 里的行索引， 因此a = 0 b = 1
在第二个const 中， 虽然iota 是从第二行， c 开始的， 但是c 所在的是第二行， 索引所以是1， d 第三行 索引是2
```




```go

求输出结果：
const (
	a = iota
	b = iota
)
const (
	t1 = "ttt"
	t2 = "ttt"
	c = iota
	d = iota
)
func main() {
	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
	fmt.Println(d)
}




结果：
0
1
2
3


解析： iota 代表每个const 的行索引。第二个const 里， c 是第三行， 索引是2 ， d 是第四行， 索引是3

```


```go
求输出结果：

const (
	a = iota
	b = iota
)
const (
	t1 = "ttt"
	c
	d = iota
)
func main() {
	fmt.Println(a)
	fmt.Println(b)
	fmt.Println(c)
	fmt.Println(d)
}



结果：
0
1
ttt
2


解析：在第二个const中， 由于c 没用任何赋值， 因此默认沿袭上面一行的值， 所以是ttt， d 代表了const 的行索引， 是第三行， 索引是2


```


```go
求输出结果：

const (
	t1 = "ttt"
	c
)
func main() {
	fmt.Println(c)
}





结果：
ttt




解析:c 没有做任何赋值， 沿袭上一行的值

```

## panic



```go
panic 1.求输出结果

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


```go

panic 2.求输出结果

func test()  {
	defer println("test 1")
	defer println("test 2")
	panic("i am dead")
}
func main() {
	defer func() {
		println("recover")
	}()

	test()
}






结果:
test 2
test 1
recover
panic: i am dead


解析： panic 如果在defer 中没遇到recover， 会导致当前进程退出，但是退出前， 会执行所有的延迟调用defer
注意， panic 会导致进程退出， 而不是单纯协程
```


```go
panic 3.求输出结果:


func test()  {
	defer println("test 1")
	defer println("test 2")
	panic("i am dead")
}
func main() {
	defer func() {
		println("recover")
		recover()
	}()

	test()
}






结果:
test 2
test 1
recover

解析： panic 会被recover方法捕获到， 因为单纯只执行了recover方法， 没有对recover 返回值做处理， 因此panic 的报错没有输出
```



```go
panic 4.求输出结果
func main() {
	defer func() {
		err := recover()
		if err != nil {
			fmt.Println("recover err != nil ", err)
		} else {
			fmt.Println("recover err == nil")
		}
	}()

	defer func() {
		panic("you am dead")
	}()

	panic("i am dead")
}






结果:
recover err != nil  you am dead


解析: 连续调用panic ， 仅仅最后一个会被捕获

```

```go
panic 5.求输出结果

func main() {
	defer recover()

	panic("i am dead")
}






结果:
panic: i am dead

解析： recover方法必须在defer 注册的方法里才能生效

```

```go
panic 6.求输出结果

func main() {
	defer func() {
		recover()
	}()

	panic("i am dead")
}






结果：
什么也不输出

```

```go

panic 7.求输出结果， 并请问进程退出还是协程退出

func test(){
	time.Sleep(time.Second * 2)
	panic("w")
}

func main() {
	go test()

	for ; ;  {
		time.Sleep(time.Second)
	}
}






结果：
panic: w
Process finished with exit code 2

解析： 进程退出， panic 会导致进程退出

```

## foreach


```go
foreach 1.求输出结果

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



```go

foreach 2.请问每行输出的结果是否一样

func main() {
	data := []int{1, 2, 3}
	for i, v := range data {
		fmt.Printf("%p %p\n", &i, &v)
	}
}






结果：
一样

解释:无论是普通for 循环 还是range 迭代，其定义的局部变量都会被重复使用

```

```go
foreach 3.求输出结果

func main() {
	data := [3]int{1, 2, 3}
	for i, v := range data {
		if i == 0 {
			data[0] = 100
			data[1] = 100
			data[2] = 100
		}
		fmt.Printf("v %v data %v\n", v, data[i])
	}
}






结果:
v 1 data 100
v 2 data 100
v 3 data 100

解释： rang 会直接复制目标数据， 这种情况对数据会收到直接的影响。 因此v 是 1 2 3
若摈弃这种情况， 我们可以使用数组指针或者切片
```

```go
foreach 4.求输出结果

func main() {
	data := []int{1, 2, 3}
	for i, v := range data {
		if i == 0 {
			data[0] = 100
			data[1] = 100
			data[2] = 100
		}
		fmt.Printf("v %v data %v\n", v, data[i])
	}
}






结果:
v 1 data 100
v 100 data 100
v 100 data 100

```


## map 字典

```go
map 1.求输出结果

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


```go
map 2.求输出结果

type user struct {
	Age int
}

func main() {
	A := make(map[int]user)
	A[1] = user{Age: 1}
	A[1].Age += 1
	fmt.Println(A)
}






结果：
cannot assign to struct field A[1].Age in map

解析： 因为涉及到内存访问安全和哈希算法等缘故， 字典被设计成 not addressable ， 故不能直接修改value成员（包括结构体和数组）

```

## waitGroup



```go
waitGroup 1.求输出结果

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
select and chan 1.求输出结果

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

```go
求输出结果

func test() bool {
	return true
}
func main() {

	ticker := time.NewTicker(time.Second * 2)
	stop := make(chan struct{})
	go func() {
		time.Sleep(time.Second * 5)
		stop <- struct{}{}
	}()

	defer func() {
		fmt.Println("defer here")
	}()

	for test() {
		select {
			case <-ticker.C:
				fmt.Println("time here")
			case <-stop:
				fmt.Println("receive stop single")
				break
		}
	}
	fmt.Println("main end")
}



结果:

time here
time here
receive stop single
time here
time here
time here
.
.
.
time here 最后会无限循环

解析： 本题主要考察select 里的break 是跳出select 而不是for 循环
```


```go

求输出结果

func test() bool {
	return true
}
func main() {

	ticker := time.NewTicker(time.Second * 2)
	stop := make(chan struct{})
	go func() {
		time.Sleep(time.Second * 5)
		stop <- struct{}{}
	}()

	defer func() {
		fmt.Println("defer here")
	}()

	loop:
	for test() {
		select {
			case <-ticker.C:
				fmt.Println("time here")
			case <-stop:
				fmt.Println("receive stop single")
				break loop
		}
	}
	fmt.Println("main end")
}



结果：
time here
time here
receive stop single
main end
defer here


解析： break loop 后会直接跳到loop 所在的位置， 但是loop 下一行的for 循环不执行
这个做法解决了， for 和select 配合使用时候， 用break 直接跳出for 循环的问题

```

## make



```go
make 1.求输出结果

func main() {

	cache := make([]int, 1)
	
	cache = append(cache, 2)
	
	fmt.Println(cache)
}






结果：
[0 2]
```

## goto 

```go
goto 1.求输出结果

func main() {
	for i:=0; i< 3; i ++ {
		loop:
			fmt.Println(i)
	}

	goto loop
}






结果：
goto loop jumps into block 

解析： goto 不能跳转到其他函数或者内层的代码块中， 本文编译时候会报错
```


## break

```go
break 1.求输出结果

func main() {
	for i:=0; i< 3; i ++ {
		inter:
		for j :=0; j < 3 ; j ++ {
			for z :=0 ; z <3 ; z ++ {
				if z == 1 {
					break inter
				}
				fmt.Println(i, j, z)
			}
			fmt.Println(i, j)
		}
		fmt.Println(i)
	}
	fmt.Println("end")
}






结果:
0 0 0
0
1 0 0
1
2 0 0
2
end

解析： break 只能跳到上级或者更上级的标签， 调到上级的标签后， 标签下的for 循环不再执行
而且 这种标签下面只能紧接着for循环

```


## continue

```go
continue 1.求输出结果

func main() {

	inter:
	for i:=0; i< 3; i ++ {
		for j :=0; j < 3 ; j ++ {
			if j == 1 {
				continue inter
			}
			fmt.Println(i, j)
		}
		fmt.Println(i)
	}
	fmt.Println("end")
}






结果:
0 0
1 0
2 0
end

解析： Go 语言的 continue 语句 有点像 break 语句。但是 continue 不是跳出循环，而是跳过当前循环执行下一次循环语句。
for 循环中，执行 continue 语句会触发for增量语句的执行。

这里continue 到标签后， 会不再执行标签下的当前循环，进而执行下一个循环
```


## 闭包

```go

闭包 1.求输出结果：

func test(data []int) []func() {
	value := make([]func(), 0, 3)

	for _, v := range data {
		value = append(value, func() {
			fmt.Println(&v, v)
		})
	}
	return value
}
func main() {

	value := test([]int{1,2,3})
	for _, f := range value {
		f()
	}
}






结果:
0xc420082008 3
0xc420082008 3
0xc420082008 3

解析： 地址和值都是一样的
闭包 是函数和引用环境的组合体

在rang 中v 的变量是复用的， 因此append 的时候在func 里村的是v这个变量的引用， 而v 最后的值是3

要想不一样， 则需要更改一下for 循环

	for _, v := range data {
		x := v
		value = append(value, func() {
			fmt.Println(&x, x)
		})
	}

```


```go

闭包 2.求输出结果

func test(data int) (func(), func()) {
	return func() {
		fmt.Println(data)
		data += 10
	}, func() {
		fmt.Println(data)
	}
}
func main() {
	a, b := test(10)
	a()
	a()
	b()
}






结果：
10
20
30

解析：闭包是函数和引用环境的组合体 多个匿名函数引用同一个环境变量，会让事情变得复杂，任何修改行为都会影响其他函数取值
这在并发模式下需要做同步处理

```

## string 字符串

```go

string 1.求输出结果

func main() {
	s := "1234"
	s[0] = '0'
	fmt.Println(s)
}






结果:
cannot assign to s[0]

解析： string 是不可变的byte 序列， 不能修改里面的值


```

## struct 结构体

```go
struct 1.求输出结果


type user struct {
	Age int
}

func main() {
	a := user{
		Age: 1,

	}

	b := user{
		Age: 1,
	}

	if  a == b {
		fmt.Println("==")
	} else {
		fmt.Println("!=")
	}
}






结果：
==

```


```go
struct 2.求输出结果

type user struct {
	Age *int
}

func main() {
	c := 1
	a := user{
		Age: &c,

	}

	b := user{
		Age: &c,
	}

	if  a == b {
		fmt.Println("==")
	} else {
		fmt.Println("!=")
	}
}






结果:
==

```


```go
struct 3.求输出结果

type user struct {
	Age *int
}

func main() {
	c := 1
	a := user{
		Age: &c,

	}
	d := 1
	b := user{
		Age: &d,
	}

	if  a == b {
		fmt.Println("==")
	} else {
		fmt.Println("!=")
	}
}






结果：
!=

```

```go
struct 4.求输出结果

type user struct {
	Age int
	cache map[int]int
}

func main() {
	a := user{
		Age: 1,
		cache: map[int]int{1:1},
	}
	b := user{
		Age: 1,
		cache: map[int]int{1:1},
	}

	if  a == b {
		fmt.Println("==")
	} else {
		fmt.Println("!=")
	}
}






结果：
invalid operation: a == b (struct containing map[int]int cannot be compared)


解析： 只有结构体的所有字段支持相等操作时候， 才可以做相等操作

map 并不支持 ，所以编译报错

```



```go
struct 5.求输出结果：

type user struct {
	Age int
}

func (u user) Add() {
	u.Age += 1
}

func (u *user) Add1() {
	u.Age += 1
}

func main() {

	a := user{
		Age: 30,
	}

	a.Add()
	fmt.Println(a.Age)

	a.Add1()
	fmt.Println(a.Age)
}






结果
30
31

解析： 
方法和函数 定义的语法区别在于 前者有前置实例接受参数（receiver）, 编译器以此确定方法所属类型

方法可以看做特殊的函数， 那么receiver 的类型自然可以是基础类型也可以是指针类型， 这会关系到调用对象实例是否被复制

如果是基础类型， 则会被复制， 如果是指针类型， 则不会被复制， 直接取地址

可以使用实例值或者指针调用方法， 编译器会根据方法 的receiver 类型 自动在基础类型和指针类型做转换

如何选择方法的reveiver 类型：

要修改实例状态 用*T
无需修改实例状态的小对象或者固定值 用T
大对象建议用 *T， 减少复制成本
引用类型， 字符串， 函数等指针包装对象， 用T
若包含Mutex 等同步字段， 用*T, 避免因赋值造成锁无效情况
其他无法确定的情况，用*T

```

