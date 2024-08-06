+++
title = 'for range'
date = 2024-04-26T21:08:38+08:00
+++

## 遍历容器

```go
for key, element = range aContainer {
	// 使用key和element ...
}
```

在此语法形式中，`for`和`range`为两个关键字，`key`和`element`称为循环变量。 如果`aContainer`是一个切片或者数组（或者数组指针，见后），则`key`的类型必须为内置类型`int`。

上面所示的`for-range`语法形式中的等号`=`也可以是一个变量短声明符号`:=`。 当短声明符号被使用的时候，`key`和`element`总是**两个新声明的变量**，这时如果`aContainer`是一个切片或者数组（或者数组指针），则`key`的类型被推断为内置类型`int`。

和传统的`for`循环流程控制一样，每个`for-range`循环流程控制形成了两个代码块，其中一个是隐式的，另一个是显式的（花括号`之间`的部分）。 此显式的代码块内嵌在隐式的代码块之中。

和`for`循环流程控制一样，`break`和`continue`也可以使用在一个`for-range`循环流程控制中的显式代码块中。

变种形式：

```go
// 忽略键值循环变量。
for _, element = range aContainer {
	// ...
}

// 忽略元素循环变量。
for key, _ = range aContainer {
	element = aContainer[key]
	// ...
}

// 舍弃元素循环变量。此形式和上一个变种等价。
for key = range aContainer {
	element = aContainer[key]
	// ...
}

// 键值和元素循环变量均被忽略。
for _, _ = range aContainer {
	// 这个变种形式没有太大实用价值。
}

// 键值和元素循环变量均被舍弃。此形式和上一个变种等价。
for range aContainer {
	// 这个变种形式没有太大实用价值。
}
```

**遍历一个nil映射或者nil切片是允许的。这样的遍历可以看作是一个空操作。**

一些关于遍历映射条目的细节：

- **映射中的条目的遍历顺序是不确定的**（可以认为是随机的）。或者说，同一个映射中的条目的两次遍历中，条目的顺序很可能是不一致的，即使在这两次遍历之间，此映射并未发生任何改变。
- 如果在一个映射中的条目的**遍历过程**中，**一个还没有被遍历到的条目被删除了，则此条目保证不会被遍历出来**。
- 如果在一个映射中的条目的**遍历过程**中，一个新的条目被添加入此映射，则此条目**并不保证**将在此遍历过程中被遍历出来。

如果可以确保没有其它协程操纵一个映射`m`，则下面的代码保证将清空`m`中所有条目（除了那些键值为`NaN`的条目）。

```go
for key := range m {
	delete(m, key)
}
```

当然，数组和切片元素也可以用传统的`for`循环来遍历。

```go
for key, element = range aContainer {...}
```

有两个重要的事实存在：

1. 被遍历的容器值是`aContainer`的**一个副本**。 注意，只有`aContainer`的**直接部分被复制**了。 此副本是一个**匿名的值**，所以它是不可被修改的。
    1. 如果`aContainer`是一个**数组**，那么在遍历过程中对此数组元素的修改不会体现到循环变量中。 原因是此数组的副本（被真正遍历的容器）和此数组**不共享任何元素**。
    2. 如果`aContainer`是一个切片（或者映射），那么在遍历过程中对此切片（或者映射）元素的修改将体现到循环变量中。 原因是此切片（或者映射）的副本和此切片（或者映射）**共享元素**（或条目）。
2. 在遍历中的每个循环步，`aContainer`副本中的一个**键值**元素对将被赋值（**复制）给循环变量**。 所以对循环变量的直接部分的修改将不会体现在`aContainer`中的对应元素中。 （因为这个原因，并且`for-range`循环是遍历映射条目的唯一途径，所以最好不要使用大尺寸的映射键值和元素类型，以避免较大的复制负担。）

```go
package main

import "fmt"

func main() {
	type Person struct {
		name string
		age  int
	}
	persons := [2]Person {{"Alice", 28}, {"Bob", 25}}
	for i, p := range persons {
		fmt.Println(i, p)
		// 此修改将不会体现在这个遍历过程中，
		// 因为被遍历的数组是persons的一个副本。
		persons[1].name = "Jack"

		// 此修改不会反映到persons数组中，因为p
		// 是persons数组的副本中的一个元素的副本。
		p.age = 31
	}
	fmt.Println("persons:", &persons)
}
```

输出结果：

```text
0 {Alice 28}
1 {Bob 25}
persons: &[{Alice 28} {Jack 25}]
```

如果我们将上例中的数组改为一个切片，则在循环中对此切片的修改将在循环过程中体现出来。 但是对循环变量的修改仍然不会体现在此切片中。

```go
// 数组改为切片
	persons := []Person {{"Alice", 28}, {"Bob", 25}}
	for i, p := range persons {
		fmt.Println(i, p)
		// 这次，此修改将反映在此次遍历过程中。
		persons[1].name = "Jack"
		// 这个修改仍然不会体现在persons切片容器中。
		p.age = 31
	}
	fmt.Println("persons:", &persons)
}
```

输出结果变成了

```text
0 {Alice 28}
1 {Jack 25}
persons: &[{Alice 28} {Jack 25}]
```

**复制一个切片或者映射的代价很小，但是复制一个大尺寸的数组的代价比较大**。 所以，一般来说，`range`关键字后跟随一个大尺寸数组不是一个好主意。 如果我们要遍历一个大尺寸数组中的元素，我们以遍历从**此数组派生出来的一个切片，或者遍历一个指向此数组的指针**（详见下一节）。

对于一个数组或者切片，如果它的**元素类型的尺寸较大**，则一般来说，用**第二个循环变量来存储每个循环步中被遍历的元素不是一个好主意**。 对于这样的数组或者切片，我们最好忽略或者舍弃`for-range`代码块中的第二个循环变量，或者使用传统的`for`循环来遍历元素。 比如，在下面这个例子中，函数`fa`中的循环效率比函数`fb`中的循环低得多。

```go
type Buffer struct {
	start, end int
	data       [1024]byte
}

func fa(buffers []Buffer) int {
	numUnreads := 0
	for _, buf := range buffers {
		numUnreads += buf.end - buf.start
	}
	return numUnreads
}

func fb(buffers []Buffer) int {
	numUnreads := 0
	for i := range buffers {
		numUnreads += buffers[i].end - buffers[i].start
	}
	return numUnreads
}
```

在Go 1.22之前，对一个如下`for-range`循环代码块（注意`range`前面是`:=`）

```go
for key, element := range aContainer {...}
```

所有被遍历的键值元素对将被赋值给**同一对**循环变量实例。 但是从Go 1.22版本开始，每组键值元素对将被赋值给一对**与众不同**的循环变量实例（即循环变量在每个循环步都会生成一份新的实例）。

下面这个例子展示了Go 1.21-和Go 1.22+之间的行为差异。

```go
package main

import "fmt"

func main() {
   for i, n := range []int{0, 1, 2} {
      defer func() {
         fmt.Println(i, n)
      }()
   }
}
```

输出结果：

```text
// go1.21 及以下
2 3
2 3
2 3
// go1.22
2 3
1 2
0 1
```

下面这个例子更加典型

```go
package main

import "fmt"

func main() {
	var m = map[*int]uint32{}
	for i, n := range []int{1, 2, 3} {
		m[&i]++
		m[&n]++
	}
	fmt.Println(len(m))
}
```

输出结果：

```text
// go1.21 
2
// go1.22
6
```



## 遍历字符串中的码点

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