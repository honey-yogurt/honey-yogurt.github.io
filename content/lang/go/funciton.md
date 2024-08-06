+++
title = '函数'
date = 2024-08-06T20:57:07+08:00
+++

## 函数

在 Go 语言中，函数可是一等的（first-class）公民，函数类型也是一等的数据类型。

这意味着函数不但可以用于封装代码、分割功能、解耦逻辑，还可以化身为**普通的值**，在其他函数间传递、赋予变量、做类型判断和转换等等，就像切片和字典的值那样。

而更深层次的含义就是：函数值可以由此成为能够被**随意传播的独立逻辑组件**（或者说功能模块）。

> 函数的签名其实就是函数的**参数列表和结果列表**的统称，它定义了可用来鉴别不同函数的那些特征，同时也定义了我们与函数交互的方式。

注意，各个参数和结果的**名称**不能算作函数签名的一部分，甚至对于结果声明来说，没有名称都可以。

> TODO：确认有没有名称的两个函数 type 是不是一个type

只要两个函数的参数列表和结果列表中的**元素顺序及其类型**是一致的，我们就可以说它们是一样的函数，或者说是实现了同一个函数类型的函数。

严格来说，函数的名称也不能算作函数签名的一部分，它只是我们在调用函数时，需要给定的标识符而已。

关于函数值和方法值的案例移步 function & method 一节。


## 函数值

**当我们声明了一个函数的时候，我们实际上同时声明了一个不可修改的函数值**。

此函数值用此函数的名称来标识。此函数值的类型的字面表示形式为此函数的原型刨去函数名部分。

注意：内置函数和`init`函数不可被用做函数值。

**函数类型属于引用类型，它的值可以为nil，而这种类型的零值恰恰就是nil。**

函数类型属于不可比较类型。 但是，和映射值以及切片值类似，一个函数值可以和类型不确定的`nil`比较。

调用一个**nil函数来开启一个协程**将产生一个致命的**不可恢复的错误**，此错误将使整个程序崩溃。 在其它情况下调用一个nil函数将产生一个**可恢复的恐慌**。

**当一个函数值被赋给另一个函数值后**，这两个函数值将**共享底层部分**（**内部的函数结构**）。 换句话说，这两个函数值表示的函数可以看作是**同一个函数**。调用它们的效果是相同的。

在实践中，我们常常将一个匿名函数赋值给一个函数类型的变量，从而可以在以后多次调用此匿名函数。

```go
func main() {
	isMultipleOfX := func (x int) func(int) bool {
		return func(n int) bool {
			return n%x == 0
		}
	}

	var isMultipleOf3 = isMultipleOfX(3)
	var isMultipleOf5 = isMultipleOfX(5)
	fmt.Println(isMultipleOf3(6))  // true
	fmt.Println(isMultipleOf3(8))  // false
	fmt.Println(isMultipleOf5(10)) // true
	fmt.Println(isMultipleOf5(12)) // false

	isMultipleOf15 := func(n int) bool {
		return isMultipleOf3(n) && isMultipleOf5(n)
	}
	fmt.Println(isMultipleOf15(32)) // false
	fmt.Println(isMultipleOf15(60)) // true
}
```

## 参数传递-值复制

```go
package main

import "fmt"

func main() {
  array1 := [3]string{"a", "b", "c"}
  fmt.Printf("The array: %v\n", array1)
  array2 := modifyArray(array1)
  fmt.Printf("The modified array: %v\n", array2)
  fmt.Printf("The original array: %v\n", array1)
}

func modifyArray(a [3]string) [3]string {
  a[1] = "x"
  return a
}
```

**所有传给函数的参数值都会被复制，函数在其内部使用的并不是参数值的原值，而是它的副本。**

由于数组是值类型，所以每一次复制都会拷贝它，以及它的所有元素值。我在modify函数中修改的只是原数组的副本而已，并不会对原数组造成任何影响。

注意，对于引用类型，比如：切片、字典、通道，像上面那样复制它们的值，**只会拷贝它们本身而已，并不会拷贝它们引用的底层数据。也就是说，这时只是浅表复制，而不是深层复制**。

以切片值为例，如此复制的时候，只是拷贝了它指向底层数组中某一个元素的指针，以及它的长度值和容量值，而它的底层数组并不会被拷贝。

```go
complexArray1 := [3][]string{
  []string{"d", "e", "f"},
  []string{"g", "h", "i"},
  []string{"j", "k", "l"},
}
```

虽然complexArray1本身是一个数组，但是其中的元素却都是切片。如果对complexArray1中的元素进行增减，那么原值就不会受到影响。但若要修改它已有的元素值，那么原值也会跟着改变。

**函数真正拿到的参数值其实只是它们的副本，函数返回给调用方的结果值也会被复制。**不过，在一般情况下，我们不用太在意。但如果函数在返回结果值之后依然保持执行并会对结果值进行修改，那么我们就需要注意了。

比如在 Go 语言中的 goroutine。在这种情况下，可以有一种场景，即函数返回一个指向某个值的指针或者是引用类型（如切片，映射或通道），然后在另一个 goroutine 中修改这个值。这种情况下，即使函数已经返回，但在另一个 goroutine 中对这个值的修改仍然会影响到函数返回的结果。

```go
func createSlice() []int {
	slice := make([]int, 5)
	go func() {
		for i := range slice {
			slice[i] = i
			time.Sleep(1 * time.Second)
		}
	}()
	return slice
}

func main() {
	slice := createSlice()
	time.Sleep(3 * time.Second)
	fmt.Println(slice) // 输出: [0 1 2 0 0]
	time.Sleep(3 * time.Second)
	fmt.Println(slice) // 输出: [0 1 2 3 4]
}
```

## 闭包（closure）

**在一个函数中存在对外来标识符的引用。所谓的外来标识符，既不代表当前函数的任何参数或结果，也不是函数内部声明的，它是直接从外边拿过来的。**

还有个专门的术语称呼它，叫**自由变量**，可见它代表的肯定是个变量。实际上，如果它是个常量，那也就形成不了闭包了，因为常量是不可变的程序实体，而闭包体现的却是由“不确定”变为“确定”的一个过程。

我们说的这个函数（以下简称闭包函数）就是因为引用了自由变量，而呈现出了一种“不确定”的状态，也叫“开放”状态。

**它的内部逻辑并不是完整的，有一部分逻辑需要这个自由变量参与完成，而后者到底代表了什么在闭包函数被定义的时候却是未知的。**

即使对于像 Go 语言这种静态类型的编程语言而言，我们在定义闭包函数的时候**最多也只能知道自由变量的类型**。

```go
type operate func(x, y int) int

type calculateFunc func(x int, y int) (int, error)

func genCalculator(op operate) calculateFunc {
	return func(x int, y int) (int, error) {
		if op == nil {
			return 0, errors.New("invalid operation")
		}
		return op(x, y), nil
	}
}
```

genCalculator函数只做了一件事，那就是定义一个匿名的、calculateFunc类型的函数并把它作为结果值返回。

而这个匿名的函数就是一个闭包函数。它里面使用的变量 op 既不代表它的任何参数或结果也不是它自己声明的，而是定义它的 genCalculator 函数的参数，所以是一个自由变量。

这个自由变量究竟代表了什么，这一点并不是在定义这个闭包函数的时候确定的，而是在genCalculator函数**被调用的时候确定的**。只有给定了该函数的参数op，我们才能知道它返回给我们的闭包函数可以用于什么运算。

![func-1](/images/lang/go/func-1.png)

那么，实现闭包的意义又在哪里呢？表面上看，我们只是延迟实现了一部分程序逻辑或功能而已，但实际上，我们是在**动态地生成那部分程序逻辑**。

## 变长参数和变长参数函数类型

一个函数**仅最后一个参数可以是一个变长参数**。一个函数可以最多有一个变长参数。一个变长参数的类型总为**一个切片类型**。 变长参数在声明的时候必须在它的（切片）类型的元素类型前面**前置三个点`...`**，以示这是一个变长参数。

```go
func (values ...int64) (sum int64)
func (sep string, tokens ...string) string
```

一个变长函数类型和一个非变长函数类型绝对不可能是同一个类型。

```go
// Sum返回所有输入实参的和。
func Sum(values ...int64) (sum int64) {
	// values的类型为[]int64。
	sum = 0
	for _, v := range values {
		sum += v
	}
	return
}
```

从上面的两个变长参数函数声明可以看出，如果一个变长参数的类型部分为`...T`，则此变长参数的类型实际为`[]T`。

在变长参数函数调用中，可以使用两种风格的方式将实参传递给类型为`[]T`的变长形参：

1. 传递一个切片做为实参。此切片必须可以被赋值给类型为`[]T`的值（或者说此切片可以被隐式转换为类型`[]T`）。 此实参切片后必须跟随三个点`...`。
2. 传递零个或者多个可以被隐式转换为`T`的实参（或者说这些实参可以赋值给类型为`T`的值）。 这些实参将被添加入一个匿名的在运行时刻创建的类型为`[]T`的切片中，然后此切片将被传递给此函数调用。

注意，这两种风格的方式不可在同一个变长参数函数调用中混用。

```go
func Concat(sep string, tokens ...string) (r string) {
	for i, t := range tokens {
		if i != 0 {
			r += sep
		}
		r += t
	}
	return
}

func main() {
	tokens := []string{"Go", "C", "Rust"}
	langsA := Concat(",", tokens...)        // 风格1
	langsB := Concat(",", "Go", "C","Rust") // 风格2
	fmt.Println(langsA == langsB)           // true
}
```

## 高阶函数

简单地说，高阶函数可以满足下面的两个条件：

+ 接受其他的函数作为参数传入
+ 把其他的函数作为结果返回

只要满足了其中任意一个特点，我们就可以说这个函数是一个高阶函数。高阶函数也是函数式编程中的重要概念和特征。



## 一些细节

### 所有的函数调用的传参均属于值复制

**和赋值一样，传参也属于值（浅）复制**。当一个值被复制时，只有它的**直接部分**被复制了。

### 有返回值的函数的调用是一种表达式

**一个有且只有一个返回值的函数的每个调用总可以被当成一个单值表达式使用**。 比如，它可以被内嵌在其它函数调用中当作实参使用，或者可以被当作其它表达式中的操作数使用。

> TODO:表达式和语句的区别？

如果一个有多个返回结果的函数的调用的返回结果没有被舍弃，则此调用可以当作**一个多值表达式**使用在两种场合：

1. 此调用可以在一个**赋值语句**中当作源值来使用，但是它**不能和其它源值掺和**到一块。
2. 此调用可以内嵌在另一个**函数调用**中当作实参来使用，但是它**不能和其它实参掺和**到一块。

```go
func HalfAndNegative(n int) (int, int) {
	return n/2, -n
}

func AddSub(a, b int) (int, int) {
	return a+b, a-b
}

func Dummy(values ...int) {}

func main() {
	// 这几行编译没问题。
	AddSub(HalfAndNegative(6))  // 方式2
	AddSub(AddSub(AddSub(7, 5)))
	AddSub(AddSub(HalfAndNegative(6)))
	Dummy(HalfAndNegative(6))
	_, _ = AddSub(7, 5)   // 方式1

	// 下面这几行编译不通过。
	/*
	_, _, _ = 6, AddSub(7, 5)
	Dummy(AddSub(7, 5), 9)
	Dummy(AddSub(7, 5), HalfAndNegative(6))
	*/
}
```

注意，在目前的标准编译器的实现中，[有几个内置函数破坏了上述规则的普遍性](https://gfw.go101.org/article/exceptions.html#nest-function-calls)。

### 同一个包中可以同名的函数

一般来说，同一个包中声明的函数的名称不能重复，但有两个例外：

1. 同一个包内可以声明若干个原型为`func ()`的名称为`init`的函数。
2. 多个函数的名称可以被声明为空标识符`_`。这样声明的函数不可被调用。

### 自定义函数的调用返回结果可以被舍弃，但是某些内置函数的调用返回结果不可被舍弃

自定义函数的调用结果都是可以被舍弃掉的。 但是大多数内置函数（除了`recover`和`copy`）的调用结果都是不可被舍弃的。 **调用结果不可被舍弃的函数是不可以被用做延迟调用函数和协程起始函数的**，比如`append`函数。

> TODO：协程起始函数

### 某些函数调用是在编译时刻被估值的

大多数函数调用都是在运行时刻被估值的。 但`unsafe`标准库包中的函数的调用都是在编译时刻估值的。 另外，某些其它内置函数（比如`len`和`cap`等）的调用在所传实参满足一定的条件的时候也将在编译时刻估值。 详见[在编译时刻估值的函数调用](https://gfw.go101.org/article/summaries.html#compile-time-evaluation)。

### 不含函数体的函数声明

我们可以使用[Go汇编（Go assembly）](https://golang.google.cn/doc/asm)来实现一个Go函数。 Go汇编代码放在后缀为`.a`的文件中。 一个使用Go汇编实现的函数依旧必须在一个`*.go`文件中声明，但是它的声明必须不能含有函数体。 换句话说，一个使用Go汇编实现的函数的声明中只含有它的原型。

### 某些有返回值的函数可以不必返回

如果一个函数有返回值，则它的函数体内的最后一条语句必须为一条[终止语句](https://golang.google.cn/ref/spec#Terminating_statements)。 Go中有多种终止语句，`return`语句只是其中一种。所以一个有返回值的函数的体内不一定需要一个`return`语句。 比如下面两个函数（它们均可编译通过）：

```go
func fa() int {
	a:
	goto a
}

func fb() bool {
	for{}
}
```

