+++
title = '栈'
date = 2024-06-28T15:29:06+08:00
+++

栈是一种“操作受限”的线性表，**只允许在一端插入和删除数据，先进后出**。

栈既可以用数组来实现，也可以用链表来实现。用数组实现的栈，我们叫作顺序栈，用链表实现的栈，我们叫作链式栈。

```go
// 本质上是一个反复写的slice，通过length控制栈顶
type Stack struct {
	Items    []string
	Length   int
	Capacity int
}

func NewStack(capacity int) *Stack {
	return &Stack{
		Items:    make([]string, capacity, capacity),
		Length:   0,
		Capacity: capacity,
	}
}

// 入栈
func (s *Stack) Push(item string) bool {
	if s.Length >= s.Capacity {
		return false
	}
	s.Length++
	s.Items[s.Length-1] = item
	return true
}

// 出栈
func (s *Stack) Pop() string {
	if s.Length == 0 {
		return ""
	}
	item := s.Items[s.Length-1]
	s.Length--
	return item
}
```


## 栈在函数调用中的应用
操作系统给每个线程分配了一块独立的内存空间，这块内存被组织成“栈”这种结构,用来存储函数调用时的临时变量。每进入一个函数，就会将临时变量作为一个“栈帧”入栈，当被调用函数执行完成，返回之后，将这个函数对应的栈帧出栈。

```c
int main() {
    int a = 1;
    int ret = 0;
    int res = 0;
    ret = add(3, 5);
    res = a + ret;
    printf("%d", res);
    reuturn 0;
}
int add(int x, int y) {
    int sum = 0;
    sum = x + y;
    return sum;
}
```
在执行到add()函数时，函数调用栈的情况:

![img.png](/images/algorithm/algo-stack-1.png)

## 栈在表达式求值中的应用
将算术表达式简化为只包含加减乘除四则运算，比如：34+13*9+44-12/3。

编译器就是通过两个栈来实现的。其中一个**保存操作数的栈**，另一个是**保存运算符的栈**。我们从左向右遍历表达式，当遇到数字，我们就直接压入操作数栈；当遇到运算符，就与运算符栈的栈顶元素进行比较。

如果比运算符栈顶元素的优先级高，就将当前运算符压入栈；如果比运算符栈顶元素的优先级低或者相同，从运算符栈中取栈顶运算符，从操作数栈的栈顶取2个操作数，然后进行计算，再把计算完的结果压入操作数栈，继续比较。

![img.png](/images/algorithm/algo-stack-2.png)

## 栈在括号匹配中的应用
我们假设表达式中只包含三种括号，圆括号()、方括号[]和花括号{}，并且它们可以任意嵌套。比如，{[{}]}或[{()}([])]等都为合法格式，而{[}()]或[({)]为不合法的格式。那我现在给你一个包含三种括号的表达式字符串，如何检查它是否合法呢？

我们用栈来保存未匹配的左括号，从左到右依次扫描字符串。当扫描到左括号时，则将其压入栈中；当扫描到右括号时，从栈顶取出一个左括号。如果能够匹配，则继续扫描剩下的字符串。如果扫描的过程中，遇到不能配对的右括号，或者栈中没有数据，则说明为非法格式。

当所有的括号都扫描完成之后，如果栈为空，则说明字符串为合法格式；否则，说明有未匹配的左括号，为非法格式。
















