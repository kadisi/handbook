---
description: golang 数据争用date races 
---

# 数据争用 date races


定义：
① 多个线程对于同一个变量、
② 同时地、
③ 进行读/写操作的现象并且
④ 至少有一个线程进行写操作。（也就是说，如果所有线程都是只进行读操作，那么将不构成数据争用） 
后果：如果发生了数据争用，读取该变量时得到的值将变得不可知，使得该多线程程序的运行结果将完全不可预测，可能直接崩溃。 
如何防止：对于有可能被多个线程同时访问的变量使用排他访问控制，具体方法包括使用mutex（互斥量）和monitor（监视器），或者使用atomic变量。


Data races are among the most common and hardest to debug types of bugs in concurrent systems. A data race occurs when two goroutines access the same variable concurrently and at least one of the accesses is a write. See the The Go Memory Model for details.

Here is an example of a data race that can lead to crashes and memory corruption:


```
func main() {
	c := make(chan bool)
	m := make(map[string]string)
	go func() {
		m["1"] = "a" // First conflicting access.
		c <- true
	}()
	m["2"] = "b" // Second conflicting access.
	<-c
	for k, v := range m {
		fmt.Println(k, v)
	}
}

```
执行
`go run -race test.go`

执行结果
```
==================
WARNING: DATA RACE
Write at 0x00c420092180 by goroutine 6:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/hashmap_fast.go:694 +0x0
  main.main.func1()
      /Downloads/test/test/test.go:20 +0x5d

Previous write at 0x00c420092180 by main goroutine:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/hashmap_fast.go:694 +0x0
  main.main()
      /Downloads/test/test/test.go:23 +0xc9

Goroutine 6 (running) created at:
  main.main()
      /Downloads/test/test/test.go:19 +0x9a
==================
2 b
1 a
Found 1 data race(s)
exit status 66

```

# 
