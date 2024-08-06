+++
title = '类型'
date = 2024-08-06T20:50:18+08:00
+++

# 定义

我们先看看在[《Go 语言编程规范》](https://go.dev/ref/spec#Types)是怎么定义类型的。

> A type determines a set of values together with operations and methods specific to those values. A type may be denoted by a type name, if it has one, which must be followed by type arguments if the type is generic. A type may also be specified using a type literal, which composes a type from existing types.
> 一种类型确定了一组值，以及与这些值特定的操作和方法。如果一个类型有名称，则可以用类型名称表示该类型，并且必须在后面跟上类型参数（如果该类型是泛型）。还可以使用类型字面量来指定一个类型，它由现有的类型组成。
>
> The language [predeclares](https://go.dev/ref/spec#Predeclared_identifiers) certain type names. Others are introduced with [type declarations](https://go.dev/ref/spec#Type_declarations) or [type parameter lists](https://go.dev/ref/spec#Type_parameter_declarations). *Composite types*—array, struct, pointer, function, interface, slice, map, and channel types—may be constructed using type literals.
>
> Go 语言预先声明了某些类型名称。其他类型通过类型声明或类型参数列表引入。复合类型——数组、结构体、指针、函数、接口、切片、映射和通道类型——可以使用类型字面量来构造。
>
> Predeclared types, defined types, and type parameters are called *named types*. An alias denotes a named type if the type given in the alias declaration is a named type.
>
> 预声明类型（内置类型）、定义类型和类型参数被称为命名类型。如果别名声明中给定的类型是一个命名类型，则该别名表示一个命名类型。

其中有一些名词需要我们去理解。

## 具名类型 vs 无名类型（Named vs Unnamed Type ）

**Named Type**:

+ Predeclared types：内置类型

+ Defined types: 定义类型—用 type 关键字定义的类型（定义的泛型类型需要是实例化类型）

+ Type parameters：一个类型参数类型（使用在自定义泛型中）

+ Alias：别名的源类型是具名类型，该别名也是一个具名类型。

> TODO: 补充泛型代码

**Unamed type**：其它类型称为无名类型。一个无名类型肯定是一个组合类型（反之则未必）。

```go
// name 由 type 定义的定义类型，显然是一个 具名类型
type name string

func (n name) f() {

}

// 该别名的 基类型 name 显然是 具名类型，所以该别名类型也是具名类型
type myName = name

// m 方法名如果是 f ，编译器还会报错 Method redeclared 'myName.f'
func (n myName) m() {

}

// 别名源类型为 无名类型，该别名类型为 无名类型
type inventory = map[string]int

// 无法通过编译
// Invalid receiver type 'map[string]int' ('map[string]int' is an unnamed type
//func (i inventory) f()  {
//
//}

// 具名类型，但是是 组合类型
type inventory2 map[string]int

func (i inventory2) f() {

}
```

Named types 可以作为方法的接受者， unnamed type 却不能。

```go
// Map 是一个具名类型，但是 map[string]string 是一个无名类型
type Map map[string]string

// SetVale 可以编译通过
func (m Map) SetVale(key,value string) {
    m[key] = value
}

// 编译失败
// Invalid receiver type 'map[string]string' ('map[string]string' is an unnamed type)
// 无效的接收器类型'map[string]string' ('map[string]string'是未命名的类型)
//func (m map[string]string)  SetVale(key,value string) {
//    m[key] = value
//}
```

## 基本类型

内置基本类型：

- 内置数值类型：
  - `int8`、`uint8`（`byte`）、`int16`、`uint16`、`int32`（`rune`）、`uint32`、`int64`、`uint64`、`int`、`uint`、`uintptr`。
  - `float32`、`float64`。
  - `complex64`、`complex128`。
- 内置布尔类型：`bool`.
- 内置字符串类型：`string`.

注意，`byte`是`uint8`的一个内置别名，`rune`是`int32`的一个内置别名。 

## 组合类型

Go支持下列组合类型：

- [指针类型](https://gfw.go101.org/article/pointer.html) - 类C指针
- [结构体类型](https://gfw.go101.org/article/struct.html) - 类C结构体
- [函数类型](https://gfw.go101.org/article/function.html) - 函数类型在Go中是一种一等公民类别
- [容器类型](https://gfw.go101.org/article/container.html)，包括:
  - 数组类型 - 定长容器类型
  - 切片类型 - 动态长度和容量容器类型
  - 映射类型（map）- 也常称为字典类型。在标准编译器中映射是使用哈希表实现的。
- [通道类型](https://gfw.go101.org/article/channel.html) - 通道用来同步并发的协程
- [接口类型](https://gfw.go101.org/article/interface.html) - 接口在反射和多态中发挥着重要角色

无名组合类型可以用它们各自的字面表示形式来表示。 下面是一些各种不同种类的无名组合类型字面表示形式的例子：

```go
// 假设T为任意一个类型，Tkey为一个支持比较的类型。

*T         // 一个指针类型
[5]T       // 一个元素类型为T、元素个数为5的数组类型
[]T        // 一个元素类型为T的切片类型
map[Tkey]T // 一个键值类型为Tkey、元素类型为T的映射类型

// 一个结构体类型
struct {
    name string
    age  int
}

// 一个函数类型
func(int) (bool, string)

// 一个接口类型
interface {
    Method0(string) int
    Method1() (int, bool)
}

// 几个通道类型
chan T
chan<- T
<-chan T
```

## 类型名和类型字面值（Type Name & Type Literal ）

> Type declarations define new named types by specifying the type specification. They take the form of `type Name UnderlyingType` where `Name` is the new type's name and `UnderlyingType` is the representation of the underlying type.
>
> 类型声明通过指定类型规范来定义新的命名类型。它们采用 type Name UnderlyingType 的形式，其中 Name 是新类型的名称，UnderlyingType 是底层类型的表示。
>
> The underlying type can be a type literal, which is an expression defining the structure and behavior of the new type without naming it.
>
> 底层类型（UnderlyingType）**可以是**类型字面值，这是一种表达式，它定义了**新类型的结构和行为但没有命名**。

**类型的字面值描述了如何构建一个特定的数据结构，而类型名为这个数据结构起了一个可以引用的名字**。在许多编程语言和文档中，“字面值”常常指的是直接出现在代码中的值或者结构，而不是通过变量或者类型名来引用。

所以单独的类型字面值一定是无名类型。

## 底层类型（Underlying Type）

在Go中，每个类型都有一个底层类型。规则：

- 一个内置类型的底层类型为它自己。
- **一个无名类型（必为一个组合类型）的底层类型为它自己**。
- **在一个类型声明中，新声明的类型和源类型共享底层类型**。
- `unsafe`标准库包中定义的`Pointer`类型的底层类型是它自己。 （至少我们可以认为是这样。事实上，关于`unsafe.Pointer`类型的底层类型，官方文档中并没有清晰的说明。我们也可以认为`unsafe.Pointer`类型的底层类型为`*T`，其中`T`表示一个任意类型。） `unsafe.Pointer`也被视为一个内置类型。

```go
// 这四个类型的底层类型均为内置类型int。
type (
    MyInt int
    Age   MyInt
)

// 下面这三个新声明的类型的底层类型各不相同。
type (
    IntSlice   []int   // 底层类型为[]int
    MyIntSlice []MyInt // 底层类型为[]MyInt
    AgeSlice   []Age   // 底层类型为[]Age
)

// 类型[]Age、Ages和AgeSlice的底层类型均为[]Age。
type Ages AgeSlice
```

如何溯源一个声明的类型的底层类型？规则很简单，在溯源过程中，**当遇到一个内置类型或者无名类型时**，溯源结束。 以上面这几个声明的类型为例，下面是它们的底层类型的溯源过程：

![image-20240806205148163](/images/lang/go/types-1.png)

## 类型定义（type definition declaration）

在Go中，我们可以用如下形式来定义新的类型。在此语法中，`type`为一个关键字。

每个类型描述创建了一个**全新的定义类型**（defined type）。

注意：

- 一个新定义的类型和它的源类型为**两个不同的类型**。
- 在两个不同的类型定义中所定义的两个类型肯定是两个不同的类型。
- 一个新定义的类型和它的源类型的底层类型（将在下面介绍）一致并且它们的值可以相互显式转换。
- 类型定义可以出现在函数体内。

一些类型定义的例子：

```go
// 下面这些新定义的类型和它们的源类型都是基本类型。
// 它们的源类型均为预声明类型。
type (
    MyInt int
    Age   int
    Text  string
)

// 下面这些新定义的类型和它们的源类型都是组合类型。
type IntPtr *int
type Book struct{author, title string; pages int}
type Convert func(in0 int, in1 bool)(out0 int, out1 string)
type StringArray [5]string
type StringSlice []string

func f() {
    // 这三个新定义的类型名称只能在此函数内使用。
    type PersonAge map[string]int
    type MessageQueue chan string
    type Reader interface{Read([]byte) int}
}
```

## 类型别名声明

从Go 1.9开始，我们可以使用下面的语法来声明自定义类型别名。此语法和类型定义类似，但是请注意每个类型描述中多了一个等号`=`。

```go
type (
    Name = string
    Age  = int
)

type table = map[string]int
type Table = map[Name]Age
```

类型别名也必须为标识符。同样地，类型别名可以被声明在函数体内。

注意：**尽管一个类型别名有一个名字，但是它可能表示一个无名类型**。 比如，`table`和`Table`这两个别名都表示同一个无名类型`map[string]int`。

```go
// name 由 type 定义的定义类型，显然是一个 具名类型
type name string

func (n name) f() {

}

// 该别名的 基类型 name 显然是 具名类型，所以该别名类型也是具名类型
type myName = name

// m 方法名如果是 f ，编译器还会报错 Method redeclared 'myName.f'
func (n myName) m() {

}

// 别名源类型为 无名类型，该别名类型为 无名类型
type inventory = map[string]int

// 无法通过编译
// Invalid receiver type 'map[string]int' ('map[string]int' is an unnamed type
//func (i inventory) f()  {
//
//}
```

**当你使用 type 声明了一个新类型，它不会继承原有类型的方法集，但是别名会继承原有类型的方法集。** 比如 myName 别名就不能定义方法 f()，否则编译器会提示：Method redeclared 'myName.f'。
