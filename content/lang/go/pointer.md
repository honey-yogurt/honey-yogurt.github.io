+++
title = '指针'
date = 2024-08-06T20:55:45+08:00
+++

## 内存地址

在编程中，**一个内存地址用来定位一段内存**。

通常地，一个内存地址用一个操作系统原生字（native word）来存储。 一个原生字在32位操作系统上占4个字节，在64位操作系统上占8个字节。 所以，32位操作系统上的理论最大支持内存容量为4GB（1GB == 230字节），64位操作系统上的理论最大支持内存容量为264Byte，即16EB（EB：艾字节，1EB == 1024PB, 1PB == 1024TB, 1TB == 1024GB）。

内存地址的字面形式常用整数的十六进制字面量来表示，比如`0x1234CDEF`。

## 值的地址

一个值的地址是指此值的**直接部分**占据的内存的**起始地址**。在Go中，每个值都包含一个直接部分，但有些值可能还包含一个或多个间接部分，[值部](https://gfw.go101.org/article/value-part.html)将对此详述。

## 什么是指针

指针是Go中的一种类型分类（kind）。 **一个指针可以存储一个内存地址**，通常为另外一个值的地址。这个内存地址可以是任何数据或代码的起始地址，比如，某个变量、某个字段或某个函数。

## 指针类型和值

在Go中，一个无名指针类型的字面形式为`*T`，其中`T`为一个任意类型。类型`T`称为指针类型`*T`的**基类型**（base type）。 如果一个指针类型的基类型为`T`，则我们可以称此指针类型为一个`T`指针类型。

虽然我们可以声明具名指针类型，但是一般不推荐这么做，因为无名指针类型的可读性更高。

如果一个指针类型的底层类型是`*T`，则它的基类型为`T`。

**如果两个无名指针类型的基类型为同一类型，则这两个无名指针类型亦为同一类型。**

一些指针类型的例子：

```go
*int  // 一个基类型为int的无名指针类型。
**int // 一个多级无名指针类型，它的基类型为*int。

type Ptr *int // Ptr是一个具名指针类型，它的基类型为int。
type PP *Ptr  // PP是一个具名多级指针类型，它的基类型为Ptr。
```

指针类型的零值的字面量使用预声明的`nil`来表示。一个nil指针（常称为空指针）中不存储任何地址。

**如果一个指针类型的基类型为`T`，则此指针类型的值只能存储类型为`T`的值的地址。**

## 引用（reference）

术语“引用”暗示着一个关系。比如，如果一个指针中存储着另外一个值的地址，则我们可以说此指针值引用着另外一个值；同时另外一个值当前至少有一个引用。

当一个指针引用着另外一个值，我们也常说此指针指向另外一个值。

## 如何获取一个指针值？

有两种方式来得到一个指针值：

1. 我们可以用内置函数`new`来为任何类型的值**开辟一块内存并将此内存块的起始地址做为此值的地址返回**。 假设`T`是任一类型，则函数调用`new(T)`返回一个类型为`*T`的指针值。 存储在返回指针值所表示的地址处的值（可被看作是一个匿名变量）为`T`的**零值**。
2. 我们也可以使用**前置取地址操作符**`&`来获取一个可寻址的值的地址。 对于一个类型为`T`的可寻址的值`t`，我们可以用`&t`来取得它的地址。`&t`的类型为`*T`。

一般说来，一个可寻址的值是指被放置在内存中某固定位置处的一个值（但放置在某固定位置处的一个值并非一定是可寻址的）。 目前，我们只需知道**所有变量都是可以寻址**的；**但是所有常量、函数返回值和强制转换结果都是不可寻址的**。 当一个变量被声明的时候，Go运行时将为此变量开辟一段内存。此内存的起始地址即为此变量的地址。

## 指针（地址）解引用

我们可以使用**前置解引用操作符**`*`来访问存储在一个指针所表示的地址处的值（即此指针所引用着的值）。 比如，对于**基类型为`T`的指针类型**的一个指针值`p`，我们可以用`*p`来表示地址`p`处的值。 **此值的类型为`T`。**`*p`称为指针`p`的解引用。解引用是取地址的逆过程。

解引用一个nil指针将产生一个panic。

下面这个例子展示了如何取地址和解引用。

```go
package main

import "fmt"

func main() {
    p0 := new(int)   // p0指向一个int类型的零值
    fmt.Println(p0)  // （打印出一个十六进制形式的地址）
    fmt.Println(*p0) // 0

    x := *p0              // x是p0所引用的值的一个复制。
    p1, p2 := &x, &x      // p1和p2中都存储着x的地址。
                          // x、*p1和*p2表示着同一个int值。
    fmt.Println(p1 == p2) // true
    fmt.Println(p0 == p1) // false
    p3 := &*p0            // <=> p3 := &(*p0)
                          // <=> p3 := p0
                          // p3和p0中存储的地址是一样的。
    fmt.Println(p0 == p3) // true
    *p0, *p1 = 123, 789
    fmt.Println(*p2, x, *p3) // 789 789 123

    fmt.Printf("%T, %T \n", *p0, x) // int, int
    fmt.Printf("%T, %T \n", p0, p1) // *int, *int
}
```

下面这张图描绘了上面这个例子中各个值之间的关系：

![image-20240806212942130](../../../static/images/lang/go/pointer-1.pngic/images/lang/go/pointer-1.png)

## Go 指针一些特性

### 在Go中返回一个局部变量的地址是安全的

和C不一样，Go是支持垃圾回收的，所以**一个函数返回其内声明的局部变量的地址是绝对安全的**。比如：

```goag-0-1hhoqq90rag-1-1hhoqq90r
func newInt() *int {
    a := 3
    return &a
}
```

### Go指针不支持算术运算

在Go中，指针是不能参与算术运算的。比如，对于一个指针`p`， 运算`p++`和`p-2`都是非法的。

如果`p`为一个指向一个数值类型值的指针，`*p++`将被编译器认为是合法的并且等价于`(*p)++`。 换句话说，**解引用操作符`*`的优先级都高于自增`++`和自减`--`操作符**。

### 一个指针类型的值不能被随意转换为另一个指针类型

在Go中，只有如下**某个条件**被满足的情况下，一个类型为`T1`的指针值才能被**显式**转换为另一个指针类型`T2`：

1. 类型`T1`和`T2`的**底层类型必须一致（忽略结构体字段的标签）**。 特别地，如果类型`T1`和`T2`中只要有一个是无名类型并且它们的底层类型一致（考虑结构体字段的标签），则此转换可以是**隐式**的。 
2. 类型`T1`和`T2`都为**无名类型**并且它们的**基类型的底层类型**一致（忽略结构体字段的标签）时可以**显式转换**。

比如，

```go
type MyInt int64
type Ta    *int64
type Tb    *MyInt


func transform1() {
    // s 是 *int64 类型，为无名类型， 底层类型是自己，即 *int64，其基类型 int64， 基类型底层类型是 int64
    var s *int64 = new(int64)
    var s1 MyInt = 1
    // s2 是 *MyInt 类型，为无名类型， 底层类型是自己，即 *MyInt。其基类型 MyInt， 基类型底层类型是 int64
    var s2 = &s1
    // 下一行 无法编译通过，Cannot use 's' (type *int64) as the type *MyInt
    // 虽然二者都是无名类型，但是二者的底层类型不同，不满足第一条，肯定不能隐式转换的。
    // 从第一条来看，大前提不满足，是不能转换（显式和隐式），但是第一条和第二条是或的关系，所以如果满足第二条，那还是可以显示转换的
    s2 = s

    // 满足第二条，可以显式转换，二者基类型底层类型相同
    s2 = (*MyInt)(s)
}
```

对于上面所示的这些指针类型，下面的事实成立：

1. 类型`*int64`的值可以被**隐式**转换到类型`Ta`，反之亦然（因为它们的底层类型均为`*int64`）。
2. 类型 `*MyInt`的值可以被**隐式**转换到类型`Tb`，反之亦然（因为它们的底层类型均为`*MyInt`）。
3. 类型`*MyInt`的值可以被**显式**转换为类型`*int64`，反之亦然（因为它们都是无名的并且它们的基类型的底层类型均为`int64`）。
4. 类型`Ta`的值不能直接被转换为类型`Tb`，即使是显式转换也是不行的。 但是，通过上述三条事实，通过**三层显式**转换`Tb((*MyInt)((*int64)(ta)))`，一个类型为`Ta`的值`ta`可以被间接地转换为类型`Tb`。

这些指针类型的任何值都无法被转换到类型`*uint64`。

### 一个指针值不能随意和其它任一指针类型的值进行比较

Go指针值是支持（使用比较运算符`==`和`!=`）比较的。 但是，两个指针只有在下列**任一条件被满足**的时候才可以比较：

1. 这两个指针的类型相同。
2. 其中一个指针可以被**隐式**转换为另一个指针的类型。换句话说，这两个指针的类型的底层类型必须一致并且至少其中一个指针类型为无名的（考虑结构体字段的标签）。
3. 其中一个并且只有一个指针用类型不确定的`nil`标识符表示。

例子：

```go
package main

func main() {
    type MyInt int64
    type Ta    *int64
    type Tb    *MyInt

    // 4个不同类型的指针：
    var pa0 Ta
    var pa1 *int64
    var pb0 Tb
    var pb1 *MyInt

    // 下面这6行编译没问题。它们的比较结果都为true。
    _ = pa0 == pa1
    _ = pb0 == pb1
    _ = pa0 == nil
    _ = pa1 == nil
    _ = pb0 == nil
    _ = pb1 == nil

    // 下面这三行编译不通过。
    /*
    _ = pa0 == pb0
    _ = pa1 == pb1
    _ = pa0 == Tb(nil)
    */
}
```

### 一个指针值不能随意被赋值给其它任意类型的指针值

一个指针值可以被赋值给另一个指针值的条件和这两个指针值可以比较的条件是一致的。

### 上述Go指针的限制是可以被打破的

[`unsafe`标准库包](https://gfw.go101.org/article/unsafe.html)中提供的非类型安全指针（`unsafe.Pointer`）机制可以被用来打破上述Go指针的安全限制。 `unsafe.Pointer`类型类似于C语言中的`void*`。 但是，通常地，非类型安全指针机制不推荐在Go日常编程中使用。