# iota

iota 代表了下一行自增长, 默认情况下，不打断的情况下iota 没增加一行，会+1

## iota 只能在常量表达式const {} 中使用

```text
fmt.Println(iota)  
编译错误： undefined: iota
```

## 每次const 出现时候， 都会让iota 初始化为0

```text
const a = iota // a=0 
const ( 
  b = iota     //b=0 
  c            //c=1 
)


```

## 自定义类型

自增长常量经常包含一个自定义枚举类型，允许你依靠编译器完成自增设置。

```go
type Stereotype int

const ( 
    TypicalNoob Stereotype = iota // 0 
    TypicalHipster                // 1 
    TypicalUnixWizard             // 2 
    TypicalStartupFounder         // 3 
)
```



## 可跳过的值

我们可以使用下划线跳过不想要的值。

```text
type AudioOutput int

const ( 
    OutMute AudioOutput = iota // 0 
    OutMono                    // 1 
    OutStereo                  // 2 
    _ 
    _ 
    OutSurround                // 5 
)


```

## 定义在一行的情况

```text
const (
    Apple, Banana = iota + 1, iota + 2
    Cherimoya, Durian
    Elderberry, Fig
)
```

```text
// Apple: 1 
// Banana: 2 
// Cherimoya: 2 
// Durian: 3 
// Elderberry: 3 
// Fig: 4
```

```text
const (
    Apple, Banana = iota + 1, iota + 2
    Cherimoya, Durian   // = iota + 1, iota + 2 
    Elderberry, Fig     //= iota + 1, iota + 2
)
iota 在下一行增长，而不是立即取得它的引用。
// Apple: 1 
// Banana: 2 
// Cherimoya: 2 
// Durian: 3 
// Elderberry: 3 
// Fig: 4
```



## 中间插队

如果 `iota` 中间出现插队的情况，那么后续的值会使用插队的值：直到重新遇到iota

```text
const ( 
    i = iota 
    j = 3.14 
    k = iota 
    l 
)

那么打印出来的结果是 i=0,j=3.14,k=2,l=3
```

如果 `iota` 中间出现插队的情况，那么后续的值会使用插队的值

```text
const (
	x = iota
	y
	z = "zz"
	k
	p = iota
)
func main() {
	fmt.Println(x,y,z,k,p)
}

// 打印结果  0 1 zz zz 4
```

```text
const (
	A, B int = iota, iota+1
	_, _
	C, D
	E int = iota+10
)
fmt.Println(A, B, C, D, E)
// 0 1 2 3 13
```

