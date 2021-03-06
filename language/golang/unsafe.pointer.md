---
description: 本文着重介绍golang 的底层unsafe Pointer 类型
---

# unsafe.Pointer

首先golang 的unsafe 包 最终是在编译阶段生效的 ，unsafe 包暴露了golang 底层的一些实现细节，虽然golang 是跨平台的， 但是每个平台上golang 底层实现是不太一样的，这就导致了不同平台上unsafe表现有所不同，unsafe 广泛用于和操作系统交互的低级包中， 比如os，runtime，syscall net 等等。

golang 的unsafe point 类似于C语言的指针，在go语言中，对于指针有严格的限制， 比如go指针不能参与算术运算， 对于两个不太相关的go类型， 他们的值不太可能互相转换。

这种现象 我们称之为类型安全的指针， 因为他隐藏了底层的一些细节。但是实际上到 go 1.11 ，go语言支持这种type-unsafe的指针， 它并没有上文提到的类型安全的限制，但是type-safe 的指针也可以调用type-unsafe 指针，golang 的unsafe point 比较像c 的pointer，在另一方面， 使用type-unsafe 指针也可以写一些高效的代码， 但是也很可能会写一些无效的代码。 所以平时要慎用unsafe。Pointer

unsafe  pointer 可以简单的理解为跟之前c 语言struct 的包对齐有关系

首先unsfate 包提供了三个标准的方法

* Alignof

  我们可以使用它或者value 的地址， 注意， 相同类型的结构字段值和非结构字段值的对齐方有可能不一样， 但是对于标准的golang 编译器， 这是医院的

* Offsetof

  我们用这个方法获得结构体变量某个字段的地址偏移量

* Sizeof

  或者这个变量的size

读到这里，我们很可能会联想到c 语言结构体的对齐,

例如



```text
func test() {
   type Student struct {
      Age int8
      BirthDay int64
      IsMale bool
   }
   type Student1 struct {
      BirthDay int64
      IsMale bool
      Age byte
   }

   s := Student{
      Age: 1,
      IsMale: false,
      BirthDay: 34,
   }
   s1 := Student1{
      Age: 1,
      IsMale: false,
      BirthDay: 34,
   }

   fmt.Printf("siezeof s %v\n", unsafe.Sizeof(s))
   fmt.Printf("siezeof s1 %v\n", unsafe.Sizeof(s1))

}

// 最终结果
siezeof s 24
siezeof s1 16
```

##  unsafe 指针 相关的转换规则

###  **Pattern 1:Convert \*T1 To Unsafe Poniter, Then Convert The Unsafe Pointer Value To \*T2**

简单的例子



```text
func testconversion() {
   type Student struct {
      Age int32
      IsMale bool
   }

   type Zoon struct {
      Nums int32
      IsPublic bool
   }

   s := &Student{
      Age: 32,
      IsMale: true,
   }

   z := (*Zoon)(unsafe.Pointer(s))

   fmt.Printf("s %v\n", *s)
   fmt.Printf("z %v\n", *z)

}
// 输出结果
s {32 true}
z {32 true}
```

 首先变\*T 转换成unsafe Pointer， 之后在转换成\*T1

基本上原理是计算地址和偏移量，之后把对应内存地址覆盖到另一个变量中

另一个例子  比较string 与\[\]byte是否相等，而不进行分配内存，



```text
func unsafeEqual(a string, b []byte) bool {
    bbp := *(*string)(unsafe.Pointer(&b))
    return a == bbp
}
```

### **Pattern 2: Convert Unsafe Pointer To Uintptr, Then Use The Uintptr Value**

\*\*\*\*

### **Pattern 3: Convert Unsafe Pointer To Uintptr, Do Arithmetic Operations With The Uintptr Value, Then Convert Back**

\*\*\*\*

\*\*\*\*

\*\*\*\*

### **Pattern 4: Convert Unsafe Pointer To uintptr When Calling syscall.Syscall**

\*\*\*\*

\*\*\*\*

### **Pattern 5: Convert The uintptr Result Of reflect.Value.Pointer Or reflect.Value.UnsafeAddr Method Call To Unsafe Pointer**

\*\*\*\*

\*\*\*\*

### **Pattern 6: Convert A reflect.SliceHeader.Data Or reflect.StringHeader.Data Field To Unsafe Pointer, And The Inverse.**

\*\*\*\*

\*\*\*\*

本文文章最终参考与：[https://go101.org/article/unsafe.html](https://go101.org/article/unsafe.html)

## Type-Unsafe Pointers

We have learned Go pointers from the article [pointers in Go](https://go101.org/article/pointer.html). From that article, we know that, comparing to C pointers, there are many restrictions made for Go pointers. For example, Go pointers can't participate arithmetic operations. And for two arbitrary pointer types, it is very possible that their values can't be converted to each other.

The pointers explained in that article are called type-safe pointers actually. Although the restrictions on type-safe pointers really make us be able to write safe Go code with ease, they also make some obstacles to write efficient code in some scenarios.

In fact, up to currrent \(Go 1.11\), Go also supports type-unsafe pointers, which are pointers without the restrictions made for safe pointers. Type-unsafe pointers are also called unsafe pointers in Go. Go unsafe pointers are much like C pointers, they are powerful, and also dangerous. For some scenarios, we can write more efficient code with the help of unsafe pointers. On the other hand, it is easy to write invalid code which is subtle and difficult to detect when using unsafe pointers.

The biggest risk of using unsafe pointers comes from the fact that the unsafe mechanism is not protected by [the Go 1 compatibility guidelines](https://golang.org/doc/go1compat). Code depending on unsafe pointers works today could break since a later Go version.

If you really desire the code efficient improvements by using unsafe pointers for any reason, not only should you know the above mentioned risks, but also should you follow the instructions written in official Go documentations and clearly understand the effects of each unsafe pointer use, so that you can write safe Go code with unsafe pointers, at least if you use the standard Go compiler in Go SDK 1.11.

#### About The `unsafe` Standard Package

Go provides a special kind of types for unsafe pointers. We must import [the `unsafe` standard package](https://golang.org/pkg/unsafe/) to use unsafe pointers. The `unsafe.Pointer` type is defined as

```text
type Pointer *ArbitraryType
```

Surely, it is not a usual type definition. Here the `ArbitraryType` just hints that a `unsafe.Pointer` value can be converted to any safe pointer values in Go, and vice versa. In other words, `unsafe.Pointer` is like the `void*` in C.

Go unsafe pointers mean the types whose underlying types are `unsafe.Pointer`.

The zero values of unsafe pointers are also represneted with the predelcared identifier `nil`.The `unsafe` standard package also provides three functions.

* `func Alignof(variable ArbitraryType) uintptr`, which is used to get the address alignment of a value. Please notes, the aligns for struct-field values and non-field values of the same type may be different, though for the standard Go compiler, they are always the same.
* `func Offsetof(selector ArbitraryType) uintptr`, which is used to get the address offset of a field in a struct value. The offset is relative to the address of the struct value. The return results should be always the same for the same corresponding fields of the struct values of the same type in the same program.
* `func Sizeof(variable ArbitraryType) uintptr`, which is used to get the size of a value \(a.ka.a, the size of the type of the value\). The return results should be always the same for all values of the same type in the same program.

Note that the types of the return results of the three functions are all `uintptr`. Below we will learn that `uintptr` values can be converted to `unsafe.Pointer`, and vice versa.

Although the return results of the three functons are consistent in the same program, they may be different crossing operation systems of different architectures, crossing compilers, and crossing compiler versions.

Calls to the three function are always evaluated at compile time. Their results can be assigned to constants.An example of using the three functions.

```text
package main

import "fmt"
import "unsafe"

func main() {
	var x struct {
		a int64
		b bool
		c string
	}
	const M, N = unsafe.Sizeof(x.c), unsafe.Sizeof(x)
	fmt.Println(M, N)   // 16 32
	
	fmt.Println(unsafe.Alignof(x.a)) // 8
	fmt.Println(unsafe.Alignof(x.b)) // 1
	fmt.Println(unsafe.Alignof(x.c)) // 8
	
	fmt.Println(unsafe.Offsetof(x.a)) // 0
	fmt.Println(unsafe.Offsetof(x.b)) // 8
	fmt.Println(unsafe.Offsetof(x.c)) // 16
}
```

Please note, the above print results are for the standard Go compiler 1.11 on Linux amd64 OSes.

#### Unsafe Pointers Related Conversion Rules

Go compilers allow the following explicit conversions.

* A safe pointer can be explicitly converted to an unsafe pointer, and vice versa.
* An uintptr value can be explicitly converted to an unsafe pointer, and vice versa.

By using these conversions, we can do conversions between any two safe pointers, between any two uintptr values, and between an arbitrary safe pointer and an uintptr value. Unsafe pointers act as bridges in these conversions.

However, although these conversions are all legal at compile time, not all of them are valid \(safe\) at run time. These conversions defeat the memory safety the whole Go type system \(except the unsafe part\) tries to maintain. We must follow the instructions listed in a later section below to write valid Go code with unsafe pointers.

#### Some Facts In Go We Should Know

Before introducing the valid unsafe pointer use patterns, we need to know some facts in Go.

**Fact 1: Unsafe Pointers Are Pointers And Uintptr Values Are Intergers**

Each of non-nil safe and unsafe pointers references another value. However uintptr values don't reference any values, they are just plain integers, though often each of them stores a memory address.

Go is a language supporting automatic garbage collection. When a Go program is running, Go runtime will[check which values are not used any more and collect the memory](https://go101.org/article/memory-block.html#when-to-collect) allocated for these unused values, from time to time. Pointers play an important role in the check process. If a value is unreachable from \(referenced by\) any values still in using, then Go runtime thinks it is an unuse value and it can be safely garbage collected.

As uintptr values are integers, they can participate arithmetic operations.

The example in the next subsection shows the differences between pointers and uintptr values.

**Fact 2: Unused Values May Be Collected At Any Time**

At run time, the garbage collector may run at any time, so when a value becomes unused, the [memory block](https://go101.org/article/memory-block.html) allocated for unused values may be [collected at any time](https://go101.org/article/memory-block.html#when-can-collect).For example:

```text
package main

import "fmt"
import "unsafe"

var x *int
var y unsafe.Pointer
var z uintptr

func main() {
	var vx, vy, vz int
	x = &vx
	y = unsafe.Pointer(&vy)
	z = uintptr(unsafe.Pointer(&vz))
	
	// At the time, even if its address of vz is stored in z.
	// vz has already become unused. Gargage collector can
	// collect the memory allocated for it now.
	// On the other hand. vx and vy are still in using,
	// for they are reachable from the x and y pointers.
	
	// uinptr can be used as operands of arithmetic operators.
	z++
	z &^= 1 // <=> z = z &^ 1
	
	fmt.Println(x, y, z)
}
```

**Fact 3: We Can Use A runtime.KeepAlive Function Call To Mark A Value As Still In Using \(Reachable\) Currently**

To mark a value still reachable, we should pass another value which references the value as the argument a `runtime.KeepAlive` function call. A pointer to the value is often used as such an argument.In the following code, a small modification is made on the example in the last subsection.

```text
func main() {
	var vx, vy, vz int
	x = &vx
	y = unsafe.Pointer(&vy)
	z = uintptr(unsafe.Pointer(&vz))
	
	// do other things ...
	
	// vz is still reachable at least up to here, so
	// it will not be garbage collected now for sure.
	runtime.KeepAlive(&vz)
}
```

**Fact 4: \*unsafe.Pointer Is A General Safe Pointer Type**

Yes, `*unsafe.Pointer` is a safe unnamed pointer type. Its base type is `unsafe.Pointer`. As it is a safe pointer, accroding the conversion rules listed above, it can be converted to `unsafe.Pointer` type, and vice versa.For example:

```text
package main

import "unsafe"

func main() {
	x := 123 // of type int
	p := unsafe.Pointer(&x)
	pp := &p // of type *unsafe.Pointer
	p = unsafe.Pointer(pp)
	pp = (*unsafe.Pointer)(p)
}
```

#### How To Use Unsafe Pointers Correctly

The `unsafe` standard package documentation lists [six unsafe pointer use patterns](https://golang.org/pkg/unsafe/#Pointer). Following will introduce and explain them one by one.

**Pattern 1: Convert \*T1 To Unsafe Poniter, Then Convert The Unsafe Pointer Value To \*T2**

As above has mentioned, by using the unsafe pointer conversion rules above, we can convert a value of `*T1` to type `*T2`, where `T1` and `T2` are two arbitrary types. However, we should only do such conversions if the size of `T1` is no larger than `T2`, and only if the conversions are meaningful.

As a result, we can also achieve the conversions between type `T1` and `T2` by using this pattern.One example is the `math.Float64bits` function, which converts a `float64` values to an `uint64` value. Each corresponding bit of the memory representations of the two values is identical. The `math.Float64bits` function does reverse conversions.

```text
func Float64bits(f float64) uint64 {
	return *(*uint64)(unsafe.Pointer(&f))
}

func Float64frombits(b uint64) float64 {
	return *(*float64)(unsafe.Pointer(&b))
}
```

Please note, the return result of the `Float64bits(aFloat64)` function is different from the result of the explicit conversion `uint64(aFloat64)`.In the following example, we use this pattern to convert a `[]MyString` slice to type `[]string`, and vice versa. The result slice and the original slice share the underlying elements. Such conversions are impossible through safe ways,

```text
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	type MyString string
	ms := []MyString{"C", "C++", "Go"}
	fmt.Printf("%s\n", ms)  // [C C++ Go]
	// ss := ([]string)(ms) // compiling error
	ss := *(*[]string)(unsafe.Pointer(&ms))
	ss[1] = "Rust"
	fmt.Printf("%s\n", ms) // [C Rust Go]
	// ms = []MyString(ss) // compiling error
	ms = *(*[]MyString)(unsafe.Pointer(&ss))
}
```

**Pattern 2: Convert Unsafe Pointer To Uintptr, Then Use The Uintptr Value**

This parttern is not very useful. Usually, we print the result uintptr values to check the memory addresses stored in them. However, there are other less verbose ways to this job. So this pattern is not much useful.Example:

```text
package main

import "fmt"
import "unsafe"

func main() {
	type T struct{a int}
	var t T
	fmt.Println(&t)                                 // &{0}
	fmt.Printf("%x\n", uintptr(unsafe.Pointer(&t))) // c6233120a8
	println(&t)                                     // 0xc6233120a8
}
```

Abviously, it is more simple to use the built-in function `println` to print an address.

**Pattern 3: Convert Unsafe Pointer To Uintptr, Do Arithmetic Operations With The Uintptr Value, Then Convert Back**

For example:

```text
package main

import "fmt"
import "unsafe"

type T struct {
	x bool
	y [3]int16
}

const N = unsafe.Offsetof(T{}.y)
const M = unsafe.Sizeof([3]int16{}[0])

// We use a package-level channel variable to make sure the T
// value used in the main function will be allocated on heap.
var c = make(chan *T, 1)

func main() {
	c <- &T{y: [3]int16{123, 456, 789}}
	t := <-c
	p := unsafe.Pointer(t)
	// "uintptr(p) + N + M + M" stores the address of t.y[2].
	ty2 := (*int16)(unsafe.Pointer(uintptr(p) + N + M + M))
	fmt.Println(*ty2) // 789
}
```

For this specified example, the conversions are not much useful. It is just a demo for education purpose.Please note, for this specified example, the above conversion line from `uintptr` to `unsafe.Pointer`shouldn't be split into two lines, like the following code shows. Please read the comments in the code for the reason.

```text
...
	p := unsafe.Pointer(t)
	// Now the t value becomes unused, its memory may
	// be garbage collected at this time. So the following
	// use of the address of t.y[2] may be invalid!
	addr := uintptr(p) + N + M + M
	ty2 := (*int16)(unsafe.Pointer(addr))
	fmt.Println(*ty2)
}
```

Such bugs are so subtle and hard to detect, which is why the uses of unsafe pointers are dangerous.If we do want to split that conversion line into two lines, we should call the `runtime.KeepAlive` function and pass the unsafe pointer `p` as the argument. after the use of the address of `t.y[2]`. Like this

```text
...
	p := unsafe.Pointer(t)
	addr := uintptr(p) + N + M + M
	ty2 := (*int16)(unsafe.Pointer(addr))
	fmt.Println(*ty2)
	// This following line assures the memory of the value
	// t will not get garbage collected currently for sure.
	runtime.KeepAlive(p)
}
```

Another detail to note is that, it is not recommended to store the end boundry of a memory block in a pointer \(either safe or unsafe one\). Doing this will prevent another memory block which closely follows the former memory block from being garbage collected.

**Pattern 4: Convert Unsafe Pointer To uintptr When Calling syscall.Syscall**

From the explanations for the last pattern, we know that the following function is dangerous.

```text
// Assume this function will not inlined.
func DoSomething(addr uintptr) {
	// read or write values at the passed address ...
}
```

The reason why the above function is dangerous is that the function itself can't guarantee the values at the passed argument address are not garbage collected yet. In fact, there may be some new values which have been allocated at the passed argument address.However, the prototype of the `Syscall` function in the `syscall` standard package is as

```text
func Syscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err Errno)
```

How does this function guarantee that the values at the passed addresses `a1`, `a2` and `a3` are still not garbage collected yet within the function internal? The function can't guarantee this. In fact, compilers will make the guarantee. It is the privilege of calls to `syscall.Syscall` alike functions.We can think that, compilers will append a `runtime.KeepAlive` call for each of `uintptr` arguments, like the folowing code shows:

```text
syscall.Syscall(SYS_READ, uintptr(fd), uintptr(unsafe.Pointer(p)), uintptr(n))
// Compilers will automatically append the following lines for the above line.
runtime.KeepAlive(unsafe.Pointer(uintptr(fd)))
runtime.KeepAlive(unsafe.Pointer(uintptr(unsafe.Pointer(p))))
runtime.KeepAlive(unsafe.Pointer(uintptr(n)))
```

Although the `syscall.Syscall` function has this privilege, there is a requirement to call this function. The conversions from unsafe pointers to `uinptr` must be present within the literal representations of the arguments of the calls to this function.For example, the following call is invalid if the `p` is not guaranteed to be still used after the call.

```text
u := uintptr(unsafe.Pointer(p))
syscall.Syscall(SYS_READ, uintptr(fd), u, uintptr(n))
```

Again, never use this pattern when calling other functions. We should append the `runtime.KeepAlive`calls manually when calling other functions with `uintptr` parameters.

**Pattern 5: Convert The uintptr Result Of reflect.Value.Pointer Or reflect.Value.UnsafeAddr Method Call To Unsafe Pointer**

The methods `Pointer` and `UnsafeAddr` of the `Value` type in the `reflect` standard package both return a result of type `uintptr` instead of `unsafe.Pointer`. This is a deliberate design, which is to avoid converting the results of calls \(to the two methods\) to any safe poiner types without importing the `unsafe`standard package.

The design requires the return result of a call to either of the two methods must be converted to an unsafe pointer immediately after making the call. Otherwise, there will be small time window in which the value allocated at the address stored in the result might lose all references and be garbage collected.For example, the following call is valid.

```text
p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
```

One the other hand, the following call is invalid.

```text
u := reflect.ValueOf(new(int)).Pointer()
p := (*int)(unsafe.Pointer(u))
```

**Pattern 6: Convert A reflect.SliceHeader.Data Or reflect.StringHeader.Data Field To Unsafe Pointer, And The Inverse.**

For the same reason mentioned in the last subsection, the `Data` fields of the struct type `SliceHeader`and `StringHeader` in the `reflect` standard package are declared with type `uintptr` instead of `unsafe.Pointer`.

It is valid to convert a pointer to a string to a `StringHeader` pointer, so that we can manipulate the internal of the string. The same, it is valid to convert a pointer to a slice to a `SliceHeader` pointer, so that we can manipulate the internal of the slice.An example of using `reflect.StringHeader`:

```text
package main

import "fmt"
import "unsafe"
import "reflect"

func main() {
	a := [...]byte{'G', 'o', 'l', 'a', 'n', 'g'}
	s := "Java"
	hdr := (*reflect.StringHeader)(unsafe.Pointer(&s))
	hdr.Data = uintptr(unsafe.Pointer(&a))
	hdr.Len = len(a)
	fmt.Println(s) // Golang
	// Now s and a share the same byte sequence,
	// which makes the string s become mutable.
	a[2], a[3], a[4], a[5] = 'o', 'g', 'l', 'e'
	fmt.Println(s) // Google
}
```

An example of using `reflect.SliceHeader`:

```text
package main

import "fmt"
import "unsafe"
import "reflect"
import "runtime"

func main() {
	bs := []byte("Golang")
	var pa *[2]byte // an array pointer
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	pa = (*[2]byte)(unsafe.Pointer(hdr.Data))
	runtime.KeepAlive(&bs)
	fmt.Printf("%s\n", pa) // &Go
	pa[1] = 'a'
	fmt.Printf("%s\n", bs) // Galang
}
```

The `runtime.KeepAlive` call is essential if the last `Printf` line is absent.In general, we should only get a `StringHeader` pointer value from an actual \(alreay existed\) string, or get a `SliceHeader` pointer value from an actual \(alreay existed\) slice. We shouldn't do the contrary, such as creating a string from a `StringHeader`, or creating a slice from a `SliceHeader`. For example, the following code is invalid.

```text
// Assume p points to a sequence of byte and
// n is the number of bytes in the sequence.
var hdr reflect.StringHeader
hdr.Data = uintptr(unsafe.Pointer(new([5]byte)))
// Now the just allocated byte array has lose all
// references and it can be garbage collected now.
hdr.Len = 5
s := *(*string)(unsafe.Pointer(&hdr))
```

The following is an example which shows how to convert byte slices to strings, by using the unsafe way, and vice versa. Different from the safe conversion from a byte slice to a string, the unsafe way doesn't allocate a new underlying byte sequence for the result string in each conversion.

```text
package main

import "fmt"
import "unsafe"
import "reflect"
import "runtime"

func ByteSlice2String(bs []byte) (str string) {
	sliceHdr := (*reflect.SliceHeader)(unsafe.Pointer(&bs))
	strHdr := (*reflect.StringHeader)(unsafe.Pointer(&str))
	strHdr.Data = sliceHdr.Data
	strHdr.Len = sliceHdr.Len
	// This KeepAlive line is essential to make the 
	// ByteSlice2String function be always valid
	// when it is provided in custom packages.
	runtime.KeepAlive(&bs)
	return
}

func main() {
	bs := []byte{'G', 'o', 'l', 'a', 'n', 'd'}
	s := ByteSlice2String(bs)
	fmt.Println(s) // Goland
	bs[5] = 'g'
	fmt.Println(s) // Golang
}
```

[The docs](https://golang.org/pkg/reflect/#SliceHeader) of the `SliceHeader` and `StringHeader` types in the `reflect` standard package are similar. The docs says the representations of the two struct types may change in a later release. So the above example may become invalid even if the unsafe rules keep unchanged. Fortunately, the current two avaliable Go compilers \(the standard Go compiler and the gccgo compiler\) both recognise the representations of the two types declared in the `reflect` standard package.

It is also possible to convert a string to a byte slice by using the unsafe way. However, we should treat the result slice as an immutable value and never modify its elements.By the way, for the standard Go compiler, currently \(Go 1.11\), there is a more efficient \(and more unsafe\) way to convert a byte slice to a string.

```text
func ByteSlice2String(bs []byte) string {
	return *(*string)(unsafe.Pointer(&bs))
}
```

This is the implementation adopted by the `String` method of the `Builder` type supported since Go 1.10 in the `strings` standard package. It makes use of the first pattern introduced above. It is more efficient than the above one.

#### Final Words

From the above content, we know that, for some scenarios, the unsafe mechanism can help us write more efficient Go code. However, it is very easy to introduce some subtle bugs when using the unsafe mechanism. A program with these bugs may run well for a long time, but suddenly behave abnormally and even crash some time later. Such bugs are very hard to detect and debug.

We should only use the unsafe mechanism when we have to, and we must use it with extreme care. In particular, we should follow the instructions described above.

And again, we should aware that the unsafe mechanism introduced above may change and even become invalid totally in later Go versions, though no evidences this will happen soon. If the unsafe mechanism rules change, the above introduced valid unsafe pointer use patterns may become invalid. So please keep it easy to switch back to the safe implementations for you code depending on the unsafe mechanism.

