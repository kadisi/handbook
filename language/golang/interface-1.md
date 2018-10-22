# 接口1

> ```text
> package main
>
> import (
> 	"fmt"
> )
>
> type People interface {
> 	Speak(string) string
> }
>
> type Stduent struct{}
>
> func (stu *Stduent) Speak(think string) (talk string) {
> 	if think == "bitch" {
> 		talk = "You are a good boy"
> 	} else {
> 		talk = "hi"
> 	}
> 	return
> }
>
> func main() {
> 	var peo People = Stduent{}
> 	think := "bitch"
> 	fmt.Println(peo.Speak(think))
> }
> ```

以上代码会报错：

```text
cannot use Stduent literal (type Stduent) as type People in assignment:
	Stduent does not implement People (Speak method has pointer receiver)
```

这是因为Speak 的receiver 是一个指针，而不是一个对象，Speak 的实现是\*Student 类型， 不是Student 类型， 所以报错

如果我们把main 函数里的Student{} 改成&Student 则编译通过

