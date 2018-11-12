# channel

![](../../.gitbook/assets/image%20%2812%29.png)



Golang使用Groutine和channels实现了CSP\(Communicating Sequential Processes\)模型，channles在goroutine的通信和同步中承担着重要的角色。在GopherCon 2017中，Golang专家Kavya深入介绍了 Go Channels 的内部机制，以及运行时调度器和内存管理系统是如何支持Channel的，本文根据Kavya的ppt学习和分析一下go channels的原理，希望能够对以后正确高效使用golang的并发带来一些启发。

以一个简单的channel应用开始，使用goroutine和channel实现一个任务队列，并行处理多个任务。

```text
func main(){
    //带缓冲的channel
    ch := make(chan Task, 3)

    //启动固定数量的worker
    for i := 0; i< numWorkers; i++ {
        go worker(ch)
    }

    //发送任务给worker
    hellaTasks := getTaks()

    for _, task := range hellaTasks {
        ch <- task
    }

    ...
}

func worker(ch chan Task){
    for {
       //接受任务
       task := <- ch
       process(task)
    }
}
```

从上面的代码可以看出，使用golang的goroutine和channel可以很容易的实现一个生产者-消费者模式的任务队列，相比[Java](https://link.zhihu.com/?target=http%3A//lib.csdn.net/base/java), c++简洁了很多。channel可以天然的实现了下面四个特性：

* goroutine安全
* 在不同的goroutine之间存储和传输值 - 提供FIFO语义\(buffered channel提供\)
* 可以让goroutine block/unblock

那么channel是怎么实现这些特性的呢？下面我们看看当我们调用make来生成一个channel的时候都做了些什么。

### make chan

上述任务队列的例子第三行，使用make创建了一个长度为3的带缓冲的channel，channel在底层是一个hchan结构体，位于src/runtime/chan.go里。其定义如下:

```text
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

make函数在创建channel的时候会在该进程的heap区申请一块内存，创建一个hchan结构体，返回执行该内存的指针，所以获取的的ch变量本身就是一个指针，在函数之间传递的时候是同一个channel。

hchan结构体使**用一个环形队列**来保存groutine之间传递的数据\(如果是缓存channel的话\)，使用**两个list**保存像该chan发送和从改chan接收数据的goroutine，还有一个mutex来保证操作这些结构的安全。

### 发送和接收

向channel发送和从channel接收数据主要涉及hchan里的四个成员变量，借用Kavya ppt里的图示，来分析发送和接收的过程。![](https://pic1.zhimg.com/v2-c2549285cd3bbfd1fcb9a131d8a6c40c_b.gif)  
  


还是以前面的任务队列为例:

```text
//G1
func main(){
    ...

    for _, task := range hellaTasks {
        ch <- task    //sender
    }

    ...
}

//G2
func worker(ch chan Task){
    for {
       //接受任务
       task := <- ch  //recevier
       process(task)
    }
}
```

其中G1是发送者，G2是接收，因为ch是长度为3的带缓冲channel，初始的时候hchan结构体的buf为空，sendx和recvx都为0，当G1向ch里发送数据的时候，会首先对buf加锁，然后将要发送的**数据copy到buf里**，并增加sendx的值，最后释放buf的锁。然后G2消费的时候首先对buf加锁，然后将buf里的**数据copy到task变量对应的内存里**，增加recvx，最后释放锁。整个过程，G1和G2没有共享的内存，底层通过hchan结构体的buf，使用copy内存的方式进行通信，最后达到了共享内存的目的，这完全符合CSP的设计理念

> Do not comminute by sharing memory;instead, share memory by communicating

一般情况下，G2的消费速度应该是慢于G1的，所以buf的数据会越来越多，这个时候G1再向ch里发送数据，这个时候G1就会阻塞，那么阻塞到底是发生了什么呢？

### Goroutine Pause/Resume

goroutine是Golang实现的用户空间的轻量级的线程，有runtime调度器调度，与操作系统的thread有多对一的关系，相关的数据结构如下图:![](https://pic1.zhimg.com/80/v2-6b7eb0b02fb5c275492909aeabfbb428_hd.jpg)

其中M是操作系统的线程，G是用户启动的goroutine，P是与调度相关的context，每个M都拥有一个P，P维护了一个能够运行的goutine队列，用于该线程执行。

当G1向buf已经满了的ch发送数据的时候，当runtine检测到对应的hchan的buf已经满了，会通知调度器，调度器会将G1的状态设置为waiting, 移除与线程M的联系，然后从P的runqueue中选择一个goroutine在线程M中执行，此时G1就是阻塞状态，但是不是操作系统的线程阻塞，所以这个时候只用消耗少量的资源。

那么G1设置为waiting状态后去哪了？怎们去resume呢？我们再回到hchan结构体，注意到hchan有个sendq的成员，其类型是waitq，查看源码如下：

```text
type hchan struct { 
    ... 
    recvq waitq // list of recv waiters 
    sendq waitq // list of send waiters 
    ... 
} 
// 
type waitq struct { 
    first *sudog 
    last *sudog 
} 
```

实际上，当G1变为waiting状态后，会创建一个代表自己的sudog的结构，然后放到sendq这个list中，sudog结构中保存了channel相关的变量的指针\(如果该Goroutine是sender，那么保存的是待发送数据的变量的地址，如果是receiver则为接收数据的变量的地址，之所以是地址，前面我们提到在传输数据的时候使用的是copy的方式\)![](https://pic2.zhimg.com/80/v2-eb2e209ff1c84b4657c8d9862707789b_hd.jpg)

当G2从ch中接收一个数据时，会通知调度器，设置G1的状态为runnable，然后将加入P的runqueue里，等待线程执行.![](https://pic4.zhimg.com/80/v2-b57542e446915d4d86693136900c30f0_hd.jpg)

### wait empty channel

前面我们是假设G1先运行，如果G2先运行会怎么样呢？如果G2先运行，那么G2会从一个empty的channel里取数据，这个时候G2就会阻塞，和前面介绍的G1阻塞一样，G2也会创建一个sudog结构体，保存接收数据的变量的地址，但是该sudog结构体是放到了recvq列表里，当G1向ch发送数据的时候，**runtime并没有对hchan结构体题的buf进行加锁，而是直接将G1里的发送到ch的数据copy到了G2 sudog里对应的elem指向的内存地址！**

![](https://pic1.zhimg.com/80/v2-4466a9880e997d27357b778583a7e166_hd.jpg)

### 总结

Golang的一大特色就是其简单搞笑的天然并发机制，使用goroutine和channel实现了CSP模型。理解channel的底层运行机制对灵活运用golang开发并发程序有很大的帮助，看了Kavya的分享，然后结合golang runtime相关的源码\(源码开源并且也是golang实现简直良心！\),对channel的认识更加的深刻，当然还有一些地方存在一些疑问，比如goroutine的调度实现相关的，还是要潜心膜拜大神们的源码！

### 参考资料

1. [https://speakerdeck.com/kavya719/understanding-channels](https://link.zhihu.com/?target=https%3A//speakerdeck.com/kavya719/understanding-channels)
2. [https://about.sourcegraph.com/go/understanding-channels-kavya-joshi](https://link.zhihu.com/?target=https%3A//about.sourcegraph.com/go/understanding-channels-kavya-joshi)
3. [https://github.com/golang/go/blob/master/src/runtime/chan.go](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/runtime/chan.go)
4. [https://github.com/golang/go/blob/master/src/runtime/runtime2.go](https://link.zhihu.com/?target=https%3A//github.com/golang/go/blob/master/src/runtime/runtime2.go)



引用：[https://zhuanlan.zhihu.com/p/27917262](https://zhuanlan.zhihu.com/p/27917262)

[https://speakerdeck.com/kavya719/understanding-channels](https://speakerdeck.com/kavya719/understanding-channels)



