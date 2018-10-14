# interface

首先要理解duck-typing \(鸭子类型\)， 

## duck typing

“当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。”

我们并不关心对象是什么类型，到底是不是鸭子，只关心行为。

比如在python中，有很多file-like的东西，比如StringIO,GzipFile,socket。它们有很多相同的方法，我们把它们当作文件使用。

又比如list.extend\(\)方法中,我们并不关心它的参数是不是list,只要它是可迭代的,所以它的参数可以是list/tuple/dict/字符串/生成器等.  


### **1. interface 底层结构**

对于interface 的大体了解可以这么认为，interface 底层数据结构可以分为type 以及存放底层数据的指针，因此interface 底层至少包含了两个对象，type 和 数据指针。

例如



```text
func Foo(x interface{}) {
    if x == nil {
        fmt.Println("empty interface")
        return
    }
    fmt.Println("non-empty interface")
}

func main() {
    var x *int = nil
    Foo(x)
}
```

\*\*\*\*

上面的例子的输出结果如下

```text
$ go run test_interface.go
non-empty interface
```

因为对于Foo 函数，传进去的x 的interface 他的底层的type 士奇上是\*int， 所以并不是nil



根据 interface 是否包含有 method，底层实现上用两种 struct 来表示：iface 和 eface。eface表示不含 method 的 interface 结构，或者叫 empty interface。对于 Golang 中的大部分数据类型都可以抽象出来 \_type 结构，同时针对不同的类型还会有一些其他信息。

```text
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
    size       uintptr // type size
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32  // hash of type; avoids computation in hash tables
    tflag      tflag   // extra type information flags
    align      uint8   // alignment of variable with this type
    fieldalign uint8   // alignment of struct field with this type
    kind       uint8   // enumeration for C
    alg        *typeAlg  // algorithm table
    gcdata    *byte    // garbage collection data
    str       nameOff  // string form
    ptrToThis typeOff  // type for pointer to this type, may be zero
}
```

iface 表示 non-empty interface 的底层实现。相比于 empty interface，non-empty 要包含一些 method。method 的具体实现存放在 itab.fun 变量里。如果 interface 包含多个 method，这里只有一个 fun 变量怎么存呢？这个下面再细说。

```text
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

// layout of Itab known to compilers
// allocated in non-garbage-collected memory
// Needs to be in sync with
// ../cmd/compile/internal/gc/reflect.go:/^func.dumptypestructs.
type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    bad    int32
    inhash int32      // has this itab been added to hash?
    fun    [1]uintptr // variable sized
}
```

。



