+++
title = '流程控制'
date = 2024-07-25T22:05:10+08:00
+++

## 条件语句

```rust
if condition == true {
    // A...
} else {
    // B...
}
```

```rust
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };

    println!("The value of number is: {}", number);
}
```

+ **if 语句块是表达式**，这里我们使用 if 表达式的返回值来给 number 进行赋值：number 的值是 5
+ 用 if 来赋值时，要保证每个分支返回的类型一样(事实上，这种说法不完全准确，还可以在循环中配合 continue，break 返回 )，此处返回的 5 和 6 就是同一个类型，如果返回类型不一致就会报错

```rust
let mut v = 0;
for i in 1..10 {
    v = if i == 9 {
        continue
    } else {
        i
    }
}
println!("{}", v);
```

```rust
fn main() {
    let n = 6;

    if n % 4 == 0 {
        println!("number is divisible by 4");
    } else if n % 3 == 0 {
        println!("number is divisible by 3");
    } else if n % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
}
```

会按照自上至下的顺序执行每一个分支判断，一旦成功，则跳出 if 语句块。

## 循环控制
### for

```rust
for 元素 in 集合 {
  // 使用元素干一些你懂我不懂的事情
}
```

使用 for 时我们往往使用**集合的引用形式**，除非你不想在后面的代码中继续使用该集合（比如我们这里使用了 container 的引用）。如果**不使用引用的话，所有权会被转移（move）到 for 语句块中**，后面就无法再使用这个集合了)：

```rust
for item in &container {
  // ...
}
```

> 对于实现了 copy 特征的数组(例如 [i32; 10] )而言， for item in arr 并不会把 arr 的所有权转移，而是直接对其进行了拷贝，因此循环之后仍然可以使用 arr 。
>

如果想在循环中，修改该元素，可以使用 mut 关键字：

```rust
for item in &mut collection {
  // ...
}
```

| 使用方法                    | 等价使用方式                                    | 所有权     |
| --------------------------- | ----------------------------------------------- | ---------- |
| for item in collection      | for item in IntoIterator::into_iter(collection) | 转移所有权 |
| for item in &collection     | for item in collection.iter()                   | 不可变借用 |
| for item in &mut collection | for item in collection.iter_mut()               | 可变借用   |



如果想在循环中**获取元素的索引**：

```rust
fn main() {
    let a = [4, 3, 2, 1];
    // `.iter()` 方法把 `a` 数组变成一个迭代器
    for (i, v) in a.iter().enumerate() {
        println!("第{}个元素是{}", i + 1, v);
    }
}
```

```rust
for _ in 0..10 {
  // ...
}
```

可以用 `_` 来替代 `i` 用于 `for` 循环中，在 Rust 中 `_` 的含义是忽略该值或者类型的意思，如果不使用 `_`，那么编译器会给你一个 `变量未使用的` 的警告。

以下代码，使用了两种循环方式：

```rust
// 第一种
let collection = [1, 2, 3, 4, 5];
for i in 0..collection.len() {
  let item = collection[i];
  // ...
}

// 第二种
for item in collection {

}
```

第一种方式是循环索引，然后通过索引下标去访问集合，第二种方式是直接循环集合中的元素，优劣如下：

- **性能**：第一种使用方式中 `collection[index]` 的索引访问，会因为边界检查(Bounds Checking)导致运行时的性能损耗 —— Rust 会检查并确认 `index` 是否落在集合内，但是第二种直接迭代的方式就不会触发这种检查，因为编译器会在编译时就完成分析并证明这种访问是合法的
- **安全**：第一种方式里对 `collection` 的索引访问是非连续的，存在一定可能性在两次访问之间，`collection` 发生了变化，**导致脏数据产生**。而第二种直接迭代的方式是**连续访问**，因此不存在这种风险( 由于**所有权限制**，在访问过程中，数据并不会发生变化)。

由于 `for` 循环无需任何条件限制，也不需要通过索引来访问，因此是最安全也是最常用的，通过与下面的 `while` 的对比，我们能看到为什么 `for` 会更加安全。

### continue

使用 `continue` 可以跳过当前当次的循环，开始下次的循环：

```rust
 for i in 1..4 {
     if i == 2 {
         continue;
     }
     println!("{}", i);
 }
```

### break

使用 `break` 可以直接跳出当前整个循环：

```rust
 for i in 1..4 {
     if i == 2 {
         break;
     }
     println!("{}", i);
 }
```

### while

如果你需要一个条件来循环，当该条件为 `true` 时，继续循环，条件为 `false`，跳出循环，那么 `while` 就非常适用：

```rust
fn main() {
    let mut n = 0;

    while n <= 5  {
        println!("{}!", n);

        n = n + 1;
    }

    println!("我出来了！");
}
```

### loop

`loop` 是一个简单的无限循环，你可以在内部实现逻辑通过 `break` 关键字来控制循环何时结束。

```rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    println!("The result is {}", result);
}
```

- **break 可以单独使用，也可以带一个返回值**，有些类似 `return`
- **loop 是一个表达式**，因此可以返回一个值

