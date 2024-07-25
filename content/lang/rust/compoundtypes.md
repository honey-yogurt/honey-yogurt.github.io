---
title : '复合类型'
date : 2024-07-24T21:19:46+08:00
---

## 元组

元组是由**多种类型**组合到一起形成的，因此它是复合类型，**元组的长度是固定的，元组中元素的顺序也是固定的**。

```rust
fn main() {
    let tup: (i32, f64, u8) = (500, 6.4, 1);
}
```

变量 `tup` 被绑定了一个元组值 `(500, 6.4, 1)`，该元组的类型是 `(i32, f64, u8)`。

可以使用模式匹配或者 `.` 操作符来获取元组中的值。

### 匹配模式结构元组

模式匹配可以让我们一次性把元组中的值**全部或者部分**获取出来。

```rust
fn main() {
    let tup = (500, 6.4, 1);

    let (x, y, z) = tup;

    println!("The value of y is: {}", y);
}
```

使用 `let (x, y, z) = tup;` 来完成一次模式匹配，因为元组是 `(n1, n2, n3)` 形式的，因此我们用一模一样的 `(x, y, z)` 形式来进行匹配，元组中对应的值会绑定到变量 `x`， `y`， `z`上。这就是解构：**用同样的形式把一个复杂对象中的值匹配出来**。

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() 返回字符串的长度

    (s, length)
}
```



### 用 . 来访问元组

```rust
fn main() {
    let x: (i32, f64, u8) = (500, 6.4, 1);

    let five_hundred = x.0;

    let six_point_four = x.1;

    let one = x.2;
}
```

元组的索引从 0 开始。

## 结构体

### 基本语法

#### 定义结构体

一个结构体由几部分组成：

- 通过关键字 `struct` 定义
- 一个清晰明确的结构体 `名称`
- 几个有名字的结构体 `字段`

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```

#### 创建结构体实例

```rust
    let user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };
```

1. 初始化实例时，**每个字段**都需要进行初始化
2. 初始化时的字段顺序**不需要**和结构体定义时的顺序一致

#### 访问结构体字段

通过 `.` 操作符即可访问结构体实例内部的字段值，也可以修改它们：

```rust
    let mut user1 = User {
        email: String::from("someone@example.com"),
        username: String::from("someusername123"),
        active: true,
        sign_in_count: 1,
    };

    user1.email = String::from("anotheremail@example.com");
```

必须要将结构体实例声明为可变的，才能修改其中的字段，Rust **不支持将某个结构体某个字段标记为可变**。

#### 简化结构体创建

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email: email,
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```

简化如下：

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,
        username,
        active: true,
        sign_in_count: 1,
    }
}
```

**当函数参数和结构体字段同名时，可以直接使用缩略的方式进行初始化**。

#### 结构体更新语法

根据已有的结构体实例，创建新的结构体实例。

例如根据已有的 `user1` 实例来构建 `user2`：

```rust
  let user2 = User {
        active: user1.active,
        username: user1.username,
        email: String::from("another@example.com"),
        sign_in_count: user1.sign_in_count,
    };
```

简化：

```rust
  let user2 = User {
        email: String::from("another@example.com"),
        ..user1
    };
```

因为 `user2` 仅仅在 `email` 上与 `user1` 不同，因此我们只需要对 `email` 进行赋值，剩下的通过结构体更新语法 `..user1` 即可完成。

`..` 语法表明凡是我们**没有显式声明的字段**，全部从 `user1` 中自动获取。需要注意的是 `..user1` 必须在结构体的**尾部**使用。

> 结构体更新语法跟赋值语句 `=` 非常相像，因此在上面代码中，`user1` 的部分字段所有权被转移到 `user2` 中：`username` 字段发生了所有权转移，作为结果，`user1` 无法再被使用，但是没有发生所有权转移的`user1`中的字段是可以使用的。
>
> Q：明明有三个字段进行了自动赋值，为何只有 `username` 发生了所有权转移？
>
> A：实现了 `Copy` 特征的类型无需所有权转移，可以直接在赋值时进行 数据拷贝，其中 `bool` 和 `u64` 类型就实现了 `Copy` 特征，因此 `active` 和 `sign_in_count` 字段在赋值给 `user2` 时，仅仅发生了拷贝，而不是所有权转移。
>
> 值得注意的是：`username` 所有权被转移给了 `user2`，导致了 `user1` 无法再被使用，但是并不代表 `user1` 内部的其它字段不能被继续使用：
>
> ```rust
> let user1 = User {
>     email: String::from("someone@example.com"),
>     username: String::from("someusername123"),
>     active: true,
>     sign_in_count: 1,
> };
> let user2 = User {
>     active: user1.active,
>     username: user1.username,
>     email: String::from("another@example.com"),
>     sign_in_count: user1.sign_in_count,
> };
> println!("{}", user1.active);
> // 下面这行会报错
> println!("{:?}", user1);
> 
> ```

### 结构体的内存排列

```rust
#[derive(Debug)]
 struct File {
   name: String,
   data: Vec<u8>,
 }

 fn main() {
   let f1 = File {
     name: String::from("f1.txt"),
     data: Vec::new(),
   };

   let f1_name = &f1.name;
   let f1_length = &f1.data.len();

   println!("{:?}", f1);
   println!("{} is {} bytes long", f1_name, f1_length);
 }
```

上面定义的 `File` 结构体在内存中的排列如下图所示：

![img](/images/lang/rust/compoundtypes-2.png)

从图中可以清晰地看出 `File` 结构体两个字段 `name` 和 `data` 分别拥有底层两个 `[u8]` 数组的所有权(`String` 类型的底层也是 `[u8]` 数组)，通过 `ptr` 指针指向底层数组的内存地址，这里你可以把 `ptr` 指针理解为 Rust 中的引用类型。

**把结构体中具有所有权的字段转移出去后，将无法再访问该字段，但是可以正常访问其它的字段**。

### 元组结构体（Tuple Struct）

结构体必须要有名称，但是结构体的字段可以没有名称，这种结构体长得很像元组，因此被称为元组结构体。

```rust
    struct Color(i32, i32, i32);
    struct Point(i32, i32, i32);

    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
```

元组结构体在你希望有一个整体名称，但是又不关心里面字段的名称时将非常有用。

### 单元结构体（Unit-like Struct）

没有任何字段和属性。

如果你定义一个类型，但是不关心该类型的内容, 只关心它的行为时，就可以使用 `单元结构体`：

```rust
struct AlwaysEqual;

let subject = AlwaysEqual;

// 我们不关心 AlwaysEqual 的字段数据，只关心它的行为，因此将它声明为单元结构体，然后再为它实现某个特征
impl SomeTrait for AlwaysEqual {

}
```

### 结构体数据的所有权

在之前的 `User` 结构体的定义中，有一处细节：我们使用了自身拥有所有权的 `String` 类型而不是基于引用的 `&str` 字符串切片类型。这是一个有意而为之的选择：因为我们想要这个结构体拥有它所有的数据，而不是从其它地方借用数据。

你也可以让 `User` 结构体从其它对象借用数据，不过这么做，就需要引入生命周期(lifetimes)这个新概念，简而言之，生命周期能确保结构体的作用范围要比它所借用的数据的作用范围要小。

总之，如果你想在结构体中使用一个引用，就必须加上生命周期，否则就会报错：

```rust
struct User {
    username: &str,
    email: &str,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    let user1 = User {
        email: "someone@example.com",
        username: "someusername123",
        active: true,
        sign_in_count: 1,
    };
}
```

`cargo run`

```tex
error[E0106]: missing lifetime specifier
 --> src/main.rs:2:15
  |
2 |     username: &str,
  |               ^ expected named lifetime parameter // 需要一个生命周期
  |
help: consider introducing a named lifetime parameter // 考虑像下面的代码这样引入一个生命周期
  |
1 ~ struct User<'a> {
2 ~     username: &'a str,
  |

error[E0106]: missing lifetime specifier
 --> src/main.rs:3:12
  |
3 |     email: &str,
  |            ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct User<'a> {
2 |     username: &str,
3 ~     email: &'a str,
  |

```

### 使用 #[derive(Debug)] 来打印结构体的信息

在前面的代码中我们使用 `#[derive(Debug)]` 对结构体进行了标记，这样才能使用 `println!("{:?}", s);` 的方式对其进行打印输出。

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!("rect1 is {}", rect1);
}
```

```tex
error[E0277]: `Rectangle` doesn't implement `std::fmt::Display`
```

如果不加，就会提示我们结构体 `Rectangle` 没有实现 `Display` 特征，这是因为如果我们使用 `{}` 来格式化输出，那对应的类型就必须实现 `Display` 特征，以前学习的基本类型，都默认实现了该特征。

那么结构体为什么不默认实现 `Display` 特征呢？原因在于结构体较为复杂，例如考虑以下问题：你想要逗号对字段进行分割吗？需要括号吗？加在什么地方？所有的字段都应该显示？类似的还有很多，由于这种复杂性，Rust 不希望猜测我们想要的是什么，而是把选择权交给我们自己来实现：如果要用 `{}` 的方式打印结构体，那就自己实现 `Display` 特征。

接下来继续阅读报错：

```tex
= help: the trait `std::fmt::Display` is not implemented for `Rectangle`
= note: in format strings you may be able to use `{:?}` (or {:#?} for pretty-print) instead
```

上面提示我们使用 `{:?}` 来试试，这个方式我们在本文的前面也见过，下面来试试:

```rust
println!("rect1 is {:?}", rect1);
```

```tex
error[E0277]: `Rectangle` doesn't implement `Debug`
= help: the trait `Debug` is not implemented for `Rectangle`
= note: add `#[derive(Debug)]` to `Rectangle` or manually `impl Debug for Rectangle`
```

提示中有一行： `add #[derive(Debug)] to Rectangle`。

首先，Rust 默认不会为我们实现 `Debug`，为了实现，有两种方式可以选择：

- 手动实现
- 使用 `derive` 派生实现

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let rect1 = Rectangle {
        width: 30,
        height: 50,
    };

    println!("rect1 is {:?}", rect1);
    println!("rect1 is {:#?}", rect1);
}
// rect1 is Rectangle { width: 30, height: 50 }
//rect1 is Rectangle {
//    width: 30,
//    height: 50,
//}
```

还有一个简单的输出 debug 信息的方法，那就是使用 `dbg!` 宏，它会拿走表达式的所有权，然后打印出相应的文件名、行号等 debug 信息，当然还有我们需要的表达式的求值结果。**除此之外，它最终还会把表达式值的所有权返回**。

`dbg!` 输出到标准错误输出 `stderr`，而 `println!` 输出到标准输出 `stdout`。

```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 2;
    let rect1 = Rectangle {
        width: dbg!(30 * scale),
        height: 50,
    };

    dbg!(&rect1);
}
```

```tex
$ cargo run
[src/main.rs:10] 30 * scale = 60
[src/main.rs:14] &rect1 = Rectangle {
    width: 60,
    height: 50,
}

```

## 枚举

枚举(enum 或 enumeration)允许你通过列举可能的成员来定义一个**枚举类型**，例如扑克牌花色：

```rust
enum PokerSuit {
  Clubs,
  Spades,
  Diamonds,
  Hearts,
}
```

**枚举类型是一个类型，它会包含所有可能的枚举成员, 而枚举值是该类型中的具体某个成员的实例。**

#### 枚举值

```rust
let heart = PokerSuit::Hearts;
let diamond = PokerSuit::Diamonds;
```

```rust
fn main() {
    let heart = PokerSuit::Hearts;
    let diamond = PokerSuit::Diamonds;

    print_suit(heart);
    print_suit(diamond);
}

fn print_suit(card: PokerSuit) {
    // 需要在定义 enum PokerSuit 的上面添加上 #[derive(Debug)]，否则会报 card 没有实现 Debug
    println!("{:?}",card);
}
```

通过 `::` 操作符来访问 `PokerSuit` 下的具体成员。

接下来，我们想让扑克牌变得更加实用，那么需要给每张牌赋予一个值：`A`(1)-`K`(13)，这样再加上花色，就是一张真实的扑克牌了，例如红心 A。

目前来说，枚举值还不能带有值，因此先用结构体来实现：

```
enum PokerSuit {
    Clubs,
    Spades,
    Diamonds,
    Hearts,
}

struct PokerCard {
    suit: PokerSuit,
    value: u8
}

fn main() {
   let c1 = PokerCard {
       suit: PokerSuit::Clubs,
       value: 1,
   };
   let c2 = PokerCard {
       suit: PokerSuit::Diamonds,
       value: 12,
   };
}
```

通过结构体 `PokerCard` 来代表一张牌，结构体的 `suit` 字段表示牌的花色，类型是 `PokerSuit` 枚举类型，`value` 字段代表扑克牌的数值。

简洁版：

```rust
enum PokerCard {
    Clubs(u8),
    Spades(u8),
    Diamonds(u8),
    Hearts(u8),
}

fn main() {
   let c1 = PokerCard::Spades(5);
   let c2 = PokerCard::Diamonds(13);
}
```

直接将数据信息关联到枚举成员上，省去近一半的代码，这种实现是不是更优雅？

不仅如此，同一个枚举类型下的不同成员还能持有不同的数据类型，例如让某些花色打印 `1-13` 的字样，另外的花色打印上 `A-K` 的字样：

```rust
enum PokerCard {
    Clubs(u8),
    Spades(u8),
    Diamonds(char),
    Hearts(char),
}

fn main() {
   let c1 = PokerCard::Spades(5);
   let c2 = PokerCard::Diamonds('A');
}
```

一个来自标准库中的例子：

```rust
struct Ipv4Addr {
    // --snip--
}

struct Ipv6Addr {
    // --snip--
}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}
```

**任何类型的数据都可以放入枚举成员中**: 例如字符串、数值、结构体甚至另一个枚举。

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let m1 = Message::Quit;
    let m2 = Message::Move{x:1,y:1};
    let m3 = Message::ChangeColor(255,255,0);
}
```

该枚举类型代表一条消息，它包含四个不同的成员：

- `Quit` 没有任何关联数据
- `Move` 包含一个匿名结构体
- `Write` 包含一个 `String` 字符串
- `ChangeColor` 包含三个 `i32`

当然，我们也可以用结构体的方式来定义这些消息：

```rust
struct QuitMessage; // 单元结构体
struct MoveMessage {
    x: i32,
    y: i32,
}
struct WriteMessage(String); // 元组结构体
struct ChangeColorMessage(i32, i32, i32); // 元组结构体
```

由于每个结构体都有自己的类型，因此我们无法在需要同一类型的地方进行使用，例如某个函数它的功能是接受消息并进行发送，那么用枚举的方式，就可以接收不同的消息，但是用结构体，该函数无法接受 4 个不同的结构体作为参数。

而且从代码规范角度来看，枚举的实现更简洁，**代码内聚性更强**，不像结构体的实现，分散在各个地方。

### 同一化类型

例如我们有一个 WEB 服务，需要接受用户的长连接，假设连接有两种：`TcpStream` 和 `TlsStream`，但是我们希望对这两个连接的处理流程相同，也就是用同一个函数来处理这两个连接，代码如下：

```rust
enum Websocket {
  Tcp(Websocket<TcpStream>),
  Tls(Websocket<native_tls::TlsStream<TcpStream>>),
}

fn new (stream: TcpStream) {
  let mut s = stream;
  if tls {
    s = negotiate_tls(stream)
  }

  // websocket是一个WebSocket<TcpStream>或者
  //   WebSocket<native_tls::TlsStream<TcpStream>>类型
  websocket = WebSocket::from_raw_socket(
    s, ......)
}
```

### Option 枚举用于处理空值

在其它编程语言中，往往都有一个 `null` 关键字，该关键字用于表明一个变量当前的值为空（不是零值，例如整型的零值是 0），也就是不存在值。当你对这些 `null` 进行操作时，例如调用一个方法，就会直接抛出**null 异常**，导致程序的崩溃，因此我们在编程时需要格外的小心去处理这些 `null` 空值。

尽管如此，空值的表达依然非常有意义，因为空值表示当前时刻变量的值是缺失的。有鉴于此，Rust 吸取了众多教训，决定**抛弃 `null`**，而改为使用 `Option` 枚举变量来表述这种结果。

`Option` 枚举包含两个成员，一个成员表示含有值：`Some(T)`, 另一个表示没有值：`None`，定义如下：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

其中 `T` 是泛型参数，`Some(T)`表示该枚举成员的数据类型是 `T`，换句话说，`Some` 可以包含任何类型的数据。

`Option<T>` 枚举是如此有用以至于它被包含在了 `prelude`（prelude 属于 Rust 标准库，Rust 会将最常用的类型、函数等提前引入其中，省得我们再手动引入）之中，你不需要将其显式引入作用域。另外，它的成员 `Some` 和 `None` 也是如此，无需使用 `Option::` 前缀就可直接使用 `Some` 和 `None`。

```rust
let some_number = Some(5);
let some_string = Some("a string");

let absent_number: Option<i32> = None;
```

如果使用 `None` 而不是 `Some`，需要告诉 Rust `Option<T>` 是什么类型的，因为编译器只通过 `None` 值无法推断出 `Some` 成员保存的值的类型。

当有一个 `Some` 值时，我们就知道存在一个值，而这个值保存在 `Some` 中。当有个 `None` 值时，在某种意义上，它跟空值具有相同的意义：并没有一个有效的值。那么，`Option<T>` 为什么就比空值要好呢？

简而言之，因为 `Option<T>` 和 `T`（这里 `T` 可以是任何类型）是不同的类型，例如，这段代码不能编译，因为它尝试将 `Option<i8>`(`Option<T>`) 与 `i8`(`T`) 相加：

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);

let sum = x + y;
```

事实上，错误信息意味着 Rust 不知道该如何将 `Option<i8>` 与 `i8` 相加，因为它们的类型不同。当在 Rust 中拥有一个像 `i8` 这样类型的值时，编译器确保它总是有一个有效的值，我们可以放心使用而无需做空值检查。只有当使用 `Option<i8>`（或者任何用到的类型）的时候才需要担心可能没有值，而编译器会确保我们在使用值之前处理了为空的情况。

换句话说，在对 `Option<T>` 进行 `T` 的运算之前必须将其转换为 `T`。通常这能帮助我们捕获到空值最常见的问题之一：期望某值不为空但实际上为空的情况。

不再担心会错误的使用一个空值，会让你对代码更加有信心。为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 `Option<T>` 中。接着，当使用这个值时，必须明确的处理值为空的情况。只要一个值不是 `Option<T>` 类型，你就 **可以** 安全的认定它的值不为空。这是 Rust 的一个经过深思熟虑的设计决策，来限制空值的泛滥以增加 Rust 代码的安全性。

那么当有一个 `Option<T>` 的值时，如何从 `Some` 成员中取出 `T` 的值来使用它呢？`Option<T>` 枚举拥有大量用于各种情况的方法：你可以查看[它的文档](https://doc.rust-lang.org/std/option/enum.Option.html)。熟悉 `Option<T>` 的方法将对你的 Rust 之旅非常有用。

总的来说，为了使用 `Option<T>` 值，需要编写处理每个成员的代码。你想要一些代码只当拥有 `Some(T)` 值时运行，允许这些代码使用其中的 `T`。也希望一些代码在值为 `None` 时运行，这些代码并没有一个可用的 `T` 值。`match` 表达式就是这么一个处理枚举的控制流结构：它会根据枚举的成员运行不同的代码，这些代码可以使用匹配到的值中的数据。

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

`plus_one` 通过 `match` 来处理不同 `Option` 的情况。

## 数组

在 Rust 中，最常用的数组有两种，第一种是速度很快但是长度固定的数组`array`，第二种是可动态增长的但是有性能损耗的动态数组 `Vector`。

### Array

array的具体定义很简单：将多个**类型相同**的元素依次组合在一起，就是一个数组。

+ 长度固定
+ 元素必须有相同的类型
+ 依次线性排列

#### 创建数组

```rust
let a = [1, 2, 3, 4, 5];
```

**数组 `array` 是存储在栈上**，性能也会非常优秀。与此对应，**动态数组 `Vector` 是存储在堆上**。当你不确定是使用数组还是动态数组时，那就应该使用后者。

在一些时候，还需要为**数组声明类型**，如下所示：

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];
```

数组类型是通过方括号语法声明，`i32` 是元素类型，分号后面的数字 `5` 是数组长度，数组类型也从侧面说明了**数组的元素类型要统一，长度要固定**。

还可以使用下面的语法初始化一个**某个值重复出现 N 次的数组**：

```rust
let a = [3; 5];
```

`a` 数组包含 `5` 个元素，这些元素的初始化值为 3。

#### 访问数组元素

通过索引的方式来访问存放其中的元素，数组的索引下标是从 0 开始的。

```rust
fn main() {
    let a = [9, 8, 7, 6, 5];

    let first = a[0]; // 获取a数组第一个元素
    let second = a[1]; // 获取第二个元素
}
```

#### 越界访问

当你尝试使用索引访问元素时，Rust 将检查你指定的索引是否小于数组长度。如果索引大于或等于数组长度，Rust 会出现 **panic**。这种检查只能在**运行时**进行。

#### **数组元素是非基本类型**

**基本类型在Rust中赋值是以Copy的形式**，`let array=[3;5]`底层就是不断的Copy出来的，但复杂类型都没有深拷贝，只能一个个创建。

```rust
let array = [String::from("rust is good!"); 8];

println!("{:#?}", array);
```

故编译错误：

```
the trait bound `String: Copy` is not satisfied
 --> src\main.rs:2:18
  |
2 |     let array = [String::from("rust is good!"); 8];
  |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Copy` is not implemented for `String`
  |
  = note: the `Copy` trait is required because this value will be copied for each element of the array
  = help: consider using `core::array::from_fn` to initialize the array
  = help: see https://doc.rust-lang.org/stable/std/array/fn.from_fn.html for more information
```

**正确的写法**，应该调用`std::array::from_fn`

```rust
let array: [String; 8] = std::array::from_fn(|_i| String::from("rust is good!"));

println!("{:#?}", array);

```

#### 示例

```rust
fn main() {
  // 编译器自动推导出one的类型
  let one             = [1, 2, 3];
  // 显式类型标注
  let two: [u8; 3]    = [1, 2, 3];
  let blank1          = [0; 3];
  let blank2: [u8; 3] = [0; 3];

  // arrays是一个二维数组，其中每一个元素都是一个数组，元素类型是[u8; 3]
  let arrays: [[u8; 3]; 4]  = [one, two, blank1, blank2];

  // 借用arrays的元素用作循环中
  for a in &arrays {
    print!("{:?}: ", a);
    // 将a变成一个迭代器，用于循环
    // 你也可以直接用for n in a {}来进行循环
    for n in a.iter() {
      print!("\t{} + 10 = {}", n, n+10);
    }

    let mut sum = 0;
    // 0..a.len,是一个 Rust 的语法糖，其实就等于一个数组，元素是从0,1,2一直增加到到a.len-1
    for i in 0..a.len() {
      sum += a[i];
    }
    println!("\t({:?} = {})", a, sum);
  }
}
```



### Vector

动态数组只能存储相同类型的元素。

#### 创建动态数组

使用 `Vec::new` 创建动态数组是最 rusty 的方式，它调用了 `Vec` 中的 `new` 关联函数：

```rust
let v: Vec<i32> = Vec::new();
```

这里，`v` 被显式地声明了类型 `Vec<i32>`，这是因为 Rust 编译器无法从 `Vec::new()` 中得到任何关于类型的暗示信息，因此也无法推导出 `v` 的具体类型，但是当你向里面增加一个元素后，一切又不同了：

```rust
let mut v = Vec::new();
v.push(1);
```

此时，`v` 就无需手动声明类型，因为编译器通过 `v.push(1)`，推测出 `v` 中的元素类型是 `i32`，因此推导出 `v` 的类型是 `Vec<i32>`。

如果预先知道要存储的元素个数，可以使用 `Vec::with_capacity(capacity)` 创建动态数组，这样可以避免因为插入大量新数据导致频繁的内存分配和拷贝，提升性能。



还可以使用宏 `vec!` 来创建数组，与 `Vec::new` 有所不同，前者能在创建同时给予初始化值：

```rust
let v = vec![1,2,3];
```

同样，此处的 `v` 也无需标注类型，编译器只需检查它内部的元素即可自动推导出 `v` 的类型是 `Vec<i32>`。

#### 更新 Vector

向数组**尾部**添加元素，可以使用 `push` 方法：

```rust
let mut v = Vec::new();
v.push(1);
```

与其它类型一样，必须将 `v` 声明为 `mut` 后，才能进行修改。

#### Vector 与其元素共存亡

`Vector` 类型在超出作用域范围后，会被自动删除：

```rust
{
    let v = vec![1,2,3];
    //...
}// v 超出作用域并在此处被删除
```

当 `Vector` 被删除后，它内部存储的所有内容也会随之被删除。目前来看，这种解决方案简单直白，但是当 `Vector` 中的元素被引用后，事情可能会没那么简单。

#### 从 Vector 中读取元素

读取指定位置的元素有两种方式可选：

- 通过下标索引访问。
- 使用 `get` 方法。

```rust
let v = vec![1,2,3,4,5];
let third: &i32 = &v[2];
println!("第三个元素是 {}",third);

match v.get(2) {
    Some(third) => println!("第三个元素是 {third}");
    None => println!("没有第三个元素");
}
```

集合类型的索引下标都是从 `0` 开始，`&v[2]` 表示借用 `v` 中的第三个元素，最终会获得该元素的引用。而 `v.get(2)` 也是访问第三个元素，但是有所不同的是，它返回了 `Option<&T>`，因此还需要额外的 `match` 来匹配解构出具体的值。

这两种方式都能成功的读取到指定的数组元素，既然如此为什么会存在两种方法？

```rust
let v = vec![1, 2, 3, 4, 5];

let does_not_exist = &v[100];
let does_not_exist = v.get(100);
```

运行以上代码，`&v[100]` 的访问方式会导致程序无情报错退出，因为发生了数组越界访问。 但是 `v.get` 就不会，它在内部做了处理，有值的时候返回 `Some(T)`，无值的时候返回 `None`，因此 `v.get` 的使用方式非常安全。

#### 同时借用多个数组元素

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];

v.push(6);

println!("The first element is: {first}");
```

首先 `first = &v[0]` 进行了不可变借用，`v.push` 进行了可变借用，如果 `first` 在 `v.push` 之后不再使用，那么该段代码可以成功编译。可是上面的代码中，`first` 这个不可变借用在可变借用 `v.push` 后被使用了，那么编译器就会报错。

按理来说，这两个引用不应该互相影响的：一个是查询元素，一个是在数组尾部插入元素，完全不相干的操作，为何编译器要这么严格呢？

原因在于：数组的大小是可变的，当旧数组的大小不够用时，Rust 会重新分配一块更大的内存空间，然后把**旧数组拷贝**过来。这种情况下，之前的引用显然会指向一块无效的内存，这非常 rusty —— 对用户进行严格的教育。

从零手撸一个动态数组[Rustonomicon](https://nomicon.purewhite.io/vec/vec.html)。

#### 迭代遍历 Vector 中的元素

如果想要依次访问数组中的元素，可以使用迭代的方式去遍历数组，这种方式比用下标的方式去遍历数组更安全也更高效（每次下标访问都会触发数组边界检查）：

```rust
let v = vec![1, 2, 3];
for i in &v {
    println!("{i}");
}
```

也可以在迭代过程中，修改 `Vector` 中的元素：

```rust
let mut v = vec![1, 2, 3];
for i in &mut v {
    *i += 10
}
```

#### 存储不同类型的元素

先来看看通过枚举如何实现：

```rust
#[derive(Debug)]
enum IpAddr {
    V4(String),
    V6(String)
}
fn main() {
    let v = vec![
        IpAddr::V4("127.0.0.1".to_string()),
        IpAddr::V6("::1".to_string())
    ];

    for ip in v {
        show_addr(ip)
    }
}

fn show_addr(ip: IpAddr) {
    println!("{:?}",ip);
}
```



## 切片

切片允许你**引用集合中部分连续的元素序列**，而不是引用整个集合。

对于字符串而言，切片就是对 `String` 类型中某一部分的引用，它看起来像这样：

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

`hello` 没有引用整个 `String s`，而是引用了 `s` 的一部分内容，通过 `[0..5]` 的方式来指定。

这就是创建切片的语法，使用方括号包括的一个序列：**[开始索引..终止索引]**，其中开始索引是切片中第一个元素的索引位置，而终止索引是最后一个元素后面的索引位置，也就是这是一个 **右半开区间**。在切片数据结构内部会保存开始的位置和切片的长度，其中长度是通过 `终止索引` - `开始索引` 的方式计算得来的。

对于 `let world = &s[6..11];` 来说，`world` 是一个切片，该切片的指针指向 `s` 的第 7 个字节(索引从 0 开始, 6 是第 7 个字节)，且该切片的长度是 `5` 个字节。

![img](/images/lang/rust/compoundtypes-1.jpg)

在使用 Rust 的 `..` [range 序列](https://course.rs/basic/base-type/numbers.html#序列range)语法时，如果你想从索引 0 开始，可以使用如下的方式，这两个是等效的：

```rust
let s = String::from("hello");

let slice = &s[0..2];
let slice = &s[..2];
```

同样的，如果你的切片想要包含 `String` 的最后一个字节，则可以这样使用：

```rust
let s = String::from("hello");

let len = s.len();

let slice = &s[4..len];
let slice = &s[4..];
```

你也可以截取完整的 `String` 切片：

```rust
let s = String::from("hello");

let len = s.len();

let slice = &s[0..len];
let slice = &s[..];
```

在对字符串使用切片语法时需要格外小心，切片的索引必须落在字符之间的边界位置，也就是 **UTF-8 字符的边界**，例如中文在 UTF-8 中占用三个字节，下面的代码就会崩溃：

```rust
 let s = "中国人";
 let a = &s[0..2];
 println!("{}",a);
```

### 其他切片

因为切片是对集合的部分引用，因此不仅仅字符串有切片，其它集合类型也有，例如数组：

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];

let slice: &[i32] = &a[1..3];

assert_eq!(slice, &[2, 3]);

```

上面的数组切片 `slice` 的类型是`&[i32]`，与之对比，数组的类型是`[i32;5]`，简单总结下切片的特点：

- 切片的长度可以与数组不同，并不是固定的，而是取决于你使用时指定的起始和结束位置
- 创建切片的代价非常小，因为切片只是针对底层数组的一个引用
- **切片类型[T]拥有不固定的大小，而切片引用类型&[T]则具有固定的大小**，因为 Rust 很多时候都需要固定大小数据类型，因此&[T]更有用,`&str`字符串切片也同理

## 字符串

