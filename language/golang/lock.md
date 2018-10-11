---
description: golang 同步机制的实现
---

# lock

Golang的提供的同步机制有sync模块下的Mutex、WaitGroup以及语言自身提供的chan等。 这些同步的方法都是以runtime中实现的底层同步机制（cas、atomic、spinlock、sem）为基础的， 本文主要探讨Golang底层的同步机制如何实现。

## 1 cas、atomic

    cas\(Compare And Swap\)和原子运算是其他同步机制的基础， 在runtime/asm\_xxx.s\(xxx代表系统架构，比如amd64\)中实现。amd64架构的系统中， 主要通过两条汇编语句来实现，一个是**LOCK**、一个是**CMPXCHG**。

    **LOCK**是一个指令前缀，其后必须跟一条“读-改-写”的指令，比如INC、XCHG、CMPXCHG等。 这条指令对CPU缓存的访问将是排他的。

    **CMPXCHG**是完成CAS动作的指令。 把LOCK和CMPXCHG一起使用，就达到了原子CAS的功能。

    atomic操作也是通过**LOCK**和其他算术操作（**XADD**、**ORB**等）组合来实现。

## 2 自旋锁

    Golang中的自旋锁用来实现其他类型的锁，自旋锁的作用和互斥量类似，不同点在于， 它不是通过休眠来使进程阻塞，而是在获得锁之前一直处于忙等状态（自旋），从而避免了进程（或者

    和自旋锁相关的函数有sync\_runtime\_canSpin和sync\_runtime\_doSpin， 前者用来判断当前是否可以进行自旋，后者执行自旋操作。二者通常一起使用。

    sync\_runtime\_canSpin函数中在以下四种情况返回false

1. 已经执行了很多次
2. 是单核CPU
3. 没有其他正在运行的P
4. 当前**P**的**G**队列为空

    条件1避免长时间自旋浪费CPU的情况。

    条件2、3用来保证除了当前在运行的Goroutine之外，还有其他Goroutine在运行。

    条件4是避免自旋锁等待的条件是由当前**P**的其他**G**来触发，这样会导致 在自旋变得没有意义，因为条件永远无法触发。

    sync\_runtime\_doSpin会调用procyield函数，该函数也是汇编语言实现。 函数内部循环调用**PAUSE**指令。**PAUSE**指令什么都不做，但是会消耗CPU时间，在执行**PAUSE**指令时， CPU不会对他做不必要的优化。

## 3 信号量

    按照runtime/sema.go中的注释：

```text
Think of them as a way to implement sleep and wakeup
```

    Golang中的sema，提供了休眠和唤醒Goroutine的功能。

    semacquire函数首先检查信号量是否为0：如果大于0，让信号量减一，返回； 如果等于0，就调用goparkunlock函数，把当前Goroutine放入该sema的等待队列，并把他设为等待状态。

    semrelease函数首先让信号量加一，然后检查是否有正在等待的Goroutine： 如果没有，直接返回；如果有，调用goready函数唤醒一个Goroutine。

## 4 sync/Mutex

    Mutex拥有**Lock**、**Unlock**两个方法，主要的实现思想都体现在**Lock**函数中。

    **Lock**执行时，分三种情况：

1. **无冲突** 通过CAS操作把当前状态设置为加锁状态；
2. **有冲突** 开始自旋，并等待锁释放，如果其他Goroutine在这段时间内释放了该锁， 直接获得该锁；如果没有释放，进入3；
3. **有冲突，且已经过了自旋阶段** 通过调用semacquire函数来让当前Goroutine进入等待状态。

    无冲突时是最简单的情况；有冲突时，首先进行自旋，是从效率方面考虑的， 因为大多数的Mutex保护的代码段都很短，经过短暂的自旋就可以获得；如果自旋等待无果，就只好通过信号量来让当前 Goroutine进入等待了。

引用： [http://ga0.github.io/golang/2015/10/11/golang-sync.html](http://ga0.github.io/golang/2015/10/11/golang-sync.html)

