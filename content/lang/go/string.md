+++
title = '字符串'
date = 2024-04-26T21:03:56+08:00
+++

## 字符串类型的内部结构定义

对于标准编译器，字符串类型的内部结构声明如下：

```go
type _string struct {
	elements *byte // 引用着底层的字节
	len      int   // 字符串中的字节数
}
```

从这个声明来看，我们可以将一个字符串的内部定义看作为一个字节序列。 事实上，我们确实可以把**一个字符串看作是一个元素类型为`byte`的（且元素不可修改的）切片**。

从前面的若干文章，我们已经了解到下列关于字符串的一些事实：

- 字符串值（和布尔以及各种数值类型的值）可以被用做常量。
- Go支持两种风格的字符串字面量表示形式：双引号风格（解释型字面表示）和反引号风格（直白字面表示）。
- 字符串类型的零值为空字符串。一个空字符串在字面上可以用`""`或者````来表示。
- 我们可以用运算符`+`和`+=`来衔接字符串。
- **字符串类型都是可比较类型**。同一个字符串类型的值可以用`==`和`!=`比较运算符来比较。 并且和整数/浮点数一样，同一个字符串类型的值也可以用`>`、`<`、`>=`和`<=`比较运算符来比较。 当比较两个字符串值的时候，它们的底层字节将逐一进行比较。如果一个字符串是另一个字符串的前缀，并且另一个字符串较长，则另一个字符串为两者中的较大者。
- **字符串值的内容（即底层字节）是不可更改的。 字符串值的长度也是不可独立被更改的。 一个可寻址的字符串只能通过将另一个字符串赋值给它来整体修改它。**
- 字符串类型没有内置的方法。我们可以使用[`strings`标准库](https://golang.google.cn/pkg/strings/)提供的函数来进行各种字符串操作。
- 调用内置函数`len`来获取一个字符串值的长度（此字符串中存储的字节数）。
- 使用[容器元素索引](https://gfw.go101.org/article/container.html#element-accessment)语法`aString[i]`来获取`aString`中的第`i`个字节。 表达式`aString[i]`是**不可寻址**的。换句话说，`aString[i]`**不可被修改**。
- 使用[子切片语法](https://gfw.go101.org/article/container.html#subslice)`aString[start:end]`来获取`aString`的一个子字符串。 这里，`start`和`end`均为`aString`中存储的字节的下标。
- 对于标准编译器来说，一个字符串的赋值完成之后，此**赋值中的目标值和源值将共享底层字节**。 一个子切片表达式`aString[start:end]`的估值结果也将和基础字符串`aString`共享一部分底层字节。

```go
package main

import (
	"fmt"
	"strings"
)

func main() {
	var helloWorld = "hello world!"

	var hello = helloWorld[:5] // 取子字符串
	// 104是英文字符h的ASCII（和Unicode）码。
	fmt.Println(hello[0])         // 104
	fmt.Printf("%T \n", hello[0]) // uint8

	// hello[0]是不可寻址和不可修改的，所以下面
	// 两行编译不通过。
	/*
	hello[0] = 'H'         // error
	fmt.Println(&hello[0]) // error
	*/

	// 下一条语句将打印出：5 12 true
	fmt.Println(len(hello), len(helloWorld),
			strings.HasPrefix(helloWorld, hello))
}
```

注意：如果在`aString[i]`和`aString[start:end]`中，`aString`和各个下标均为常量，则编译器将在编译时刻验证这些下标的合法性，但是这样的元素访问和子切片表达式的估值结果总是非常量（这是Go语言设计之初的一个失误，但[因为兼容性的原因导致难以弥补](https://github.com/golang/go/issues/28591)）。比如下面这个程序将打引出`4 0`。

```go
package main

import "fmt"

const s = "Go101.org" // len(s) == 9

// len(s)是一个常量表达式，但len(s[:])却不是。
var a byte = 1 << len(s) / 128
var b byte = 1 << len(s[:]) / 128

func main() {
	fmt.Println(a, b) // 4 0
}
```

`a`和`b`两个变量估值不同的具体原因请阅读[移位操作类型推断规则](https://gfw.go101.org/article/operators.html#bitwise-shift-left-operand-type-deduction)和[哪些函数调用在编译时刻被估值](https://gfw.go101.org/article/summaries.html#compile-time-evaluation)。

## 字符串编码和Unicode码点

Unicode标准为全球各种人类语言中的每个字符制定了一个独一无二的值。 但Unicode标准中的基本单位不是字符，而是码点（code point）。大多数的码点实际上就对应着一个字符。 但也有少数一些字符是由多个码点组成的。

码点值在Go中用[rune值](https://gfw.go101.org/article/basic-types-and-value-literals.html#rune)来表示。 内置`rune`类型为内置`int32`类型的一个别名。

在具体应用中，码点值的编码方式有很多，比如UTF-8编码和UTF-16编码等。 目前最流行编码方式为UTF-8编码。在Go中，所有的字符串常量都被视为是UTF-8编码的。 在编译时刻，非法UTF-8编码的字符串常量将导致编译失败。 在运行时刻，Go运行时无法阻止一个字符串是非法UTF-8编码的。

在UTF-8编码中，一个码点值可能由1到4个字节组成。 比如，每个英语码点值（均对应一个英语字符）均由一个字节组成，而每个中文码点值（均对应一个中文字符）均由三个字节组成。

## 字符串相关的类型转换

整数可以被显式转换为字符串类型（但是反之不行）。

这里介绍两种新的字符串相关的类型转换规则：

1. 一个字符串值可以被显式转换为一个字节切片（byte slice），反之亦然。 一个字节切片类型是一个元素类型的底层类型为内置类型`byte`的切片类型。
2. 一个字符串值可以被显式转换为一个码点切片（rune slice），反之亦然。 一个码点切片类型是一个元素类型的底层类型为内置类型`rune`的切片类型。

在一个从码点切片到字符串的转换中，码点切片中的每个码点值将被UTF-8编码为一到四个字节至结果字符串中。 如果一个码点值是一个不合法的Unicode码点值，则它将被视为Unicode替换字符（码点）值`0xFFFD`（Unicode replacement character）。 替换字符值`0xFFFD`将被UTF-8编码为三个字节`0xef 0xbf 0xbd`。

当一个字符串被转换为一个码点切片时，此字符串中存储的字节序列将被解读为一个一个码点的UTF-8编码序列。 非法的UTF-8编码字节序列将被转化为Unicode替换字符值`0xFFFD`。

当一个字符串被转换为一个字节切片时，结果切片中的底层字节序列是此字符串中存储的字节序列的一份深复制。 即Go运行时将为结果切片开辟一块足够大的内存来容纳被复制过来的所有字节。当此字符串的长度较长时，此转换开销是比较大的。 同样，当一个字节切片被转换为一个字符串时，此字节切片中的字节序列也将被深复制到结果字符串中。 当此字节切片的长度较长时，此转换开销同样是比较大的。 在这两种转换中，必须使用深复制的原因是字节切片中的字节元素是可修改的，但是字符串中的字节是不可修改的，所以一个字节切片和一个字符串是不能共享底层字节序列的。

请注意，在字符串和字节切片之间的转换中，

- 非法的UTF-8编码字节序列将被保持原样不变。
- 标准编译器做了一些优化，从而使得这些转换在某些情形下将不用深复制。 这样的情形将在下一节中介绍。

Go并不支持字节切片和码点切片之间的直接转换。我们可以用下面列出的方法来实现这样的转换：

- 利用字符串做为中间过渡。这种方法相对方便但效率较低，因为需要做两次深复制。
- 使用[unicode/utf8](https://golang.google.cn/pkg/unicode/utf8/)标准库包中的函数来实现这些转换。 这种方法效率较高，但使用起来不太方便。
- 使用[`bytes`标准库包中的`Runes`函数](https://golang.google.cn/pkg/bytes/#Runes)来将一个字节切片转换为码点切片。 但此包中没有将码点切片转换为字节切片的函数。

一个展示了上述各种转换的例子：

```go
package main

import (
	"bytes"
	"unicode/utf8"
)

func Runes2Bytes(rs []rune) []byte {
	n := 0
	for _, r := range rs {
		n += utf8.RuneLen(r)
	}
	n, bs := 0, make([]byte, n)
	for _, r := range rs {
		n += utf8.EncodeRune(bs[n:], r)
	}
	return bs
}

func main() {
	s := "颜色感染是一个有趣的游戏。"
	bs := []byte(s) // string -> []byte
	s = string(bs)  // []byte -> string
	rs := []rune(s) // string -> []rune
	s = string(rs)  // []rune -> string
	rs = bytes.Runes(bs) // []byte -> []rune
	bs = Runes2Bytes(rs) // []rune -> []byte
}
```

## 字符串和字节切片之间的转换的编译器优化

上面已经提到了字符串和字节切片之间的转换将深复制它们的底层字节序列。 标准编译器做了一些优化，从而在某些情形下避免了深复制。 至少这些优化在当前（Go官方工具链1.21版本）是存在的。 这样的情形包括：

- 一个`for-range`循环中跟随`range`关键字的从字符串到字节切片的转换；
- 一个在映射元素读取索引语法中被用做键值的从字节切片到字符串的转换（注意：对修改写入索引语法无效）；
- 一个字符串比较表达式中被用做比较值的从字节切片到字符串的转换；
- 一个（至少有一个被衔接的字符串值为非空字符串常量的）字符串衔接表达式中的从字节切片到字符串的转换。

一个例子：

```go
package main

import "fmt"

func main() {
	var str = "world"
	// 这里，转换[]byte(str)将不需要一个深复制。
	for i, b := range []byte(str) {
		fmt.Println(i, ":", b)
	}

	key := []byte{'k', 'e', 'y'}
	m := map[string]string{}
	// 这个string(key)转换仍然需要深复制。
	m[string(key)] = "value"
	// 这里的转换string(key)将不需要一个深复制。
	// 即使key是一个包级变量，此优化仍然有效。
	fmt.Println(m[string(key)]) // value
}
```

注意：在最后一行中，如果在估值`string(key)`的时候有数据竞争的情况，则这行的输出有可能并不是`value`。 但是，无论如何，此行都不会造成恐慌（即使有数据竞争的情况发生）。

```go
package main

import "fmt"
import "testing"

var s string
var x = []byte{1023: 'x'}
var y = []byte{1023: 'y'}

func fc() {
	// 下面的四个转换都不需要深复制。
	if string(x) != string(y) {
		s = (" " + string(x) + string(y))[1:]
	}
}

func fd() {
	// 两个在比较表达式中的转换不需要深复制，
	// 但两个字符串衔接中的转换仍需要深复制。
	// 请注意此字符串衔接和fc中的衔接的差别。
	if string(x) != string(y) {
		s = string(x) + string(y)
	}
}

func main() {
	fmt.Println(testing.AllocsPerRun(1, fc)) // 1
	fmt.Println(testing.AllocsPerRun(1, fd)) // 3
}
```

## 使用`for-range`循环遍历字符串中的码点

`for-range`循环控制中的`range`关键字后可以跟随一个字符串，用来遍历此字符串中的码点（而非字节元素）。 字符串中非法的UTF-8编码字节序列将被解读为Unicode替换码点值`0xFFFD`。

```go\
package main

import "fmt"

func main() {
	s := "éक्षिaπ囧"
	for i, rn := range s {
		fmt.Printf("%2v: 0x%x %v \n", i, rn, string(rn))
	}
	fmt.Println(len(s))
}
```

```go
 0: 0x65 e
 1: 0x301 ́
 3: 0x915 क
 6: 0x94d ्
 9: 0x937 ष
12: 0x93f ि
15: 0x61 a
16: 0x3c0 π
18: 0x56e7 囧
21
```

从此输出结果可以看出：

1. 下标循环变量的值并非连续。原因是下标循环变量为字符串中字节的下标，而一个码点可能需要多个字节进行UTF-8编码。
2. 第一个字符`é`由两个码点（共三字节）组成，其中一个码点需要两个字节进行UTF-8编码。
3. 第二个字符`क्षि`由四个码点（共12字节）组成，每个码点需要三个字节进行UTF-8编码。
4. 英语字符`a`由一个码点组成，此码点只需一个字节进行UTF-8编码。
5. 字符`π`由一个码点组成，此码点只需两个字节进行UTF-8编码。
6. 汉字`囧`由一个码点组成，此码点只需三个字节进行UTF-8编码。

那么如何遍历一个字符串中的字节呢？使用传统`for`循环：

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ囧"
	for i := 0; i < len(s); i++ {
		fmt.Printf("第%v个字节为0x%x\n", i, s[i])
	}
}
```

当然，我们也可以利用前面介绍的编译器优化来使用`for-range`循环遍历一个字符串中的字节元素。 对于官方标准编译器来说，此方法比刚展示的方法效率更高。

```go
package main

import "fmt"

func main() {
	s := "éक्षिaπ囧"
	// 这里，[]byte(s)不需要深复制底层字节。
	for i, b := range []byte(s) {
		fmt.Printf("The byte at index %v: 0x%x \n", i, b)
	}
}
```

从上面几个例子可以看出，`len(s)`将返回字符串`s`中的字节数。 `len(s)`的时间复杂度为`*O*(1)`。 如何得到一个字符串中的码点数呢？使用刚介绍的`for-range`循环来统计一个字符串中的码点数是一种方法，使用`unicode/utf8`标准库包中的[RuneCountInString](https://golang.google.cn/pkg/unicode/utf8/#RuneCountInString)是另一种方法。 这两种方法的效率基本一致。第三种方法为使用`len([]rune(s）)`来获取字符串`s`中码点数。标准编译器从1.11版本开始，对此表达式做了优化以避免一个不必要的深复制，从而使得它的效率和前两种方法一致。 注意，这三种方法的时间复杂度均为`*O*(n)`。

## 更多字符串衔接方法

除了使用`+`运算符来衔接字符串，我们也可以用下面的方法来衔接字符串：

- `fmt`标准库包中的`Sprintf`/`Sprint`/`Sprintln`函数可以用来衔接各种类型的值的字符串表示，当然也包括字符串类型的值。
- 使用`strings`标准库包中的`Join`函数。
- `bytes`标准库包提供的`Buffer`类型可以用来构建一个字节切片，然后我们可以将此字节切片转换为一个字符串。
- 从Go 1.10开始，`strings`标准库包中的`Builder`类型可以用来拼接字符串。 和`bytes.Buffer`类型类似，此类型内部也维护着一个字节切片，但是它在将此字节切片转换为字符串时避免了底层字节的深复制。

标准编译器对使用`+`运算符的字符串衔接做了特别的优化。 所以，一般说来，在被衔接的字符串的数量是已知的情况下，使用`+`运算符进行字符串衔接是比较高效的。

## 语法糖：将字符串当作字节切片使用

内置函数`copy`和`append`可以用来复制和添加切片元素。 事实上，做为一个特例，如果这两个函数的调用中的第一个实参为一个字节切片的话，那么第二个实参可以是一个字符串。 （对于`append`函数调用，字符串实参后必须跟随三个点`...`。） 换句话说，在此特例中，字符串可以当作字节切片来使用。

```go
package main

import "fmt"

func main() {
	hello := []byte("Hello ")
	world := "world!"

	// helloWorld := append(hello, []byte(world)...) // 正常的语法
	helloWorld := append(hello, world...)            // 语法糖
	fmt.Println(string(helloWorld))

	helloWorld2 := make([]byte, len(hello) + len(world))
	copy(helloWorld2, hello)
	// copy(helloWorld2[len(hello):], []byte(world)) // 正常的语法
	copy(helloWorld2[len(hello):], world)            // 语法糖
	fmt.Println(string(helloWorld2))
}
```

## 更多关于字符串的比较

上面已经提到了比较两个字符串事实上逐个比较这两个字符串中的字节。 Go编译器一般会做出如下的优化：

- 对于`==`和`!=`比较，如果这两个字符串的长度不相等，则这两个字符串肯定不相等（无需进行字节比较）。
- 如果这两个字符串底层引用着字符串切片的指针相等，则比较结果等同于比较这两个字符串的长度。

所以两个相等的字符串的比较的时间复杂度取决于它们底层引用着字符串切片的指针是否相等。 如果相等，则对它们的比较的时间复杂度为`*O*(1)`，否则时间复杂度为`*O*(n)`。

上面已经提到了，对于标准编译器，一个字符串赋值完成之后，目标字符串和源字符串将共享同一个底层字节序列。 所以比较这两个字符串的代价很小。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	bs := make([]byte, 1<<26)
	s0 := string(bs)
	s1 := string(bs)
	s2 := s1

	// s0、s1和s2是三个相等的字符串。
	// s0的底层字节序列是bs的一个深复制。
	// s1的底层字节序列也是bs的一个深复制。
	// s0和s1底层字节序列为两个不同的字节序列。
	// s2和s1共享同一个底层字节序列。

	startTime := time.Now()
	_ = s0 == s1
	duration := time.Now().Sub(startTime)
	fmt.Println("duration for (s0 == s1):", duration)

	startTime = time.Now()
	_ = s1 == s2
	duration = time.Now().Sub(startTime)
	fmt.Println("duration for (s1 == s2):", duration)
}
```

```go
duration for (s0 == s1): 10.462075ms
duration for (s1 == s2): 136ns
```

1ms等于1000000ns！所以请尽量避免比较两个很长的不共享底层字节序列的相等的（或者几乎相等的）字符串。