+++
title = 'Once'
date = 2024-08-04T21:01:46+08:00

+++

**Once 可以用来执行且仅仅执行一次动作，常常用于单例对象的初始化场景。**

初始化单例资源有很多方法，比如定义 package 级别的变量，这样程序在启动的时候就可以初始化：

```go
package abc

import time

var startTime = time.Now()
```

或者在 init 函数中进行初始化：

```go
package abc

var startTime time.Time

func init() {
  startTime = time.Now()
}

```

又或者在 main 函数开始执行的时候，执行一个初始化的函数：

```go
package abc

var startTime time.Tim

func initApp() {
    startTime = time.Now()
}
func main() {
  initApp()
}
```

这三种方法都是线程安全的，并且后两种方法还可以根据传入的参数实现定制化的初始化操作。

但是很多时候我们是要延迟进行初始化的，所以有时候单例资源的初始化，我们会使用下面的方法：

```go
package main

import (
    "net"
    "sync"
    "time"
)

// 使用互斥锁保证线程(goroutine)安全
var connMu sync.Mutex
var conn net.Conn

func getConn() net.Conn {
    connMu.Lock()
    defer connMu.Unlock()

    // 返回已创建好的连接
    if conn != nil {
        return conn
    }

    // 创建连接
    conn, _ = net.DialTimeout("tcp", "baidu.com:80", 10*time.Second)
    return conn
}

// 使用连接
func main() {
    conn := getConn()
    if conn == nil {
        panic("conn is nil")
    }
}
```

这种方式虽然实现起来简单，但是有性能问题。一旦连接创建好，每次请求的时候还是得竞争锁才能读取到这个连接，这是比较浪费资源的，因为连接如果创建好之后，其实就不需要锁的保护了。怎么办呢？

# Once 的使用场景

**sync.Once 只暴露了一个方法 Do，你可以多次调用 Do 方法，但是只有第一次调用 Do 方法时 f 参数才会执行，这里的 f 是一个无参数无返回值的函数。**

```go
func (o *Once) Do(f func())
```

因为当且仅当第一次调用 Do 方法的时候参数 f 才会执行，即使第二次、第三次、第 n 次调用时 f 参数的值不一样，也不会被执行，比如下面的例子，虽然 f1 和 f2 是不同的函数，但是第二个函数 f2 就不会执行。

```go
package main


import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once

    // 第一个初始化函数
    f1 := func() {
        fmt.Println("in f1")
    }
    once.Do(f1) // 打印出 in f1

    // 第二个初始化函数
    f2 := func() {
        fmt.Println("in f2")
    }
    once.Do(f2) // 无输出
}
```

因为这里的 f 参数是一个无参数无返回的函数，所以你可能会**通过闭包的方式引用外面的参数**，比如：

```go
    var addr = "baidu.com"

    var conn net.Conn
    var err error

    once.Do(func() {
        conn, err = net.Dial("tcp", addr)
    })
```

而且在实际的使用中，**绝大多数情况下，你会使用闭包的方式去初始化外部的一个资源**。

在标准库内部实现中也常常能看到 Once 的身影。比如标准库内部 [cache](https://github.com/golang/go/blob/f0e97546962736fe4aa73b7c7ed590f0134515e1/src/cmd/go/internal/cache/default.go) 的实现上，就使用了 Once 初始化 Cache 资源，包括 defaultDir 值的获取：

```go
    func Default() *Cache { // 获取默认的Cache
    defaultOnce.Do(initDefaultCache) // 初始化cache
    return defaultCache
  }
  
    // 定义一个全局的cache变量，使用Once初始化，所以也定义了一个Once变量
  var (
    defaultOnce  sync.Once
    defaultCache *Cache
  )

    func initDefaultCache() { //初始化cache,也就是Once.Do使用的f函数
    ......
    defaultCache = c
  }

    // 其它一些Once初始化的变量，比如defaultDir
    var (
    defaultDirOnce sync.Once
    defaultDir     string
    defaultDirErr  error
  )
```

除此之外，还有保证只调用一次 copyenv 的 envOnce，strings 包下的 Replacer，time 包中的[测试](https://github.com/golang/go/blob/b71eafbcece175db33acfb205e9090ca99a8f984/src/time/export_test.go#L12)，Go 拉取库时的[proxy](https://github.com/golang/go/blob/8535008765b4fcd5c7dc3fb2b73a856af4d51f9b/src/cmd/go/internal/modfetch/proxy.go#L103)，net.pipe，crc64，Regexp，……，数不胜数。我给你重点介绍一下很值得我们学习的 math/big/sqrt.go 中实现的一个数据结构，它通过 Once 封装了一个只初始化一次的值：

```go
   // 值是3.0或者0.0的一个数据结构
   var threeOnce struct {
    sync.Once
    v *Float
  }
  
    // 返回此数据结构的值，如果还没有初始化为3.0，则初始化
  func three() *Float {
    threeOnce.Do(func() { // 使用Once初始化
      threeOnce.v = NewFloat(3.0)
    })
    return threeOnce.v
  }
```

它将 sync.Once 和 *Float **封装成一个对象**，提供了只初始化一次的值 v。 你看它的 three 方法的实现，虽然每次都调用 threeOnce.Do 方法，但是参数只会被调用一次。

当你使用 Once 的时候，你也可以尝试采用这种结构，**将值和 Once 封装成一个新的数据结构，提供只初始化一次的值。**

**Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。**

# 如何实现一个 Once？

很多人认为实现一个 Once 一样的并发原语很简单，只需使用一个 flag 标记是否初始化过即可，最多是用 atomic 原子操作这个 flag，比如下面的实现

```go
type Once struct {
    done uint32
}

func (o *Once) Do(f func()) {
    if !atomic.CompareAndSwapUint32(&o.done, 0, 1) {
        return
    }
    f()
}
```

这确实是一种实现方式，但是，这个实现有一个很大的问题，就是**如果参数 f 执行很慢的话，后续调用 Do 方法的 goroutine 虽然看到 done 已经设置为执行过了，但是获取某些初始化资源的时候可能会得到空的资源，因为 f 还没有执行完。**

所以，**一个正确的 Once 实现要使用一个互斥锁，这样初始化的时候如果有并发的 goroutine，就会进入doSlow 方法**。互斥锁的机制保证只有一个 goroutine 进行初始化，同时利用**双检查的机制（double-checking）**，再次判断 o.done 是否为 0，如果为 0，则是第一次执行，执行完毕后，就将 o.done 设置为 1，然后释放锁。

即使此时有多个 goroutine 同时进入了 doSlow 方法，因为双检查的机制，后续的 goroutine 会看到 o.done 的值为 1，也不会再次执行 f。

这样既保证了并发的 goroutine 会等待 f 完成，而且还不会多次执行 f。

```go
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}


func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    // 双检查
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

# 易错场景

## 死锁

**Do 方法会执行一次 f，但是如果 f 中再次调用这个 Once 的 Do 方法的话，就会导致死锁的情况出现。**这还不是无限递归的情况，而是的的确确的 Lock 的递归调用导致的死锁。

```go
func main() {
    var once sync.Once
    once.Do(func() {
        once.Do(func() {
            fmt.Println("初始化")
        })
    })
}
```

## 未初始化

**如果 f 方法执行的时候 panic，或者 f 执行初始化资源的时候失败了，这个时候，Once 还是会认为初次执行已经成功了，即使再次调用 Do 方法，也不会再次执行 f。**

比如下面的例子，由于一些防火墙的原因，googleConn 并没有被正确的初始化，后面如果想当然认为既然执行了 Do 方法 googleConn 就已经初始化的话，会抛出空指针的错误：

```go
func main() {
    var once sync.Once
    var googleConn net.Conn // 到Google网站的一个连接

    once.Do(func() {
        // 建立到google.com的连接，有可能因为网络的原因，googleConn并没有建立成功，此时它的值为nil
        googleConn, _ = net.Dial("tcp", "google.com:80")
    })
    // 发送http请求
    googleConn.Write([]byte("GET / HTTP/1.1\r\nHost: google.com\r\n Accept: */*\r\n\r\n"))
    io.Copy(os.Stdout, googleConn)
}
```

既然执行过 Once.Do 方法也可能因为函数执行失败的原因未初始化资源，**并且以后也没机会再次初始化资源**，那么这种**初始化未完成的问题**该怎么解决呢？

我们可以自己**实现一个类似 Once 的并发原语**，**既可以返回当前调用 Do 方法是否正确完成，还可以在初始化失败后调用 Do 方法再次尝试初始化，直到初始化成功才不再初始化了**。

```go
// 一个功能更加强大的Once
type Once struct {
    m    sync.Mutex
    done uint32
}
// 传入的函数f有返回值error，如果初始化失败，需要返回失败的error
// Do方法会把这个error返回给调用者
func (o *Once) Do(f func() error) error {
    if atomic.LoadUint32(&o.done) == 1 { //fast path
        return nil
    }
    return o.slowDo(f)
}
// 如果还没有初始化
func (o *Once) slowDo(f func() error) error {
    o.m.Lock()
    defer o.m.Unlock()
    var err error
    if o.done == 0 { // 双检查，还没有初始化
        err = f()
        if err == nil { // 初始化成功才将标记置为已初始化
            atomic.StoreUint32(&o.done, 1)
        }
    }
    return err
}
```

我们所做的改变就是 Do 方法和参数 f 函数都会返回 error，如果 f 执行失败，会把这个错误信息返回。对 slowDo 方法也做了调整，如果 f 调用失败，我们不会更改 done 字段的值，这样后续的 goroutine 还会继续调用 f。如果 f 执行成功，才会修改 done 的值为 1。

我们怎么查询是否初始化过呢？

目前的 Once 实现可以保证你调用任意次数的 once.Do 方法，它只会执行这个方法一次。但是，有时候我们需要打一个标记。如果初始化后我们就去执行其它的操作，标准库的 Once 并不会告诉你是否初始化完成了，只是让你放心大胆地去执行 Do 方法，所以，你还**需要一个辅助变量，自己去检查是否初始化过了**，比如通过下面的代码中的 inited 字段：

```go
type AnimalStore struct {once   sync.Once;inited uint32}
func (a *AnimalStore) Init() // 可以被并发调用
  a.once.Do(func() {
    longOperationSetupDbOpenFilesQueuesEtc()
    atomic.StoreUint32(&a.inited, 1)
  })
}
func (a *AnimalStore) CountOfCats() (int, error) { // 另外一个goroutine
  if atomic.LoadUint32(&a.inited) == 0 { // 初始化后才会执行真正的业务逻辑
    return 0, NotYetInitedError
  }
        //Real operation
}
```

```go
// Once 是一个扩展的sync.Once类型，提供了一个Done方法
type Once struct {
    sync.Once
}

// Done 返回此Once是否执行过
// 如果执行过则返回true
// 如果没有执行过或者正在执行，返回false
func (o *Once) Done() bool {
    return atomic.LoadUint32((*uint32)(unsafe.Pointer(&o.Once))) == 1
}

func main() {
    var flag Once
    fmt.Println(flag.Done()) //false

    flag.Do(func() {
        time.Sleep(time.Second)
    })

    fmt.Println(flag.Done()) //true
}
```



















