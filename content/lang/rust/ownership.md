+++
title = '所有权和借用'
date = 2024-04-25T22:10:30+08:00
+++

一般来说，对于固定尺寸类型，会默认放在栈上；而非固定尺寸类型，会默认创建在堆上，成为堆上的一个资源，然后在栈上用一个局部变量来指向它。

```rust
fn main() {
    let a = 10u32;
    let b = a;
    println!("{a}");
    println!("{b}");
}
```
运行结果：
```text
10
10
```

```rust
fn main() {
    let s1 = String::from("hello, yogurt.");
    let s2 = s1;
    println!("{s1}");
    println!("{s2}");
}
```
运行结果：
```text
error[E0382]: borrow of moved value: `s1` 
//            借用了移动后的值 `s1`
 --> src\main.rs:4:15
  |
2 |     let s1 = String::from("hello, yogurt.");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
//             移动发生了，因为 `s1` 的类型是 `String`，而这种类型并没有实现 `Copy` trait."。
3 |     let s2 = s1;
  |              -- value moved here
//                  在这里值移动了
4 |     println!("{s1}");
  |               ^^^^ value borrowed here after move
//                     值在被移动后在这里被借用
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
//    如果性能成本可以接受的话，考虑克隆这个值
  |
3 |     let s2 = s1.clone();
  |                ++++++++
```

为什么看似相同的代码行为，却有着截然不同的结果呢？这就涉及到了 Rust 的所有权。

所谓 所有权 是指**一个变量“拥有”了一块内存区域**。这个内存区域可以是在堆上，可以在栈上，也可以在代码段，还有些内存地址是直接用于 I/O 地址映射的。这些都是内存区域可能存在的位置。

在高级语言中，这个内存位置要在程序中要能被访问，必然就会与一个或多个变量建立关联关系（低级语言如汇编语言，可以直接访问内存地址）。也就是说，**通过这一个或多个变量，就能访问这个内存地址**。

这就引出三个问题：
+ 内存的不正确访问引发的内存安全问题
  + 使用未初始化的内存
  + 对空指针解引用
  + 悬垂指针(使用已经被释放的内存)
  + 缓冲区溢出
  + 非法释放内存(释放未分配的指针或重复释放指针)
+ 由于多个变量指向同一块内存区域导致的数据一致性问题
+ 由于变量在多个线程中传递，导致的数据竞争的问题

Rust 的语言特性为上述问题提供了解决方案，如下表所示：

| 问题                | 解决方案                                                                                                                                                    |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| 使用未初始化的内存         | 编译器禁止变量读取未赋值的变量                                                                                                                                         |
| 对空指针解引用           | 使用 Option 枚举替代空指针                                                                                                                                       |
| 悬垂指针              | 生命周期标识与编译器检查                                                                                                                                            |
| 缓冲区溢出             | 编译器检查，拒绝超越缓冲区边界的数据访问                                                                                                                                    |
| 非法释放内存            | 语言级的 RAII 机制，只有唯一的所有者才有权释放内存                                                                                                                            |
| 多个变量修改同一块内存区域     | 允许多个变量借用所有权，但是同一时间只允许一个可变借用                                                                                                                             |
| 变量在多个线程中传递时的安全问题  | 对基本数据类型用 Sync 和 Send 两个 Trait 标识其线程安全特性，即能否转移所有权或传递可变借用，把这作为基本事实。再利用泛型限定语法和 Trait impl 语法描述出类型线程安全的规则。编译期间使用类似规则引擎的机制，基于基本事实和预定义规则为用户代码中的跨线程数据传递做推理检查。  |


## 变量绑定和所有权赋予
Rust 中为什么叫“变量绑定”而不叫“变量赋值"。我们先来看一段 C++ 代码，以及对应的 Rust 代码。
```c++
#include <iostream>

int main()
{
    int a = 1;
    std::cout << &a << std::endl;   /* 输出 0x62fe1c */
    a = 2;
    std::cout << &a << std::endl;   /* 输出 0x62fe1c */
}
```

```rust
fn main() {
    let a = 1;
    println!("a:{}",a);     // 输出1
    println!("&a:{:p}",&a); // 输出0x9cf974
    //a=2;                  // 编译错误，不可变绑定不能修改绑定的值
    let a = 2;              // 重新绑定
    println!("&a:{:p}",&a); // 输出0x9cfa14地址发生了变化
    let mut b = 1;          // 创建可变绑定
    println!("b:{}",b);     // 输出1
    println!("&b:{:p}",&b); // 输出0x9cfa6c
    b = 2;
    println!("b:{}",b);     // 输出2
    println!("&b:{:p}",&b); // 输出0x9cfa6c地址没有变化
    let b = 2;              // 重新绑定新值
    println!("&b:{:p}",&b); // 输出0x9cfba4地址发生了变化
}
```

可以看出，**赋值是将值写入变量关联的内存区域，绑定是建立变量与内存区域的关联关系，Rust 里，还会把这个内存区域的所有权赋予这个变量**。

不可变绑定的含义是：将变量绑定到一个内存地址，并赋予所有权，通过该变量**只能读取**该地址的数据，不能修改该地址的数据。对应的，可变绑定就可以通过变量修改关联内存区域的数据。从语法上看，有 let 关键字是绑定, 没有就是赋值。

Rust 的变量绑定概念是一个很关键的概念，它是所有权的起点。**有了明确的绑定才有了所有权的归属，同时解绑定的时机也确定了资源释放的时机**。

所有权规则：
+ Rust 中每一个值都被一个变量所拥有，该变量被称为值的所有者 
+ 任何一个时刻，一个值只有一个所有者。 
+ 当所有者(变量)离开作用域范围时，这个值将被丢弃(drop)

> 例外：标准库提供了引用计数指针类型 Rc 和 Arc，它们允许值在某些限制下有多个所有者。
> 
> 丢弃的过程实际上内部是调用 Drop 特型种一个名为 drop 的函数来销毁数据释放内存，类似析构函数

作为所有者，它有如下权利：
+ 控制资源的释放
+ 出借所有权
+ 转移所有权

这种**堆内存资源随着关联的栈上局部变量一起被回收的内存管理特性**，叫作 RAII（Resource Acquisition Is Initialization）。

## 所有权的转移
Rust 可以将值从一个所有者转移到另一个所有者，例如变量赋值、参数传递、函数返回等行为都会发生所有权的移动。

**为什么要转移所有权**？ 我们知道，C/C++/Rust 的变量关联了某个内存区域，但变量总会在表达式中进行操作再赋值给另一个变量，或者在函数间传递。实际上**期望被传递的是变量绑定的内存区域的内容**，如果这块内存区域比较大，复制内存数据到给新的变量就是开销很大的操作。所以需要把**所有权转移给新的变量，同时当前变量放弃所有权**。所以归根结底，转移所有权还是为了**性能**。

**所有权转移的时机**总结下来有以下两种情况：
+ 位置表达式出现在值上下文时转移所有权
+ 变量跨作用域传递时转移所有权

第一条规则是一个精确的学术表达，涉及到位置表达式，值表达式，位置上下文，值上下文等语言概念。它的简单理解就是**各种各样的赋值行为**。能**明确指向某一个内存区域位置的表达式是位置表达式，其它的都是值表达式。各种带有赋值语义的操作的左侧是位置上下文，右侧是值上下文**。

当位置表达式出现在值上下文时，其程序语义就是要把这边位置表达式所指向的数据赋给新的变量，所有权发生转移。

第二条规则是“变量跨作用域时转移所有权”：
+ 变量被花括号内使用
+ match 匹配
+ if let 和 While let
+ 移动语义函数参数传递
+ 闭包捕获移动语义变量
+ 变量从函数内部返回

Rust 要求变量在跨越作用域时明确转移所有权，编译器可以很清楚作用域边界内外哪个变量拥有所有权，能对变量的非法使用作出明确无误的检查，增加的代码的安全性。

**所有权转移的方式**有两种：
+ 移动语义-执行所有权转移
+ 复制语义-不执行转移，只按位复制变量（浅拷贝）

这里把 ”复制语义“定义为所有权转移的方式之一，也就是说“不转移”也是一种转移方式。看起来很奇怪。实际上逻辑是一致的，因为**触发复制执行的时机跟触发转移的时机是一致的**。只是这个数据类型被打上了 Copy 标签 trait, **在应该执行转移动作的时候，编译器改为执行按位复制（浅拷贝）**。

Rust 的标准库中为所有基础类型实现的 Copy Trait。

这里要注意，标准库中的
```rust
impl<T: ?Sized> Copy for &T {}
```
为所有引用类型实现了 Copy, 这意味着我们使用引用参数调用某个函数时，引用变量本身是按位复制的。**标准库没有为可变借用 &mut T 实现 Copy Trait , 因为可变借用只能有一个 (move)**。后文讲闭包捕获变量的所有权时我们可以看到例子。

现在我们解释为什么本文一开始的代码中，相同的代码行为产生不同的结果了。

刚刚提及：如果一个数据结构实现了 Copy 特型，那么它就会使用 Copy 语义，而不是 Move 语义。此时赋值或者传参时，值会自动按位拷贝（即浅拷贝）。比如标准库中的整数、浮点数、字符这些简单类型，不受所有权转移的约束（其实是在转移时，copy让转移变成了复制），它们会直接在栈中复制一份副本。

u32 实现了 Copy，所以在 ` let b = a;` 时候理论上应该发生转移（位置表达式出现在值上下文），但是因为实现了 copy，编译器将转移改成了复制行为。

```rust
fn main() {
    let a = 10u32;
    let b = a;
    println!("{:p}",&a); // 打印地址 0xe9475bfb70
    println!("{:p}",&b); // 打印地址 0xe9475bfb74
}
```
可以看到的的确确是复制，地址都不一样，也就是没有发生所有权转移，进行了复制再绑定。

那为什么 String 不进行拷贝复制呢？

实际上， String 类型是一个复杂类型，由存储在栈中的堆指针、字符串长度、字符串容量共同组成，其中堆指针是最重要的，它指向了真实存储字符串内容的堆内存。

总之 String 类型指向了一个堆上的空间，这里存储着它的真实数据，下面对上面代码中的 let s2 = s1 分成两种情况讨论：
+ 拷贝 String 和存储在堆上的字节数组 如果该语句是拷贝所有数据(深拷贝)，那么无论是 String 本身还是底层的堆上数据，都会被全部拷贝，这对于性能而言会造成非常大的影响
+ 只拷贝 String 本身 这样的拷贝非常快，因为在 64 位机器上就拷贝了 8字节的指针、8字节的长度、8字节的容量，总计 24 字节，但是带来了新的问题，还记得我们之前提到的所有权规则吧？其中有一条就是：一个值只允许有一个所有者，而现在这个值（堆上的真实字符串数据）有了两个所有者：s1 和 s2。

就假定一个值可以拥有两个所有者，会发生什么呢？

当变量离开作用域后，Rust 会自动调用 drop 函数并清理变量的堆内存。不过由于两个 String 变量指向了同一位置。这就有了一个问题：当 s1 和 s2 离开作用域，它们都会尝试释放相同的内存。这是一个叫做 二次释放（double free） 的错误，也是之前提到过的内存安全性 BUG 之一。两次释放（相同）内存会导致内存污染，它可能会导致潜在的安全漏洞。

因此，Rust 这样解决问题：**当 s1 被赋予 s2 后，Rust 认为 s1 不再有效，因此也无需在 s1 离开作用域后 drop 任何东西，这就是把所有权从 s1 转移给了 s2，s1 在被赋予 s2 后就马上失效了**。这就是所有权转移。

如果你在其他语言中听说过术语 浅拷贝(shallow copy) 和 深拷贝(deep copy)，那么拷贝指针、长度和容量而不拷贝数据听起来就像浅拷贝，但是又因为 Rust 同时使第一个变量 s1 无效了，因此这个操作被称为 **移动(move)**，而不是浅拷贝。上面的例子可以解读为 s1 被移动到了 s2 中。

![img.png](/images/lang/rust/ownership-1.png)

## 所有权不会转移的情况
+ clone：即克隆数据（即深拷贝）
+ 数据结构实现了 Copy Trait，不会 move，而会按位拷贝（浅拷贝）
+ borrowing：即 “借用” 数据，可以对值进行 “借用”，以获得值的引用

## 深拷贝 (clone)
**Rust 永远也不会自动创建数据的 “深拷贝”**。因此，任何**自动**的复制都不是深拷贝，可以被认为对运行时性能影响较小。

如果我们确实需要深度复制 String 中堆上的数据，而不仅仅是栈上的数据，可以使用一个叫做 clone 的方法。
```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("{:p}",&s1); // 0x43d658f508
println!("{:p}",&s2); // 0x43d658f520
```
可以看到是拷贝（深）再绑定。

如果代码性能无关紧要，例如初始化程序时或者在某段时间只会执行寥寥数次时，你可以使用 clone 来简化编程。但是对于执行较为频繁的代码(热点路径)，使用 clone 会极大的降低程序性能，需要小心使用！

**只有实现了 Clone 特型的类型才可以进行克隆**，调用 clone() 方法可以拷贝变量的数据，克隆了一个副本，克隆是深拷贝，这样就不会使得原始变量的所有权发生转移，而导致原始变量变成未初始化状态。

### 实现 Clone Trait
自定义类型时，在类型上面加上 #[derive(Clone)] 属性即可实现 Clone 特型，这样该类型有拥有了克隆的能力。
```rust
#[derive(Clone)]
struct Test {
  age: i32
};
```

## 浅拷贝 (Copy Trait)
> trait 称为特型，可以理解成接口

**浅拷贝只发生在栈上**，因此性能很高，在日常编程中，浅拷贝无处不在。

如果值对应的类型实现了 Copy 特型，当有赋值、传参等场景需要"移动"这个值时，值就会自动按位拷贝（浅拷贝），而不是发生所有权的转移。

> 实际上，这种栈上的数据足够简单，而且拷贝非常非常快，只需要复制一个整数大小的内存即可，因此在这种情况下，拷贝的速度远比在堆上创建内存来得快的多。

任何基本类型的组合可以 Copy ，不需要分配内存或某种形式资源的类型是可以 Copy 的。如下是一些 Copy 的类型：
+ 所有整数类型，比如 u32
+ 布尔类型，bool，它的值是 true 和 false
+ 所有浮点数类型，比如 f64
+ 字符类型，char
+ 元组，当且仅当其包含的类型也都是 Copy 的时候。比如，(i32, i32) 是 Copy 的，但 (i32, String) 就不是
+ 不可变引用 &T ，例如转移所有权中的最后一个例子，但是注意: 可变引用 &mut T 是不可以 Copy的

### 实现 Copy Trait
如果自定义的类型（比如结构体）的所有字段本身都是 Copy 类型，则可以在类型上方加上 #[derive(Copy)]，即可为该类型实现 Copy 特型。
```rust
#[derive(Copy)]
struct Test {
  age: i32
}
```
注意：实现了 Copy 的类型也要求实现 Clone，因为 Copy 特型是 Clone 特型的子特型

## 借用 (borrowing)
**获取变量的引用，称之为借用(borrowing)**。

拥有所有权的变量借出其所有权有“引用”和“智能指针”两种方式：
+ 引用（包含可变借用和不可变借用），
+ 智能指针
  + 独占式智能指针 Box<T>
  + 非线程安全的引用计数智能指针 Rc<T>
  + 线程安全的引用计数智能指针 Arc<T>
  + 弱指针 Weak<T>

引用实际上也是指针，指向的是实际的内存位置。

Rust 的核心包中有两个泛型 trait ，core::borrow::Borrow 与 core::borrow::BorrowMut，可以用来表达"借用"的抽象含义，分别代表可变借用和不可变借用。

Borrow 的定义如下：
```rust
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}
```
它只有一个方法，要求返回指定类型的引用。

标准库为所有类型 T 实现了 Borrow Trait, 也为 &T 实现了 Borrow Trait。

```rust
impl<T: ?Sized> Borrow<T> for T {
    fn borrow(&self) -> &T { // 是 fn borrow(self: &Self）的缩写，所以 self 的类型就是 &T
        self
    }
}

impl<T: ?Sized> Borrow<T> for &T {
    fn borrow(&self) -> &T { // 是 fn borrow(self: &Self)->&T 的缩写，所以 self 的类型就是 &&T
        &**self
    }
}
```
这正是 Rust 语言很有意思的地方，非常巧妙的体现了语言的一致性。既然 Borrow<T> 的方法是为了能获取 T 的引用，那么类型 T 和 &T 当然也可以做到这一点。

在 Borrow for T 的实现中， fn borrow(&self)->&T 是 fn borrow(self: &Self)->&T 的缩写，所以 self 的类型就是 &T,可以直接被返回。在 Borrow for &T 的实现中，fn borrow(&self)->&T 是 fn borrow(self: &Self)->&T 的缩写，所以 self 的类型就是 &&T, 需要被两次解引用得到 T, 再返回其引用。

智能指针 Box<T>,Rc<T>,Arc<T>,都实现了 Borrow<T> ，其获取 &T 实例的方式都是两次解引用在取引用。Weak<T> 没有实现 Borrow<T>, 它需要升级成 Rc<T> 才能获取数据。

借用有两个重要的安全规则：
+ 代表借用的变量，其生命周期不能比被借用的变量(所有者)的生命周期长
+ **同一个变量的可变借用只能有一个**

```rust
fn main() {
    let s = String::from("hello");

    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(", world");
}
```

运行：

```text
error[E0596]: cannot borrow `*some_string` as mutable, as it is behind a `&` reference
  --> src\main.rs:20:5
   |
20 |     some_string.push_str(", world");
   |     ^^^^^^^^^^^ `some_string` is a `&` reference, so the data it refers to cannot be borrowed as mutable
   |
help: consider changing this to be a mutable reference
   |
19 | fn change(some_string: &mut String) {
   |                         +++
```
正如变量默认不可变一样，引用指向的值默认也是不可变的。

```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

首先，声明 s 是可变类型，其次创建一个可变的引用 &mut s 和接受可变引用参数 some_string: &mut String 的函数。

只读引用实现了 Copy trait，也就意味着引用的赋值、传参都会产生新的浅拷贝。

**引用型变量的作用域是从它定义起到它最后一次使用时结束**。

**可变引用与不可变引用不能同时存在**。

**可变借用之间的作用域也不能交叠**。

```rust
fn main() {
    let mut a = 10u32;
    let r1 = &a;
    a = 20;
    
    println!("{r1}");
}
```

```text
error[E0596]: cannot borrow `*some_string` as mutable, as it is behind a `&` reference
//            不能给a赋值，因为它被借用了
  --> src\main.rs:26:5
   |
26 |     some_string.push_str(", world");
   |     ^^^^^^^^^^^ `some_string` is a `&` reference, so the data it refers to cannot be borrowed as mutable
   |
help: consider changing this to be a mutable reference
   |
25 | fn change(some_string: &mut String) {
   |                         +++
```

提示在有借用的情况下，不能对所有权变量进行更改值的操作（写操作）。

**也就是在借用的前提下，即使是变量的所有者也不能进行写操作，直到借用的作用域结束**。

总结一下：
+ 所有权型变量的作用域是从它定义时开始到所属那层花括号结束。
+ 引用型变量的作用域是从它定义起到它最后一次使用时结束。
+ 引用（不可变引用和可变引用）型变量的作用域不会长于所有权变量的作用域。这是肯定的，不然就会出现悬锤引用，这是典型的内存安全问题。
+ 一个所有权型变量的不可变引用可以同时存在多个，可以复制多份。
+ 一个所有权型变量的可变引用与不可变引用的作用域不能交叠，也可以说不能同时存在。
+ 某个时刻对某个所有权型变量只能存在一个可变引用，不能有超过一个可变借用同时存在，也可以说，对同一个所有权型变量的可变借用之间的作用域不能交叠。
+ 在有借用存在的情况下，不能通过原所有权型变量对值进行更新。当借用完成后（借用的作用域结束后），物归原主，又可以使用所有权型变量对值做更新操作了。

```rust
fn main() {
    let mut a = 10u32;
    let r1 = &mut a;
    let r2 = r1;
    
    println!("{r1}")
}
```
cargo run:
```text
error[E0596]: cannot borrow `*some_string` as mutable, as it is behind a `&` reference
  --> src\main.rs:30:5
   |
30 |     some_string.push_str(", world");
   |     ^^^^^^^^^^^ `some_string` is a `&` reference, so the data it refers to cannot be borrowed as mutable
   |
help: consider changing this to be a mutable reference
   |
29 | fn change(some_string: &mut String) {
   |                         +++
```

**可变引用的再赋值，会执行移动操作**。赋值后，原来的那个可变引用变量就不能用了。这有点类似于所有权的转移，因此**一个所有权型变量的可变引用也具有所有权特征**，它可以被理解为那个所有权变量的独家代理，具有**排它性**。

### 多级引用
```rust
fn main() {
    let mut a1 = 10u32;
    let mut a2 = 15u32;

    let mut b = &mut a1;
    b = &mut a2;

    let mut c = &a1;
    c = &a2;
}
```

```text
error[E0596]: cannot borrow `*some_string` as mutable, as it is behind a `&` reference
  --> src\main.rs:33:5
   |
33 |     some_string.push_str(", world");
   |     ^^^^^^^^^^^ `some_string` is a `&` reference, so the data it refers to cannot be borrowed as mutable
   |
help: consider changing this to be a mutable reference
   |
32 | fn change(some_string: &mut String) {
   |                         +++
```

```rust
fn main() {
    let mut a1 = 10u32;
    let mut b = &mut a1;
    *b = 20;

    let c = &mut b;
    **c = 30;          // 多级解引用操作
    
    println!("{c}");
}
// 输出 
30
```

假如我们解引用错误会怎样：
```rust
fn main() {
    let mut a1 = 10u32;
    let mut b = &mut a1;
    *b = 20;

    let c = &mut b;
    *c = 30;            // 这里对二级可变引用只使用一级解引用操作
    
    println!("{c}");
}
```

```text
error[E0308]: mismatched types
  --> src\main.rs:36:10
   |
36 |     *c = 30;            // 这里对二级可变引用只使用一级解引用操作
   |     --   ^^ expected `&mut u32`, found integer
   |     |
   |     expected due to the type of this binding
   |
help: consider dereferencing here to assign to the mutably borrowed value
   |
36 |     **c = 30;            // 这里对二级可变引用只使用一级解引用操作
   |     +
```
它正确识别到了中间引用的类型为 &mut u32，而我们却要给它赋值为 u32。

+ 对于多级可变引用，要利用可变引用去修改目标资源的值的时候，需要做正确的多级解引用操作，比如例子中的 **c，做了两级解引用。
+ 只有全是多级可变引用的情况下，才能修改到目标资源的值。
+ 对于多级引用（包含可变和不可变），打印语句中，可以自动为我们解引用正确的层数，直到访问到目标资源的值，这很符合人的直觉和业务的需求。
