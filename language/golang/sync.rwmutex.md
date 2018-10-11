# sync.RWMutex

golang 读写锁 RWMutex 包含5个核心变量

```text
type RWMutex struct {
	w           Mutex  // held if there are pending writers
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers
	readerWait  int32  // number of departing readers
}
```

w 互斥锁，用来互斥写操作

writeSem 和 readersem 两个信号量分别表示 

1 给write 操作提供的信号量，用来等待完成的reader， 

2 表示给read 提供的信号量，用来等待完成的writer

首先讲一下信号量**Semaphore**

{% hint style="info" %}
It’s a data structure invented by Edsger Dijkstra and it’s useful to solve many synchronization problems. It’s an integer with two operations:

* _acquire_ \(known also as _wait_, _decrement_ or P\)
* _release_ \(_signal_, _increment_ or _V_\)

Operation _acquire_ decreases the semaphore by 1. If the result is negative then thread blocks and cannot resume until other thread will increment the semaphore. If result is positive then thread continues execution.

Operation _release_ increases the semaphore by 1. If there’re blocked threads then one of them gets unblocked.

Go’s runtime provides _runtime\_SemacquireMutex and runtime\_Semrelease_functions which are used e.g. to implement _sync.RWMutex_
{% endhint %}

信号量提供两个方法 acquire 和release

acquire 每次调用会给信号量值减一，如果值的结果是负数，会让线程阻塞住，知道其他的线程让这个信号量+1 成为正数后才会恢复

release 每次调用会让信号量+1

golang 的runtime 提供_runtime\_SemacquireMutex and runtime\_Semrelease 方法用于实现sync 的读写锁_

对于读锁： RLock



```text
func (rw *RWMutex) RLock() {
    ...
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {    
        runtime_SemacquireMutex(&rw.readerSem, false)
    }
    ...
}
```

readerCount 是一个int32值，代表了已经加锁的RLock的数量，或者是被Write block 的数量，每次只需Rlock 都会给这个值做原子性操作 + 1， 值+1 后发现如果最后的值还是小于0，说明有write操作，这时候会执行readerSem 信号量的acquire 操作，如果信号量值是负数，则会阻塞在这

接着看write 的lock 操作

Lock



```text
func (rw *RWMutex) Lock() {
    ...
    rw.w.Lock()
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {     
        runtime_SemacquireMutex(&rw.writerSem, false)
    }
    ...
}
```

首先使用w 互斥锁，排除其他的写操作

对readCounter 减去一个常量 这个常量值是 1&lt;&lt; 30

```text
rwmutexMaxReaders
```

所以一旦执行了减操作，原来RLock 的readerCounter + 1 操作的结果会是负数，会进入获取信号量的操作

同时把readCount原来的值传给r 变量

对readerWait 进行原子操作， 加上 r 即原来的readCount数目，这时候readerWait 代表了，此时这个阶段 正在加锁的Rlock 数量，如果ReadWait 数量不是0， 则WriteLock 会进入对writeSem信号量的acquire 操作，writeSem 减一后如果是负数，则会进入阻塞操作，知道writeSem 成为了正数，那什么时候writeSem 成为正数呢

看读的解锁操作，RUnlock



```text
func (rw *RWMutex) RUnlock() {
    ...
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        if r+1 == 0 || r+1 == -rwmutexMaxReaders {
            race.Enable()
            throw("sync: RUnlock of unlocked RWMutex")
        }
        // A writer is pending.
        if atomic.AddInt32(&rw.readerWait, -1) == 0 {
            // The last reader unblocks the writer.       
            runtime_Semrelease(&rw.writerSem, false)
        }
    }
    ...
}
```



RUnlock 是RLock 的反向操作，因此首先会讲ReaderCount 减一，如果结果是小于零的，说明这时候存在写锁，这时候开始给ReadWait 原子减一，减一后如果发现是0了，这说明这个操作是当时write 锁这个阶段最后一个释放的读锁，这时候开始对writeSem 信号量执行release操作，对信号量+1，这时一旦信号量是正数，原来阻塞在

```text
runtime_SemacquireMutex
```

的写锁操作，会重新恢复，这样整个写和读的锁流程算是完成



最后看写锁的解锁操作



```text
func (rw *RWMutex) Unlock() {
    ...
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
    if r >= rwmutexMaxReaders {
        race.Enable()
        throw("sync: Unlock of unlocked RWMutex")
    }
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false)
    }
    rw.w.Unlock()
    ...
}
```

写锁的解锁是写锁的上锁的反向操作，上锁时候会直接对readCount 减去一个常数，解锁会再加上这个常数，获得新的readCount 值 这时候r 代表了在write 过程中，新阻塞的read 锁，需要对readSem 信号量执行release 操作，有几个执行几遍，使用for 循环， 



引用参考

[https://medium.com/golangspec/sync-rwmutex-ca6c6c3208a0](https://medium.com/golangspec/sync-rwmutex-ca6c6c3208a0)

