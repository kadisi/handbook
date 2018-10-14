---
description: 解释golang 中select 和switch 的区别
---

# select 和switch

引用：[https://colobu.com/2017/07/07/select-vs-switch-in-golang/](https://colobu.com/2017/07/07/select-vs-switch-in-golang/)



select 主要是针对channel 操作的，select case 可以同事监控多个channel 的读写信号， 只要有一个channel 有io 操作，就会触发select 的case ，但是如果有多个channel 同时有io 操作时候， select 会随机处理一个case，如果多个case 都没有io 操作，select 会处理default 操作

而switch case 默认是顺序执行的，如果case 里没有break 语句的话



`select` 和 `switch` 是 Go语言中进行分支操作的两个方式，各有各的应用场景。

#### select {#select}

`select`只能应用于channel的操作，既可以用于channel的数据接收，也可以用于channel的数据发送。

如果`select`的多个分支都满足条件，则会随机的选取其中一个满足条件的分支， 如语言规范中所说：

> If multiple cases can proceed, a uniform pseudo-random choice is made to decide which single communication will execute.

｀case｀语句的表达式可以为一个变量或者两个变量赋值。

有`default`语句。

下面的代码是 [go by example 上的例子](https://gobyexample.com/select):

```text
package main
import "time"
import "fmt"
func main() {
    c1 := make(chan string)
    c2 := make(chan string)
    go func() {
        time.Sleep(time.Second * 1)
        c1 <- "one"
    }()
    go func() {
        time.Sleep(time.Second * 2)
        c2 <- "two"
    }()
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-c1:
            fmt.Println("received", msg1)
        case msg2 := <-c2:
            fmt.Println("received", msg2)
        }
    }
}
```

#### switch {#switch}

`switch`可以为各种类型进行分支操作， 设置可以为接口类型进行分支判断\(通过i.\(type\)\)。

`switch` 分支是顺序执行的，这和`select`不同。

```text
package main
import "fmt"
import "time"
func main() {
    i := 2
    fmt.Print("Write ", i, " as ")
    switch i {
    case 1:
        fmt.Println("one")
    case 2:
        fmt.Println("two")
    case 3:
        fmt.Println("three")
    }
    switch time.Now().Weekday() {
    case time.Saturday, time.Sunday:
        fmt.Println("It's the weekend")
    default:
        fmt.Println("It's a weekday")
    }
    t := time.Now()
    switch {
    case t.Hour() < 12:
        fmt.Println("It's before noon")
    default:
        fmt.Println("It's after noon")
    }
    whatAmI := func(i interface{}) {
        switch t := i.(type) {
        case bool:
            fmt.Println("I'm a bool")
        case int:
            fmt.Println("I'm an int")
        default:
            fmt.Printf("Don't know type %T\n", t)
        }
    }
    whatAmI(true)
    whatAmI(1)
    whatAmI("hey")
}
```

