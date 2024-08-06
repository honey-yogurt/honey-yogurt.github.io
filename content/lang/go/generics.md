+++
title = '泛型'
date = 2024-08-06T20:58:28+08:00
+++

使用泛型的根本目的是：**类型安全的参数传递，以及对实现的类型进行抽象**。

**Go 泛型方案的实质是对类型参数（type parameter）的支持**：

+ 泛型函数（generic function）：带有类型参数的函数；
+ 泛型类型（generic type）：带有类型参数的自定义类型；
+ 泛型方法（generic method）：泛型类型的方法。

## 类型参数

类型参数列表看起来像普通的参数列表，只不过它使用方括号（`[]`）而不是圆括号（`()`）。

![类型形参与类型实参](../../../static/images/lang/go/generics-1.pngc/images/lang/go/generics-1.png)

Go 语言规范规定：**函数的类型参数列表位于函数名与函数参数列表之间，由方括号括起的固定个数的、由逗号分隔的类型参数声明组成**，其一般形式如下：

```go
func genericsFunc[T1 constraint1, T2, constraint2, ..., Tn constraintN](ordinary parameters list) (return values list)
```

函数一旦拥有类型参数，就可以用该参数作为常规参数列表和返回值列表中修饰参数和返回值的类型。

按 Go 惯例，**类型参数名的首字母通常采用大写形式，并且类型参数必须是具名的**，即便你在后续的函数参数列表、返回值列表和函数体中没有使用该类型参数。

```go
func print[T any]() { // 正确
}     

func print[any]() {   // 编译错误：all type parameters must be named 
}
```

和常规参数列表中的参数名唯一一样，在同一个类型参数列表中，类型参数名字也要唯一，下面这样的代码将会导致 Go 编译器报错：

```go
func print[T1 any, T1 comparable](sl []T) { //  编译错误：T1 redeclared in this block
    //...
}
```

常规参数列表中的参数有其特定作用域，即从参数声明处开始到函数体结束。和常规参数类似，泛型函数中类型参数也有其作用域范围，这个范围从类型参数列表左侧的方括号[开始，一直持续到函数体结束，如下图所示：

![作用域](../../../static/images/lang/go/generics-2.pngc/images/lang/go/generics-2.png)

类型参数的作用域也决定了类型参数的声明顺序并不重要，也不会影响泛型函数的行为。

```go
// 和上图的泛型函数声明是等价的
func foo[M map[E]T, T any, E comparable](m M)(E, T) {
    //... ...
}
```

## 泛型函数

我们在上一节就是通过泛型函数来解释什么是类型参数。

### 类型形参和类型实参

和普通函数有形式参数与实际参数一样，类型参数也有类型形参（type parameter）和类型实参（type argument）之分。其中类型形参就是泛型函数声明中的类型参数。

```go
// 泛型函数声明：T为类型形参
func maxGenerics[T ordered](sl []T) T

// 调用泛型函数：int为类型实参
m := maxGenerics[int]([]int{1, 2, -4, -6, 7, 0})
```

**在调用泛型函数时，除了要传递普通参数列表对应的实参之外，还要显式传递类型实参**，比如这里的 int。并且，显式传递的类型实参要放在函数名和普通参数列表前的方括号中。

### 实例化

```go
maxGenerics([]int{1, 2, -4, -6, 7, 0})
```

上面代码是对 maxGenerics 泛型函数的一次调用，Go 对这段泛型函数调用代码的处理分为两个阶段，如下图所示：

![泛型调用](../../../static/images/lang/go/generics-3.pngc/images/lang/go/generics-3.png)

Go 首先会对泛型函数进行**实例化**（instantiation），即根据自动推断出的类型实参生成一个新函数（当然这一过程是在**编译阶段**完成的，不会对运行时性能产生影响），然后才会调用这个新函数对输入的函数参数进行处理。

```go
maxGenericsInt := maxGenerics[int] // 实例化后得到的泛型函数实例：maxGenericsInt
fmt.Printf("%T\n", maxGenericsInt) // func([]int) int
maxGenericsInt([]int{1, 2, -4, -6, 7, 0}) // 输出：7
```

当我们使用相同类型实参对泛型函数进行多次调用时，**Go 仅会做一次实例化，并复用实例化后的函数**，比如：

```go
maxGenerics([]int{1, 2, -4, -6, 7, 0})
maxGenerics([]int{11, 12, 14, -36,27, 0}) // 复用第一次调用后生成的原型为func([]int) int的函数
```

类型实例化分两步进行：

1. 首先，编译器在整个泛型函数或类型中将所有类型形参（type parameters）**替换**为它们各自的类型实参（type arguments）。
2. 其次，编译器**验证**每个类型参数是否满足相应的**约束**。

## 泛型类型

泛型类型，就是在**类型声明中带有类型参数的 Go 类型**。

```go
// maxable_slice.go

type maxableSlice[T ordered] struct {
    elems []T
}
```

maxableSlice 是一个自定义切片类型，这个类型的特点是总可以获取其内部元素的最大值，其唯一的要求是其内部元素是可排序的，它**通过带有 ordered 约束的类型参数来明确这一要求**。像这样在定义中带有类型参数的类型就被称为泛型类型（generic type）。

在泛型类型中，类型参数列表放在类型名字后面的方括号中。和泛型函数一样，泛型类型可以有多个类型参数，类型参数名通常是**首字母大写**的，这些类型参数也必须是**具名**的，且命名**唯一**。

```go
type TypeName[T1 constraint1, T2 constraint2, ..., Tn constraintN] TypeLiteral
```

泛型类型中类型参数的作用域范围也是从类型参数列表左侧的方括号[开始，一直持续到类型定义结束的位置。

![泛型类型作用域](../../../static/images/lang/go/generics-4.pngc/images/lang/go/generics-4.png)

这样的作用域将方便我们在各个字段中灵活使用类型参数，下面是一些自定义泛型类型的示例：

```go
type Set[T comparable] map[T]struct{}

type sliceFn[T any] struct {
  s   []T
  cmp func(T, T) bool
}

type Map[K, V any] struct {
  root    *node[K, V]
  compare func(K, K) int
}

type element[T any] struct {
  next *element[T]
  val  T
}

type Numeric interface {
  ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
    ~float32 | ~float64 |
    ~complex64 | ~complex128
}

type NumericAbs[T Numeric] interface {
  Abs() T
}
```

泛型类型中的类型参数可以用来作为类型声明中字段的类型（比如上面的 element 类型）、复合类型的元素类型（比如上面的 Set 和 Map 类型）或方法的参数和返回值类型（如 NumericAbs 接口类型）等。

如果要在**泛型类型声明的内部引用该类型名，必须要带上类型参数**，如上面的 element 结构体中的 next 字段的类型：*element[T]。按照泛型设计方案，如果泛型类型有**不止一个类型参数**，那么在其声明内部引用该类型名时，**不仅要带上所有类型参数，类型参数的顺序也要与声明中类型参数列表中的顺序一致**，比如：

```go
type P[T1, T2 any] struct {
    F *P[T1, T2]  // ok
}
```

不过从实测结果来看，Go 1.19 版本对于下面不符合技术方案的泛型类型声明也并未报错：

```go
type P[T1, T2 any] struct {
    F *P[T2, T1] // 不符合技术方案，但Go 1.19编译器并未报错
}
```

### 实例化

```go
var sl = maxableSlice[int]{
    elems: []int{1, 2, -4, -6, 7, 0},
} 
```

Go 会根据传入的类型实参（int）生成一个新的类型并创建该类型的变量实例，sl 的类型等价于下面代码：

```go
type maxableIntSlice struct {
    elems []int
}
```

泛型类型是否可以像泛型函数那样实现类型实参的自动推断呢？很遗憾，目前的 Go 1.19 尚不支持，下面代码会遭到 Go 编译器的报错：

```go
var sl = maxableSlice {
    elems: []int{1, 2, -4, -6, 7, 0}, // 编译器错误：cannot use generic type maxableSlice[T ordered] without instantiation
} 
```

不过这一特性在 Go 的未来版本中可能会得到支持。

### 泛型类型 vs 非泛型类型

#### 泛型类型与类型别名

类型别名与其绑定的原类型是完全等价的，但这仅限于原类型是一个直接类型，即可直接用于声明变量的类型。那么将类型别名与泛型类型绑定是否可行呢？

```go
type foo[T1 any, T2 comparable] struct {
    a T1
    b T2
}
  
type fooAlias = foo // 编译器错误：cannot use generic type foo[T1 any, T2 comparable] without instantiation
```

泛型类型只是一个生产真实类型的“工厂”，它自身在未实例化之前是不能直接用于声明变量的，因此不符合类型别名机制的要求。泛型类型只有实例化后才能得到一个真实类型，例如下面的代码就是合法的：

```go
type fooAlias = foo[int, string]
```

#### 泛型类型与类型嵌入

引入泛型类型之后，我们依然可以在泛型类型定义中嵌入普通类型，比如下面示例中 Lockable 类型中嵌入的 sync.Mutex：

```go
type Lockable[T any] struct {
    t T
    sync.Mutex
}

func (l *Lockable[T]) Get() T {
    l.Lock()
    defer l.Unlock()
    return l.t
}

func (l *Lockable[T]) Set(v T) {
    l.Lock()
    defer l.Unlock()
    l.t = v
}
```

在泛型类型定义中，我们也可以将其他**泛型类型实例化后的类型作为成员**。现在我们改写一下上面的 Lockable，为其嵌入另外一个泛型类型实例化后的类型 Slice[int]：

```go
type Slice[T any] []T
  
func (s Slice[T]) String() string {
    if len(s) == 0 {
        return ""
    }
    var result = fmt.Sprintf("%v", s[0])
    for _, v := range s[1:] {
        result = fmt.Sprintf("%v, %v", result, v)
    }
    return result
}

type Lockable[T any] struct {
    t T
    Slice[int]
    sync.Mutex
}

func main() {
    n := Lockable[string]{
        t:     "hello",
        Slice: []int{1, 2, 3},
    }
    println(n.String()) // 输出：1, 2, 3
}
```

同理，在普通类型定义中，我们也可以使用实例化后的泛型类型作为成员，比如让上面的 Slice[int]嵌入到一个普通类型 Foo 中，示例代码如下：

```go
type Foo struct {
    Slice[int]
}

func main() {
    f := Foo{
        Slice: []int{1, 2, 3},
    }
    println(f.String()) // 输出：1, 2, 3
}
```

此外，Go 泛型设计方案**支持在泛型类型定义中嵌入类型参数作为成员**，比如下面的泛型类型 Lockable 内嵌了一个类型 T，且 T 恰为其类型参数：

```go
type Lockable[T any] struct {
    T
    sync.Mutex
}
```

不过，Go 1.19 版本编译上述代码时会针对嵌入 T 的那一行报如下错误：

```go
编译器报错：embedded field type cannot be a (pointer to a) type parameter
```

关于这个错误，Go 官方在其 issue 中给出了临时的结论：暂不支持。

#### 泛型方法



### 类型推断

### 函数类型实参的自动推断

如果泛型函数的类型形参较多，那么逐一显式传入类型实参会让泛型函数的调用显得十分冗长。

```go
foo[int, string, uint32, float64](1, "hello", 17, 3.14)
```

Go 团队的泛型实现者们也考虑了这个问题，并给出了解决方法：**函数类型实参的自动推断**（function argument type inference）。

顾名思义，这个机制就是通过判断传递的函数实参的类型来推断出类型实参的类型，从而允许开发者不必显式提供类型实参，下面是以 maxGenerics 函数为例的类型实参推断过程示意图：

![函数类型推断](../../../static/images/lang/go/generics-5.pngc/images/lang/go/generics-5.png)

**函数类型实参类型推断只适用于函数参数中使用的类型参数，而不适用于仅在函数结果中或仅在函数体中使用的类型参数。**

```go
func foo[T comparable, E any](a int, s E) {
}

foo(5, "hello") // 编译器错误：cannot infer T
```

在编译器无法推断出结果时，我们可以给予编译器“部分提示”，比如既然编译器无法推断出 T 的实参类型，那我们就显式告诉编译器 T 的实参类型，即在泛型函数调用时，在类型实参列表中显式传入 T 的实参类型，但 E 的实参类型依然由编译器自动推断。

```go
var s = "hello"
foo[int](5, s)  //ok
foo[int,](5, s) //ok
```

另外，**不能通过返回值类型来推断类型实参。**

```go
func foo[T any](a int) T {
    var zero T
    return zero
}

var a int = foo(5) // 编译器错误：cannot infer T
println(a)
```









------------------------------------   分界线  -------------------------------------------------------------------



泛型为Go语言添加了三个新的重要特性:

1. 函数和类型的类型参数。
2. 将接口类型定义为类型集，包括没有方法的类型。
3. 类型推断，它允许在调用函数时在许多情况下省略类型参数。

## 类型参数

### 类型参数的使用

除了函数中支持使用类型参数列表外，类型也可以使用类型参数列表。

```go
type Slice[T int | string] []T

type Map[K int | string, V float32 | float64] map[K]V

type Tree[T interface{}] struct {
	left, right *Tree[T]
	value       T
}
```

在上述泛型类型中，`T`、`K`、`V`都属于类型形参，类型形参后面是类型约束，类型实参需要满足对应的类型约束。

泛型类型可以有方法，例如为上面的`Tree`实现一个查找元素的`Lookup`方法。

```go
func (t *Tree[T]) Lookup(x T) *Tree[T] { ... }
```

**要使用泛型类型，必须进行实例化**。`Tree[string]`是使用类型实参`string`实例化 `Tree` 的示例。

```go
var stringTree Tree[string]
```

### 类型约束

类型参数列表中每个类型参数都有一个**类型约束**。类型约束定义了一个类型集——只有在这个类型集中的类型才能用作类型实参。

**Go 语言中的类型约束是接口类型。**

类型约束接口可以直接在类型参数列表中使用。

```go
// 类型约束字面量，通常外层interface{}可省略
func min[T interface{ int | float64 }](a, b T) T {
	if a <= b {
		return a
	}
	return b
}
```

作为类型约束使用的接口类型可以事先定义并支持复用。

```go
// 事先定义好的类型约束类型
type Value interface {
	int | float64
}
func min[T Value](a, b T) T {
	if a <= b {
		return a
	}
	return b
}
```

在使用类型约束时，如果省略了外层的`interface{}`会引起歧义，那么就不能省略。例如：

```go
type IntPtrSlice [T *int] []T  // T*int ?

type IntPtrSlice[T *int,] []T  // 只有一个类型约束时可以添加`,`
type IntPtrSlice[T interface{ *int }] []T // 使用interface{}包裹
```

## 类型集

**Go1.18开始接口类型的定义也发生了改变，由过去的接口类型定义方法集（method set）变成了接口类型定义类型集（type set）。**

**也就是说，接口类型现在可以用作值的类型，也可以用作类型约束。**

![type set](../../../static/images/lang/go/generics-6.pngc/images/lang/go/generics-6.png)

把接口类型当做类型集相较于方法集有一个优势: 我们可以显式地向集合添加类型，从而以新的方式控制类型集。

Go语言扩展了接口类型的语法，让我们能够向接口中添加类型。例如

```go
type V interface {
	int | string | bool
}
```

上面的代码就定义了一个包含 `int`、 `string` 和 `bool` 类型的类型集。

![type set](../../../static/images/lang/go/generics-7.pngc/images/lang/go/generics-7.png)

**从 Go 1.18 开始，一个接口不仅可以嵌入其他接口，还可以嵌入任何类型、类型的联合或共享相同底层类型的无限类型集合。**

当用作类型约束时，由接口定义的类型集精确地指定允许作为相应类型参数的类型。

+ ` | ` 符号

`T1 | T2`表示类型约束为T1和T2这两个类型的并集，例如下面的`Integer`类型表示由`Signed`和`Unsigned`组成。

```go
type Integer interface {
	Signed | Unsigned
}
```

+ ` ~ ` 符号

`~T`表示所以底层类型是T的类型，例如`~string`表示所有底层类型是`string`的类型集合。

**注意：**`~`符号后面只能是基本类型

### any 接口

空接口在类型参数列表中很常见，在Go 1.18引入了一个新的预声明标识符，作为**空接口类型的别名**。

```go
// src/builtin/builtin.go

type any = interface{}
```

由此，我们可以使用如下代码：

```go
func foo[S ~[]E, E any]() {
	// ...
}
```

## 类型推断

### 约束类型推断

Go 语言支持另一种类型推断，即*约束类型推断*。

```go
// Scale 返回切片中每个元素都乘c的副本切片
func Scale[E constraints.Integer](s []E, c E) []E {
    r := make([]E, len(s))
    for i, v := range s {
        r[i] = v * c
    }
    return r
}
```

现在假设我们有一个多维坐标的 `Point` 类型，其中每个 `Point` 只是一个给出点坐标的整数列表。这种类型通常会实现一些业务方法，这里假设它有一个`String`方法。

```go
type Point []int32

func (p Point) String() string {
    b, _ := json.Marshal(p)
    return string(b)
}
```

由于一个`Point`其实就是一个整数切片，我们可以使用前面编写的`Scale`函数：

```go
func ScaleAndPrint(p Point) {
    r := Scale(p, 2)
    fmt.Println(r.String()) // 编译失败
}
```

不幸的是，这代码会编译失败，输出`r.String undefined (type []int32 has no field or method String`的错误。

问题是`Scale`函数返回类型为`[]E`的值，其中`E`是参数切片的元素类型。当我们使用`Point`类型的值调用`Scale`（其基础类型为[]int32）时，我们返回的是`[]int32`类型的值，而不是`Point`类型。这源于泛型代码的编写方式，但这不是我们想要的。

为了解决这个问题，我们必须更改 `Scale` 函数，以便为切片类型使用类型参数。

```go
func Scale[S ~[]E, E constraints.Integer](s S, c E) S {
    r := make(S, len(s))
    for i, v := range s {
        r[i] = v * c
    }
    return r
}
```

我们引入了一个新的类型参数`S`，它是切片参数的类型。我们对它进行了约束，使得基础类型是`S`而不是`[]E`，函数返回的结果类型现在是`S`。由于`E`被约束为整数，因此效果与之前相同：第一个参数必须是某个整数类型的切片。对函数体的唯一更改是，现在我们在调用`make`时传递`S`，而不是`[]E`。

现在这个`Scale`函数，不仅支持传入普通整数切片参数，也支持传入`Point`类型参数。

这里需要思考的是，为什么不传递显式类型参数就可以写入 `Scale` 调用？也就是说，为什么我们可以写 `Scale(p, 2)`，没有类型参数，而不是必须写 `Scale[Point, int32](p, 2)` ？

新 `Scale` 函数有两个类型参数——`S` 和 `E`。在不传递任何类型参数的 `Scale(p, 2)` 调用中，如上所述，函数参数类型推断让编译器推断 `S` 的类型参数是 `Point`。但是这个函数也有一个类型参数 `E`，它是乘法因子 `c` 的类型。相应的函数参数是`2`，因为`2`是一个非类型化的常量，函数参数类型推断不能推断出 `E` 的正确类型(最好的情况是它可以推断出`2`的默认类型是 `int`，而这是错误的，因为Point 的基础类型是`[]int32`)。相反，编译器推断 `E` 的类型参数是切片的元素类型的过程称为**约束类型推断**。

**约束类型推断从类型参数约束推导类型参数。当一个类型参数具有根据另一个类型参数定义的约束时使用。当其中一个类型参数的类型参数已知时，约束用于推断另一个类型参数的类型参数。**

### 通常的情况是，当一个约束对某种类型使用 *~type* 形式时，该类型是使用其他类型参数编写的。我们在 `Scale` 的例子中看到了这一点。`S` 是 `~[]E`，后面跟着一个用另一个类型参数写的类型`[]E`。如果我们知道了 `S` 的类型实参，我们就可以推断出`E`的类型实参。`S` 是一个切片类型，而 `E`是该切片的元素类型。
