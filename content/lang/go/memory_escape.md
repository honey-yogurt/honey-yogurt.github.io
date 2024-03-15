+++
title = '内存逃逸'
date = 2024-03-15T15:51:27+08:00
+++
## 什么是内存逃逸？

在一段程序中，每一个函数都会有自己的内存区域存放自己的局部变量、返回地址等，这些内存会由编译器在栈中进行分配，每一个函数都会分配一个栈桢，在函数运行结束后进行销毁，但是**有些变量我们想在函数运行结束后仍然使用它**，那么就需要把这个变量在堆上分配，这种**从"栈"上逃逸到"堆"上的现象就成为内存逃逸**。

## 什么是逃逸分析

`Go`语言引入了`GC`机制，`GC`机制会对位于堆上的对象进行自动管理，当某个对象不可达时(即没有其对象引用它时)，他将会被回收并被重用。虽然引入`GC`可以让开发人员降低对内存管理的心智负担，但是`GC`也会给程序带来性能损耗，当堆内存中有大量待扫描的堆内存对象时，将会给`GC`带来过大的压力，虽然`Go`语言使用的是标记清除算法，并且在此基础上使用了三色标记法和写屏障技术，提高了效率，但是如果我们的程序仍在堆上分配了大量内存，依赖会对`GC`造成不可忽视的压力。因此为了减少`GC`造成的压力，`Go`语言引入了逃逸分析，也就是**想法设法尽量减少在堆上的内存分配，可以在栈中分配的变量尽量留在栈中**。

**逃逸分析就是指程序在编译阶段根据代码中的数据流，对代码中哪些变量需要在栈中分配，哪些变量需要在堆上分配进行静态分析的方法。**堆和栈相比，堆适合不可预知大小的内存分配。但是为此付出的代价是分配速度较慢，而且会形成内存碎片。栈内存分配则会非常快。栈分配内存只需要两个CPU指令：“PUSH”和“RELEASE”，分配和释放；而堆分配内存首先需要去找到一块大小合适的内存块，之后要通过垃圾回收才能释放。所以逃逸分析更做到更好内存分配，提高程序的运行速度。

## `Go`语言中的逃逸分析

粗略看了一下逃逸分析的代码，大概有`1500+`行（go1.15.7）。代码路径：src/cmd/compile/internal/gc/escape.go，大家可以自己看一遍注释，其逃逸分析原理如下：

- `pointers to stack objects cannot be stored in the heap`：**指向栈对象的指针不能存储在堆中**
- `pointers to a stack object cannot outlive that object`：**指向栈对象的指针不能超过该对象的存活期**，也就说指针不能在栈对象被销毁后依旧存活。（例子：声明的函数返回并销毁了对象的栈帧，或者它在循环迭代中被重复用于逻辑上不同的变量）

既然逃逸分析是在编译阶段进行的，那我们就可以通过`go build -gcflags '-m -m -l'`命令查看到逃逸分析的结果，分析内联优化时使用的`-gcflags '-m -m'`，能看到所有的编译器优化，这里使用`-l`禁用掉内联优化，只关注逃逸优化就好了。

## 几个逃逸分析的例子

### 函数返回局部指针变量

```go
func Add(x,y int) *int {
    res := 0
    res = x + y
    return &res
}

func main()  {
    Add(1,2)
}
```

查看逃逸分析结果：

```bash
go build -gcflags="-m -m -l" .\main.go
# command-line-arguments
.\main.go:8:2: res escapes to heap:
.\main.go:8:2:   flow: ~r0 = &res:
.\main.go:8:2:     from &res (address-of) at .\main.go:10:9
.\main.go:8:2:     from return &res (return) at .\main.go:10:2
.\main.go:8:2: moved to heap: res
```

分析结果很明了，函数返回的局部变量是一个指针变量，当函数`Add`执行结束后，对应的栈桢就会被销毁，但是引用已经返回到函数之外，如果我们在外部解引用地址，就会导致程序访问非法内存，所以编译器经过逃逸分析后将其在堆上分配内存。

### interface类型逃逸

```go
func main()  {
    str := "how to earn losts of money"
    fmt.Printf("%v",str)
}
```

查看逃逸分析结果：

```bash
go build -gcflags="-m -m -l" .\main.go
# command-line-arguments
.\main.go:7:19: str escapes to heap:
.\main.go:7:19:   flow: {storage for ... argument} = &{storage for str}:
.\main.go:7:19:     from str (spill) at .\main.go:7:19
.\main.go:7:19:     from ... argument (slice-literal-element) at .\main.go:7:12
.\main.go:7:19:   flow: {heap} = {storage for ... argument}:
.\main.go:7:19:     from ... argument (spill) at .\main.go:7:12
.\main.go:7:19:     from fmt.Printf("%v", ... argument...) (call parameter) at .\main.go:7:12
.\main.go:7:12: ... argument does not escape
.\main.go:7:19: str escapes to heap
```

`str`是`main`函数中的一个局部变量，传递给`fmt.Println()`函数后发生了逃逸，这是因为`fmt.Println()`函数的入参是一个`interface{}`类型，**如果函数参数为`interface{}`，那么在编译期间就很难确定其参数的具体类型，也会发生逃逸**。

观察这个分析结果，我们可以看到没有`moved to heap: str`，这也就是说明`str`变量并没有在堆上进行分配，只是它存储的值逃逸到堆上了，也就说任何被`str`引用的对象必须分配在堆上。如果我们把代码改成这样：

```go
func main()  {
    str := "how to earn losts of money"
    fmt.Printf("%p",&str)
}
```

查看逃逸分析结果：

```go
go build -gcflags="-m -m -l" .\main.go
# command-line-arguments
.\main.go:6:2: str escapes to heap:
.\main.go:6:2:   flow: {storage for ... argument} = &str:
.\main.go:6:2:     from &str (address-of) at .\main.go:7:19
.\main.go:6:2:     from &str (interface-converted) at .\main.go:7:19
.\main.go:6:2:     from ... argument (slice-literal-element) at .\main.go:7:12
.\main.go:6:2:   flow: {heap} = {storage for ... argument}:
.\main.go:6:2:     from ... argument (spill) at .\main.go:7:12
.\main.go:6:2:     from fmt.Printf("%p", ... argument...) (call parameter) at .\main.go:7:12
.\main.go:6:2: moved to heap: str
.\main.go:7:12: ... argument does not escape
```

这回`str`也逃逸到了堆上，在堆上进行内存分配，这是因为我们访问`str`的地址，因为入参是`interface`类型，所以变量`str`的地址以实参的形式传入`fmt.Printf`后被装箱到一个`interface{}`形参变量中，装箱的形参变量的值要在堆上分配，但是还要存储一个栈上的地址，也就是`str`的地址，**堆上的对象不能存储一个栈上的地址**，所以`str`也逃逸到堆上，在堆上分配内存。（**这里注意一个知识点：Go语言的参数传递只有值传递**）

### 闭包产生的逃逸

```go
func Increase() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}

func main() {
    in := Increase()
    fmt.Println(in()) // 1
}
```

查看逃逸分析结果：

```bash
go build -gcflags="-m -m -l" .\main.go
# command-line-arguments
.\main.go:17:2: Increase capturing by ref: n (addr=false assign=true width=8)
.\main.go:18:9: func literal escapes to heap:
.\main.go:18:9:   flow: ~r0 = &{storage for func literal}:
.\main.go:18:9:     from func literal (spill) at .\main.go:18:9
.\main.go:18:9:     from return func literal (return) at .\main.go:18:2
.\main.go:17:2: n escapes to heap:
.\main.go:17:2:   flow: {storage for func literal} = &n:
.\main.go:17:2:     from n (captured by a closure) at .\main.go:19:3
.\main.go:17:2:     from n (reference) at .\main.go:19:3
.\main.go:17:2: moved to heap: n
.\main.go:18:9: func literal escapes to heap
.\main.go:7:16: in() escapes to heap:
.\main.go:7:16:   flow: {storage for ... argument} = &{storage for in()}:
.\main.go:7:16:     from in() (spill) at .\main.go:7:16
.\main.go:7:16:     from ... argument (slice-literal-element) at .\main.go:7:13
.\main.go:7:16:   flow: {heap} = {storage for ... argument}:
.\main.go:7:16:     from ... argument (spill) at .\main.go:7:13
.\main.go:7:16:     from fmt.Println(... argument...) (call parameter) at .\main.go:7:13
.\main.go:7:13: ... argument does not escape
.\main.go:7:16: in() escapes to heap
```

因为函数也是一个指针类型，所以匿名函数当作返回值时也发生了逃逸，在匿名函数中使用外部变量`n`，这个变量`n`会一直存在直到`in`被销毁，所以`n`变量逃逸到了堆上。

### 变量大小不确定及栈空间不足引发逃逸

我们先使用`ulimit -a`查看操作系统的栈空间：

```bash
ulimit -a
-t: cpu time (seconds)              unlimited
-f: file size (blocks)              unlimited
-d: data seg size (kbytes)          unlimited
-s: stack size (kbytes)             8192
-c: core file size (blocks)         0
-v: address space (kbytes)          unlimited
-l: locked-in-memory size (kbytes)  unlimited
-u: processes                       2784
-n: file descriptors                256
```

我的电脑的栈空间大小是`8192`，所以根据这个我们写一个测试用例：

```go
package main

import (
    "math/rand"
)

func LessThan8192()  {
    nums := make([]int, 100) // = 64KB
    for i := 0; i < len(nums); i++ {
        nums[i] = rand.Int()
    }
}


func MoreThan8192(){
    nums := make([]int, 1000000) // = 64KB
    for i := 0; i < len(nums); i++ {
        nums[i] = rand.Int()
    }
}


func NonConstant() {
    number := 10
    s := make([]int, number)
    for i := 0; i < len(s); i++ {
        s[i] = i
    }
}

func main() {
    NonConstant()
    MoreThan8192()
    LessThan8192()
}
```

查看逃逸分析结果：

```bash
go build -gcflags="-m -m -l" main.go 
# command-line-arguments
./main.go:8:14: make([]int, 100) does not escape
./main.go:15:14: make([]int, 1000000) escapes to heap:
./main.go:15:14:   flow: {heap} = &{storage for make([]int, 1000000)}:
./main.go:15:14:     from make([]int, 1000000) (too large for stack) at ./main.go:15:14
./main.go:15:14: make([]int, 1000000) escapes to heap
./main.go:23:11: make([]int, number) escapes to heap:
./main.go:23:11:   flow: {heap} = &{storage for make([]int, number)}:
./main.go:23:11:     from make([]int, number) (non-constant size) at ./main.go:23:11
./main.go:23:11: make([]int, number) escapes to heap
```

我们可以看到，**当栈空间足够时，不会发生逃逸，但是当变量过大时，已经完全超过栈空间的大小时，将会发生逃逸到堆上分配内存**。

**同样当我们初始化切片时，没有直接指定大小，而是填入的变量，这种情况为了保证内存的安全，编译器也会触发逃逸，在堆上进行分配内存**。

## 总结

- 逃逸分析在编译阶段确定哪些变量可以分配在栈中，哪些变量分配在堆上
- 逃逸分析减轻了`GC`压力，提高程序的运行速度
- 栈上内存使用完毕不需要`GC`处理，堆上内存使用完毕会交给`GC`处理
- 函数传参时对于需要修改原对象值，或占用内存比较大的结构体，选择传指针。对于只读的占用内存较小的结构体，直接传值能够获得更好的性能
- 根据代码具体分析，尽量减少逃逸代码，减轻`GC`压力，提高性能