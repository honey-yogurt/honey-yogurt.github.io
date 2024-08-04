+++
title = '反射'
date = 2024-08-03T21:28:02+08:00

+++

## 什么是反射

**反射本质是程序在运行期探知对象的类型信息和内存结构。**

**Go 语言提供了一种机制在运行时更新变量和检查它们的值、调用它们的方法，但是在编译时并不知道这些变量的具体类型，这称为反射机制。**

使用反射的常见场景有以下两种：

1. 不能明确接口调用哪个函数，需要根据传入的参数在运行时决定。
2. 不能明确传入函数的参数类型，需要在运行时处理任意对象。

不推荐使用反射的理由有哪些？

1. 与反射相关的代码，经常是难以阅读的。在软件工程中，代码可读性也是一个非常重要的指标。
2. Go 语言作为一门静态语言，编码过程中，编译器能提前发现一些类型错误，但是对于反射代码是无能为力的。所以包含反射相关的代码，很可能会运行很久，才会出错，这时候经常是直接 panic，可能会造成严重的后果。
3. 反射对性能影响还是比较大的，比正常代码运行速度慢一到两个数量级。所以，对于一个项目中处于运行效率关键位置的代码，尽量避免使用反射特性。 

## Go 是怎么实现反射的

当向接口（interface）变量赋予一个实体类型的时候，接口会存储实体的类型信息，**反射就是通过接口的类型信息实现的，反射建立在类型的基础上**。

即接口变量同时保留变量值和变量类型，Go 的反射就是在运行时操作 interface 中的值和类型的特性。

## reflect.Type类型和值

反射第一定律：**反射可以将 interface 类型变量转换成反射对象**。

```go
func TypeOf(i interface{}) Type 
```

通过调用`reflect.TypeOf`函数，我们可以从一个任何非接口类型的值创建一个`reflect.Type`值，此`reflect.Type`值表示着此非接口值的类型。

看似说的是非接口值创建一个`reflect.Type`，实际上是从一个接口值中提取中一个接口中值的类型信息。因为这个函数的入参是 `any` 类型。任何非接口类型最终都会转为接口类型。

通过此值，我们可以得到很多此非接口类型的信息。 当然，我们也可以将一个接口值传递给一个`reflect.TypeOf`函数调用，但是此调用将返回一个表示着此接口值的动态类型的`reflect.Type`值。 实际上，`reflect.TypeOf`函数的唯一参数的类型为`interface{}`， `reflect.TypeOf`函数将总是返回一个表示着此唯一接口参数值的动态类型的`reflect.Type`值。 那如何得到一个表示着某个接口类型的`reflect.Type`值呢？ 我们必须通过下面将要介绍的一些间接途径来达到这一目的。

从Go 1.22开始，我们也可以调用`reflect.TypeFor`函数来得到一个表示着一个**编译时刻已知**的类型的`reflect.Type`值。 此编译时刻已知的类型可以是一个非接口类型，也可以是一个接口类型。

类型`reflect.Type`为一个接口类型，它指定了[若干方法](https://golang.google.cn/pkg/reflect/#Type)。 通过这些方法，我们能够观察到一个`reflect.Type`值所表示的Go类型的各种信息。 这些方法中的有些适用于[所有种类](https://golang.google.cn/pkg/reflect/#Kind)的类型，有些只适用于一种或几种类型。 通过不合适的`reflect.Type`属主值调用某个方法将在运行时产生一个 panic 。 请阅读`reflect`代码库中各个方法的文档来获取如何正确地使用这些方法。

### 相关数据结构

```go
type Type interface {
	// 此类型的变量对齐后所占用的字节数
	Align() int
	
	// 如果是 struct 的字段，对齐后占用的字节数
	FieldAlign() int

	// 返回类型方法集里的第 `i` (传入的参数)个方法
	Method(int) Method

	// 通过名称获取方法
	MethodByName(string) (Method, bool)

	// 获取类型方法集里导出的方法个数
	NumMethod() int

	// 类型名称
	Name() string

	// 返回类型所在的路径，如：encoding/base64
	PkgPath() string

	// 返回类型的大小，和 unsafe.Sizeof 功能类似
	Size() uintptr

	// 返回类型的字符串表示形式
	String() string

	// 返回类型的类型值
	Kind() Kind

	// 类型是否实现了接口 u
	Implements(u Type) bool

	// 是否可以赋值给 u
	AssignableTo(u Type) bool

	// 是否可以类型转换成 u
	ConvertibleTo(u Type) bool

	// 类型是否可以比较
	Comparable() bool

	// 下面这些函数只有特定类型可以调用
	// 如：Key, Elem 两个方法就只能是 Map 类型才能调用
	
	// 类型所占据的位数
	Bits() int

	// 返回通道的方向，只能是 chan 类型调用
	ChanDir() ChanDir

	// 返回类型是否是可变参数，只能是 func 类型调用
	// 比如 t 是类型 func(x int, y ... float64)
	// 那么 t.IsVariadic() == true
	IsVariadic() bool

	// 返回内部子元素类型，只能由类型 Array, Chan, Map, Ptr, or Slice 调用
	Elem() Type

	// 返回结构体类型的第 i 个字段，只能是结构体类型调用
	// 如果 i 超过了总字段数，就会 panic
	Field(i int) StructField

	// 返回嵌套的结构体的字段
	FieldByIndex(index []int) StructField

	// 通过字段名称获取字段
	FieldByName(name string) (StructField, bool)

	// FieldByNameFunc returns the struct field with a name
	// 返回名称符合 func 函数的字段
	FieldByNameFunc(match func(string) bool) (StructField, bool)

	// 获取函数类型的第 i 个参数的类型
	In(i int) Type

	// 返回 map 的 key 类型，只能由类型 map 调用
	Key() Type

	// 返回 Array 的长度，只能由类型 Array 调用
	Len() int

	// 返回类型字段的数量，只能由类型 Struct 调用
	NumField() int

	// 返回函数类型的输入参数个数
	NumIn() int

	// 返回函数类型的返回值个数
	NumOut() int

	// 返回函数类型的第 i 个值的类型
	Out(i int) Type

    // 返回类型结构体的相同部分
	common() *rtype
	
	// 返回类型结构体的不同部分
	uncommon() *uncommonType
}
```

```go
// Method 结构体用于描述一个类型的方法
type Method struct {
	// 方法名称
	Name string

	// 方法名的限定包路径，仅用于未导出（小写）的方法。对于导出的方法，这个字段是空字符串。PkgPath 和 Name 组合在一起可以唯一地标识一个方法。
	PkgPath string

	Type  Type  // 方法的类型，描述了方法的签名，包括参数和返回值。
	Func  Value // 表示方法本身，可以通过 reflect.Value.Call 来调用这个方法。方法的接收者作为第一个参数传递。
	Index int   // 在其所属类型的方法集中（method set）中的索引。这个索引用于通过 Type.Method 来快速访问特定的方法。
}
```

```go
// 用于描述结构体中的单个字段。它提供了有关结构体字段的详细信息
type StructField struct {
	// 字段的名称
	Name string

	// 字段名的限定包路径，仅用于未导出（小写）字段。对于导出的字段，这个字段是空字符串。PkgPath 和 Name 组合在一起可以唯一地标识一个字段。
	PkgPath string

	Type      Type      // 字段的类型
	Tag       StructTag // 字段的标签字符串（结构体标签）
	Offset    uintptr   // 字段在结构体中的字节偏移量。这用于在内存中定位字段的位置，通常在低级别内存操作或优化中使用。
	Index     []int     // 字段在结构体中的索引序列
	Anonymous bool      // 字段是否是匿名嵌入字段。匿名字段（嵌入字段）在嵌入结构体中直接暴露其字段和方法。
}
```

### 一些示例

```go
package main

import "fmt"
import "reflect"

func main() {
	type A = [16]int16
	var c <-chan map[A][]byte
	tc := reflect.TypeOf(c)
	fmt.Println(tc.Kind())    // chan
	fmt.Println(tc.ChanDir()) // <-chan
	tm := tc.Elem()  // map[A][]byte
    ta, tb := tm.Key(), tm.Elem() // array([16]int16)  slice([]byte)
	fmt.Println(tm.Kind(), ta.Kind(), tb.Kind()) // map array slice
	tx, ty := ta.Elem(), tb.Elem()

	// byte是uint8类型的别名。
	fmt.Println(tx.Kind(), ty.Kind()) // int16 uint8
	fmt.Println(tx.Bits(), ty.Bits()) // 16 8
	fmt.Println(tx.ConvertibleTo(ty)) // true
	fmt.Println(tb.ConvertibleTo(ta)) // false

	// 切片类型和映射类型都是不可比较类型。
	fmt.Println(tb.Comparable()) // false
	fmt.Println(tm.Comparable()) // false
	fmt.Println(ta.Comparable()) // true
	fmt.Println(tc.Comparable()) // true
}
```

目前，Go支持[26种种类的类型](https://golang.google.cn/pkg/reflect/#Kind)。

在上面这个例子中，我们使用方法`Elem`来得到某些类型的元素类型。 实际上，此方法也可以用来**得到一个指针类型的基类型**。一个例子：

```go
package main

import "fmt"
import "reflect"

type T []interface{m()}
func (T) m() {}

func main() {
	tp := reflect.TypeOf(new(interface{}))
	tt := reflect.TypeOf(T{})
	fmt.Println(tp.Kind(), tt.Kind()) // ptr slice

	// 使用间接的方法得到表示两个接口类型的reflect.Type值。
	ti, tim := tp.Elem(), tt.Elem()
	fmt.Println(ti.Kind(), tim.Kind()) // interface interface

	fmt.Println(tt.Implements(tim))  // true
	fmt.Println(tp.Implements(tim))  // false
	fmt.Println(tim.Implements(tim)) // true

	// 所有的类型都实现了任何空接口类型。
	fmt.Println(tp.Implements(ti))  // true
	fmt.Println(tt.Implements(ti))  // true
	fmt.Println(tim.Implements(ti)) // true
	fmt.Println(ti.Implements(ti))  // true
}
```

> 接收者是 `T` 的值类型。接收者直接使用类型 `T` 而没有变量名，这是一种有效的语法，用于表示这个方法属于类型 `T`。这种写法表示方法属于某个类型，但**没有在方法体内使用接收者变量**。

上面这个例子同时也展示了如何**通过间接的途径得到一个表示一个接口类型的`reflect.Type`值**。

我们可以通过反射列出一个类型的所有方法和一个结构体类型的所有（导出和非导出）字段的类型。 我们也可以通过反射列出一个函数类型的各个输入参数和返回结果类型。

```go
package main

import "fmt"
import "reflect"

type F func(string, int) bool
func (f F) m(s string) bool {
	return f(s, 32)
}
func (f F) M() {}

type I interface{m(s string) bool; M()}

func main() {
	var x struct {
		F F
		i I
	}
	tx := reflect.TypeOf(x)
	fmt.Println(tx.Kind())        // struct
	fmt.Println(tx.NumField())    // 2
	fmt.Println(tx.Field(1).Name) // i
	// 包路径（PkgPath）是非导出字段（或者方法）的内在属性。
	fmt.Println(tx.Field(0).PkgPath) // 
	fmt.Println(tx.Field(1).PkgPath) // main

	tf, ti := tx.Field(0).Type, tx.Field(1).Type
	fmt.Println(tf.Kind())               // func
	fmt.Println(tf.IsVariadic())         // false
	fmt.Println(tf.NumIn(), tf.NumOut()) // 2 1
	t0, t1, t2 := tf.In(0), tf.In(1), tf.Out(0)
	// 下一行打印出：string int bool
	fmt.Println(t0.Kind(), t1.Kind(), t2.Kind())

	fmt.Println(tf.NumMethod(), ti.NumMethod()) // 1 2
	fmt.Println(tf.Method(0).Name)              // M
	fmt.Println(ti.Method(1).Name)              // m
	_, ok1 := tf.MethodByName("m")
	_, ok2 := ti.MethodByName("m")
	fmt.Println(ok1, ok2) // false true
}
```

从上面这个例子我们可以看出：

1. 对于非接口类型，`reflect.Type.NumMethod`方法只返回一个类型的**所有导出**的方法（包括通过内嵌得来的隐式方法）的个数，并且 方法`reflect.Type.MethodByName`不能用来获取一个类型的非导出方法； 而对于接口类型，则并无这些限制（Go 1.16之前的文档对这两个方法的描述不准确，并没有体现出这个差异）。 此情形同样存在于下文将要介绍的`reflect.Value`类型上的相应方法。
2. 虽然`reflect.Type.NumField`方法返回一个结构体类型的**所有字段**（包括非导出字段）的数目，但是[不推荐](https://golang.org/pkg/reflect/#pkg-note-BUG)使用方法`reflect.Type.FieldByName`来获取非导出字段。

我们可以[通过反射来检视结构体字段的标签信息](https://golang.org/pkg/reflect/#StructTag)。 结构体字段标签的类型为`reflect.StructTag`，它的方法`Get`和`Lookup`用来检视字段标签中的键值对。 一个例子：

```go
package main

import "fmt"
import "reflect"

type T struct {
	X    int  `max:"99" min:"0" default:"0"`
	Y, Z bool `optional:"yes"`
}

func main() {
	t := reflect.TypeOf(T{})
	x := t.Field(0).Tag
	y := t.Field(1).Tag
	z := t.Field(2).Tag
	fmt.Println(reflect.TypeOf(x)) // reflect.StructTag
	// v的类型为string
	v, present := x.Lookup("max")     
	fmt.Println(len(v), present)      // 2 true
	fmt.Println(x.Get("max"))         // 99
	fmt.Println(x.Lookup("optional")) //  false
	fmt.Println(y.Lookup("optional")) // yes true
	fmt.Println(z.Lookup("optional")) // yes true
}
```

注意：

- 键值对中的**键不能包含空格**（Unicode值为32）、**双引号**（Unicode值为34）和**冒号**（Unicode值为58）。
- 为了形成键值对，所设想的键值对形式中的**冒号的后面不能紧跟着空格字符**。所以
  ``optional: "yes"``不形成键值对。
- 键值对中的值中的空格不会被忽略。所以
  ``json:"author, omitempty“``、
  ``json:" author,omitempty“``以及
  ``json:"author,omitempty“``各不相同。
- 每个字段标签应该呈现为单行才能使它的整个部分都对键值对的形成有贡献。

`reflect`代码包也提供了一些其它函数来动态地创建出来一些无名组合类型。

```go
package main

import "fmt"
import "reflect"

func main() {
	ta := reflect.ArrayOf(5, reflect.TypeOf(123))
	fmt.Println(ta) // [5]int
	tc := reflect.ChanOf(reflect.SendDir, ta)
	fmt.Println(tc) // chan<- [5]int
	tp := reflect.PtrTo(ta)
	fmt.Println(tp) // *[5]int
	ts := reflect.SliceOf(tp)
	fmt.Println(ts) // []*[5]int
	tm := reflect.MapOf(ta, tc)
	fmt.Println(tm) // map[[5]int]chan<- [5]int
	tf := reflect.FuncOf([]reflect.Type{ta},
				[]reflect.Type{tp, tc}, false)
	fmt.Println(tf) // func([5]int) (*[5]int, chan<- [5]int)
	tt := reflect.StructOf([]reflect.StructField{
		{Name: "Age", Type: reflect.TypeOf("abc")},
	})
	fmt.Println(tt)            // struct { Age string }
	fmt.Println(tt.NumField()) // 1
}
```

注意，到目前为止（Go 1.22），我们**无法通过反射动态创建一个接口类型**。这是Go反射目前的一个限制。

另一个限制是使用反射动态创建结构体类型的时候可能会有各种不完美的情况出现。

第三个限制是我们**无法通过反射来声明一个新的类型**。

## reflect.Value类型和值

反射第二定律：**反射可以将反射对象还原成 interface 对象**。

反射第三定律：**反射对象可修改，value 值必须是可设置的**。

```go
func ValueOf(i interface{}) Value
```

我们可以通过调用`reflect.ValueOf`函数，从一个非接口类型的值创建一个`reflect.Value`值。 此`reflect.Value`值代表着此非接口值。 和`reflect.TypeOf`函数类似，`reflect.ValueOf`函数也只有一个`interface{}`类型的参数。 当我们将一个接口值传递给一个`reflect.ValueOf`函数调用时，此调用返回的是代表着此接口值的动态值的一个`reflect.Value`值。 我们必须通过间接的途径获得一个代表一个接口值的`reflect.Value`值。

被一个`reflect.Value`值代表着的值常称为此`reflect.Value`值的底层值（underlying value）。

`reflect.Value`类型有[很多方法](https://golang.google.cn/pkg/reflect/)。 我们可以调用这些方法来观察和操纵一个`reflect.Value`属主值表示的Go值。 这些方法中的有些适用于所有种类类型的值，有些只适用于一种或几种类型的值。 通过不合适的`reflect.Value`属主值调用某个方法将在运行时产生一个 panic 。 请阅读`reflect`代码库中各个方法的文档来获取如何正确地使用这些方法。

### 数据结构

```go
// 用于表示任意 Go 值
type Value struct {
	// 这是一个指向 abi.Type 类型的指针，保存了 Value 所表示值的类型信息。
	// 通过 typ 方法访问 typ_ 可以避免 v 的逃逸分析（即值被从栈上分配到堆上）。
	typ_ *abi.Type

	// 这是一个 unsafe.Pointer，指向值的数据。
	// 如果 flagIndir 被设置，ptr 是指向数据的指针的指针。当 flagIndir 被设置或 typ.pointers() 为 true 时有效。
	ptr unsafe.Pointer

	// 一个无符号整数（uintptr），包含了有关值的元数据。
	// 最低的五个位表示值的 Kind（类型），与 typ.Kind() 对应。
    // 标志位：
    // 		flagStickyRO: 值是通过未导出的非嵌入字段获得的，因此是只读的。
    // 		flagEmbedRO: 值是通过未导出的嵌入字段获得的，因此是只读的。
    //      flagIndir: val 保存了指向数据的指针。
    //		flagAddr: v.CanAddr 为 true，意味着 flagIndir 被设置且 ptr 非 nil。
    //		flagMethod: v 是一个方法值。
    // 如果 ifaceIndir(typ) 返回 true，代码可以假定 flagIndir 被设置。
    // 剩余的 22+ 位表示方法值的方法编号。如果 flag.kind() != Func，代码可以假定 flagMethod 未设置。
	flag
}

type flag uintptr
```



```go
// 设置切片的 len 字段，如果类型不是切片，就会panic
 func (v Value) SetLen(n int)
 
 // 设置切片的 cap 字段
 func (v Value) SetCap(n int)
 
 // 设置字典的 kv
 func (v Value) SetMapIndex(key, val Value)

 // 返回切片、字符串、数组的索引 i 处的值
 func (v Value) Index(i int) Value
 
 // 根据名称获取结构体的内部字段值
 func (v Value) FieldByName(name string) Value
 
 // 用来获取 int 类型的值
func (v Value) Int() int64

// 用来获取结构体字段（成员）数量
func (v Value) NumField() int

// 尝试向通道发送数据（不会阻塞）
func (v Value) TrySend(x reflect.Value) bool

// 通过参数列表 in 调用 v 值所代表的函数（或方法
func (v Value) Call(in []Value) (r []Value) 

// 调用变参长度可变的函数
func (v Value) CallSlice(in []Value) []Value 

// Elem 返回接口 v 包含的值或者指针 v 指向的指针。
// 如果 v 的 Kind 不是 [Interface] 或 [Pointer]，它会崩溃。
// 如果 v 为 nil，则返回零值。
func (v Value) Elem() Value

// ...
```

### 一些示例

一个`reflect.Value`值的`CanSet`方法将返回此`reflect.Value`值代表的Go值是否可以被修改（可以被赋值）。 如果一个Go值可以被修改，则我们可以调用对应的`reflect.Value`值的`Set`方法来修改此Go值。 注意：**`reflect.ValueOf`函数直接返回的`reflect.Value`值都是不可修改的**。

```go
package main

import "fmt"
import "reflect"

func main() {
	n := 123
	p := &n
	vp := reflect.ValueOf(p)
	fmt.Println(vp.CanSet(), vp.CanAddr()) // false false
	vn := vp.Elem() // 取得vp的底层指针值引用的值的代表值
	fmt.Println(vn.CanSet(), vn.CanAddr()) // true true
	vn.Set(reflect.ValueOf(789)) // <=> vn.SetInt(789)
	fmt.Println(n)               // 789
}
```

**一个结构体值的非导出字段不能通过反射来修改**。

```go
package main

import "fmt"
import "reflect"

func main() {
	var s struct {
		X interface{} // 一个导出字段
		y interface{} // 一个非导出字段
	}
	vp := reflect.ValueOf(&s)
	// 如果vp代表着一个指针，下一行等价于"vs := vp.Elem()"。
	vs := reflect.Indirect(vp)
	// vx和vy都各自代表着一个接口值。
	vx, vy := vs.Field(0), vs.Field(1)
	fmt.Println(vx.CanSet(), vx.CanAddr()) // true true
	// vy is addressable but not modifiable.
	fmt.Println(vy.CanSet(), vy.CanAddr()) // false true
	vb := reflect.ValueOf(123)
	vx.Set(vb)     // okay, 因为vx代表的值是可修改的。
	// vy.Set(vb)  // 会造成 panic ，因为vy代表的值是不可修改的。
	fmt.Println(s) // {123 <nil>
	fmt.Println(vx.IsNil(), vy.IsNil()) // false true
}
```

上例中同时也展示了如何间接地获取底层值为接口值的`reflect.Value`值。

从上两例中，我们可以得知有两种方法获取一个代表着一个指针所引用着的值的`reflect.Value`值：

1. 通过调用代表着此指针值的`reflect.Value`值的`Elem`方法。
2. 将代表着此指针值的`reflect.Value`值的传递给一个`reflect.Indirect`函数调用。 （如果传递给一个`reflect.Indirect`函数调用的实参不代表着一个指针值，则此调用返回此实参的一个**复制**。）

注意：`reflect.Value.Elem`方法也可以用来获取一个代表着一个接口值的动态值的`reflect.Value`值，比如下例中所示。

```go
package main

import "fmt"
import "reflect"

func main() {
	var z = 123
	var y = &z
	var x interface{} = y
	v := reflect.ValueOf(&x)
	vx := v.Elem()
	vy := vx.Elem()
	vz := vy.Elem()
	vz.Set(reflect.ValueOf(789))
	fmt.Println(z) // 789
}
```

`reflect`标准库包中也提供了一些对应着内置函数或者各种非反射功能的函数。 下面这个例子展示了如何利用这些函数将一个（效率不高的）自定义泛型函数绑定到不同的类型的函数值上。

```go
package main

import "fmt"
import "reflect"

func InvertSlice(args []reflect.Value) []reflect.Value {
	inSlice, n := args[0], args[0].Len()
	outSlice := reflect.MakeSlice(inSlice.Type(), 0, n)
	for i := n-1; i >= 0; i-- {
		element := inSlice.Index(i)
		outSlice = reflect.Append(outSlice, element)
	}
	return []reflect.Value{outSlice}
}

func Bind(p interface{}, 
		f func ([]reflect.Value) []reflect.Value) {
	// invert代表着一个函数值。
	invert := reflect.ValueOf(p).Elem()
	invert.Set(reflect.MakeFunc(invert.Type(), f))
}

func main() {
	var invertInts func([]int) []int
	Bind(&invertInts, InvertSlice)
	fmt.Println(invertInts([]int{2, 3, 5})) // [5 3 2]

	var invertStrs func([]string) []string
	Bind(&invertStrs, InvertSlice)
	fmt.Println(invertStrs([]string{"Go", "C"})) // [C Go]
}
```

如果一个`reflect.Value`值的底层值为一个函数值，则我们可以调用此`reflect.Value`值的`Call`方法来调用此函数。 每个`Call`方法调用接受一个`[]reflect.Value`类型的参数（表示传递给相应函数调用的各个实参）并返回一个同类型结果（表示相应函数调用返回的各个结果）。

```go
package main

import "fmt"
import "reflect"

type T struct {
	A, b int
}

func (t T) AddSubThenScale(n int) (int, int) {
	return n * (t.A + t.b), n * (t.A - t.b)
}

func main() {
	t := T{5, 2}
	vt := reflect.ValueOf(t)
	vm := vt.MethodByName("AddSubThenScale")
	results := vm.Call([]reflect.Value{reflect.ValueOf(3)})
	fmt.Println(results[0].Int(), results[1].Int()) // 21 9

	neg := func(x int) int {
		return -x
	}
	vf := reflect.ValueOf(neg)
	fmt.Println(vf.Call(results[:1])[0].Int()) // -21
	fmt.Println(vf.Call([]reflect.Value{
		vt.FieldByName("A"), // 如果是字段b，则造成 panic 
	})[0].Int()) // -5
}
```

请注意：**非导出结构体字段值不能用做反射函数调用中的实参**。 如果上例中的`vt.FieldByName("A")`被替换为`vt.FieldByName("b")`，则将产生一个 panic 。

下面是一个使用映射反射值的例子。

```go
package main

import "fmt"
import "reflect"

func main() {
	valueOf := reflect.ValueOf
	m := map[string]int{"Unix": 1973, "Windows": 1985}
	v := valueOf(m)
	// 第二个实参为Value零值时，表示删除一个映射条目。
	v.SetMapIndex(valueOf("Windows"), reflect.Value{})
	v.SetMapIndex(valueOf("Linux"), valueOf(1991))
	for i := v.MapRange(); i.Next(); {
		fmt.Println(i.Key(), "\t:", i.Value())
	}
}
```

注意：方法`reflect.Value.MapRange`方法是从Go 1.12开始才支持的。

下面是一个使用通道反射值的例子。

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan string, 2)
	vc := reflect.ValueOf(c)
	vc.Send(reflect.ValueOf("C"))
	succeeded := vc.TrySend(reflect.ValueOf("Go"))
	fmt.Println(succeeded) // true
	succeeded = vc.TrySend(reflect.ValueOf("C++"))
	fmt.Println(succeeded) // false
	fmt.Println(vc.Len(), vc.Cap()) // 2 2
	vs, succeeded := vc.TryRecv()
	fmt.Println(vs.String(), succeeded) // C true
	vs, sentBeforeClosed := vc.Recv()
	fmt.Println(vs.String(), sentBeforeClosed) // Go true
	vs, succeeded = vc.TryRecv()
	fmt.Println(vs.String()) // <$1>
	fmt.Println(succeeded)   // false
}
```

`reflect.Value`类型的`TrySend`和`TryRecv`方法对应着只有一个`case`分支和一个`default`分支的`select`流程控制代码块。

我们可以使用`reflect.Select`函数在运行时刻来模拟具有不定`case`分支数量的`select`流程控制代码块。

```go
package main

import "fmt"
import "reflect"

func main() {
	c := make(chan int, 1)
	vc := reflect.ValueOf(c)
	succeeded := vc.TrySend(reflect.ValueOf(123))
	fmt.Println(succeeded, vc.Len(), vc.Cap()) // true 1 1

	vSend, vZero := reflect.ValueOf(789), reflect.Value{}
	branches := []reflect.SelectCase{
		{Dir: reflect.SelectDefault, Chan: vZero, Send: vZero},
		{Dir: reflect.SelectRecv, Chan: vc, Send: vZero},
		{Dir: reflect.SelectSend, Chan: vc, Send: vSend},
	}
	selIndex, vRecv, sentBeforeClosed := reflect.Select(branches)
	fmt.Println(selIndex)         // 1
	fmt.Println(sentBeforeClosed) // true
	fmt.Println(vRecv.Int())      // 123
	vc.Close()
	// 再模拟一次select流程控制代码块。因为vc已经关闭了，
	// 所以需将最后一个case分支去除，否则它可能会造成一个 panic 。
	selIndex, _, sentBeforeClosed = reflect.Select(branches[:2])
	fmt.Println(selIndex, sentBeforeClosed) // 1 false
}
```

一些`reflect.Value`值可能表示着不合法的Go值。 这样的值为`reflect.Value`类型的零值（即没有底层值的`reflect.Value`值）。

```go
package main

import "reflect"
import "fmt"

func main() {
	var z reflect.Value // 一个reflect.Value零值
	fmt.Println(z)      // <$1>
	v := reflect.ValueOf((*int)(nil)).Elem()
	fmt.Println(v)      // <$1>
	fmt.Println(v == z) // true
	var i = reflect.ValueOf([]interface{}{nil}).Index(0)
	fmt.Println(i)             // <nil>
	fmt.Println(i.Elem())      // <$1>
	fmt.Println(i.Elem() == z) // true
}
```

从上面的例子中，我们知道，使用空接口`interface{}`值做为中介，一个Go值可以转换为一个`reflect.Value`值。 逆过程类似，通过调用一个`reflect.Value`值的`Interface`方法得到一个`interface{}`值，然后将此`interface{}`断言为原来的Go值。 但是，请注意，调用一个代表着非导出字段的`reflect.Value`值的`Interface`方法将导致一个 panic 。

```go
package main

import (
	"fmt"
	"reflect"
	"time"
)

func main() {
	vx := reflect.ValueOf(123)
	vy := reflect.ValueOf("abc")
	vz := reflect.ValueOf([]bool{false, true})
	vt := reflect.ValueOf(time.Time{})

	x := vx.Interface().(int)
	y := vy.Interface().(string)
	z := vz.Interface().([]bool)
	m := vt.MethodByName("IsZero").Interface().(func() bool)
	fmt.Println(x, y, z, m()) // 123 abc [false true] true

	type T struct {x int}
	t := &T{3}
	v := reflect.ValueOf(t).Elem().Field(0)
	fmt.Println(v)             // 3
	fmt.Println(v.Interface()) // panic
}
```

`Value.IsZero`方法是Go 1.13中引进的，此方法用来查看一个值是否为零值。

从Go 1.17开始，[一个切片可以被转化为一个相同元素类型的数组的指针类型](https://gfw.go101.org/article/container.html#slice-to-array-pointer)。 但是如果在这样的一个转换中数组类型的长度过长，将导致 panic 产生。 因此Go 1.17同时引入了一个`Value.CanConvert(T Type)`方法，用来检查一个转换是否会成功（即不会产生 panic ）。

一个使用了`CanConvert`方法的例子：

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	s := reflect.ValueOf([]int{1, 2, 3, 4, 5})
	ts := s.Type()
	t1 := reflect.TypeOf(&[5]int{})
	t2 := reflect.TypeOf(&[6]int{})
	fmt.Println(ts.ConvertibleTo(t1)) // true
	fmt.Println(ts.ConvertibleTo(t2)) // true
	fmt.Println(s.CanConvert(t1))     // true
	fmt.Println(s.CanConvert(t2))     // false
}
```

## 如何比较两个对象是否完全相同？

```go
func DeepEqual(x, y interface{}) bool
```

`DeepEqual` 函数的参数是两个 `interface`，实际上也就是可以输入任意类型，输出 true 或者 flase 表示输入的两个变量是否是“深度”相等。

先明白一点，如果是不同的类型，即使是底层类型（underlying type）相同，相应的值也相同，那么两者也不是“深度”相等。

```go
type MyInt int
type YourInt int

func main() {
	m := MyInt(1)
	y := YourInt(1)

	fmt.Println(reflect.DeepEqual(m, y)) // false
}
```

在源码里，有对 DeepEqual 函数的非常清楚地注释，列举了不同类型，DeepEqual 的比较情形，这里做一个总结：

| 类型                                  | 深度相等情形                                                 |
| ------------------------------------- | ------------------------------------------------------------ |
| Array                                 | 相同索引处的元素“深度”相等                                   |
| Struct                                | 相应字段，包含导出和不导出，“深度”相等                       |
| Func                                  | 只有两者都是 nil 时                                          |
| Interface                             | 两者存储的具体值“深度”相等                                   |
| Map                                   | 1、都为 nil；2、非空、长度相等，指向同一个 map 实体对象，或者相应的 key 指向的 value “深度”相等 |
| Pointer                               | 1、使用 == 比较的结果相等；2、指向的实体“深度”相等           |
| Slice                                 | 1、都为 nil；2、非空、长度相等，首元素指向同一个底层数组的相同元素，即 &x[0] == &y[0] 或者 相同索引处的元素“深度”相等 |
| numbers, bools, strings, and channels | 使用 == 比较的结果为真                                       |

一般情况下，DeepEqual 的实现只需要递归地调用 == 就可以比较两个变量是否是真的“深度”相等。

但是，有一些异常情况：比如 func 类型是不可比较的类型，只有在两个 func 类型都是 nil 的情况下，才是“深度”相等；float 类型，由于精度的原因，也是不能使用 == 比较的；包含 func 类型或者 float 类型的 struct， interface， array 等。

对于指针而言，当两个值相等的指针就是“深度”相等，因为两者指向的内容是相等的，即使两者指向的是 func 类型或者 float 类型，这种情况下不关心指针所指向的内容。

同样，对于指向相同 slice， map 的两个变量也是“深度”相等的，不关心 slice， map 具体的内容。

对于“有环”的类型，比如循环链表，比较两者是否“深度”相等的过程中，需要对已比较的内容作一个标记，一旦发现两个指针之前比较过，立即停止比较，并判定二者是深度相等的。这样做的原因是，及时停止比较，避免陷入无限循环。

