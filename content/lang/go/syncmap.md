+++
title = 'Map'
date = 2024-08-04T21:02:34+08:00

+++

避免 map 并发读写 panic 的方式之一就是加锁，考虑到读写性能，可以使用读写锁提供性能。

## 加读写锁：扩展 map，支持并发读写

```go
type RWMap struct { // 一个读写锁保护的线程安全的map
    sync.RWMutex // 读写锁保护下面的map字段
    m map[int]int
}
// 新建一个RWMap
func NewRWMap(n int) *RWMap {
    return &RWMap{
        m: make(map[int]int, n),
    }
}
func (m *RWMap) Get(k int) (int, bool) { //从map中读取一个值
    m.RLock()
    defer m.RUnlock()
    v, existed := m.m[k] // 在锁的保护下从map中读取
    return v, existed
}

func (m *RWMap) Set(k int, v int) { // 设置一个键值对
    m.Lock()              // 锁保护
    defer m.Unlock()
    m.m[k] = v
}

func (m *RWMap) Delete(k int) { //删除一个键
    m.Lock()                   // 锁保护
    defer m.Unlock()
    delete(m.m, k)
}

func (m *RWMap) Len() int { // map的长度
    m.RLock()   // 锁保护
    defer m.RUnlock()
    return len(m.m)
}

func (m *RWMap) Each(f func(k, v int) bool) { // 遍历map
    m.RLock()             //遍历期间一直持有读锁
    defer m.RUnlock()

    for k, v := range m.m {
        if !f(k, v) {
            return
        }
    }
}
```

## 分片加锁：更高效的并发 map

在并发编程中，我们的一条原则就是尽量减少锁的使用。一些单线程单进程的应用（比如 Redis 等），基本上不需要使用锁去解决并发线程访问的问题，所以可以取得很高的性能。但是对于 Go 开发的应用程序来说，并发是常用的一个特性，在这种情况下，我们能做的就是，**尽量减少锁的粒度和锁的持有时间**。

减少锁的粒度常用的方法就是分片（Shard），将一把锁分成几把锁，每个锁控制一个分片。Go 比较知名的分片并发 map 的实现是[orcaman/concurrent-map](https://github.com/orcaman/concurrent-map)。

它默认采用 32 个分片，**GetShard 是一个关键的方法，能够根据 key 计算出分片索引**。

```go

    var SHARD_COUNT = 32
  
    // 分成SHARD_COUNT个分片的map
  type ConcurrentMap []*ConcurrentMapShared
  
  // 通过RWMutex保护的线程安全的分片，包含一个map
  type ConcurrentMapShared struct {
    items        map[string]interface{}
    sync.RWMutex // Read Write mutex, guards access to internal map.
  }
  
  // 创建并发map
  func New() ConcurrentMap {
    m := make(ConcurrentMap, SHARD_COUNT)
    for i := 0; i < SHARD_COUNT; i++ {
      m[i] = &ConcurrentMapShared{items: make(map[string]interface{})}
    }
    return m
  }
  

  // 根据key计算分片索引
  func (m ConcurrentMap) GetShard(key string) *ConcurrentMapShared {
      // 使用 fnv32 哈希函数将 key 转换为一个 32 位的无符号整数
      // 将哈希值对 SHARD_COUNT 取模，得到分片索引
    return m[uint(fnv32(key))%uint(SHARD_COUNT)]
  }
```

增加或者查询的时候，首先根据分片索引得到分片对象，然后对分片对象加锁进行操作：

```go
func (m ConcurrentMap) Set(key string, value interface{}) {
    // 根据key计算出对应的分片
    shard := m.GetShard(key)
    shard.Lock() //对这个分片加锁，执行业务操作
    shard.items[key] = value
    shard.Unlock()
}

func (m ConcurrentMap) Get(key string) (interface{}, bool) {
    // 根据key计算出对应的分片
    shard := m.GetShard(key)
    shard.RLock()
    // 从这个分片读取key的值
    val, ok := shard.items[key]
    shard.RUnlock()
    return val, ok
}
```

解决 map 并发 panic 的两个方法：**加锁和分片**。

**在使用并发 map 的过程中，加锁和分片加锁这两种方案都比较常用，如果是追求更高的性能，显然是分片加锁更好，因为它可以降低锁的粒度，进而提高访问此 map 对象的吞吐。如果并发性能要求不是那么高的场景，简单加锁方式更简单。**

## sync.Map

 sync.Map 是 Go 官方线程安全 map 的标准实现。虽然是官方标准，反而是不常用的，为什么呢？一句话来说就是 map 要解决的场景很难描述，很多时候在做抉择时根本就不知道该不该用它。但是呢，确实有一些**特定的场景**，我们需要用到 sync.Map 来实现，所以还是很有必要学习这个知识点。

那这些特殊的场景是啥呢？[官方的文档](https://golang.org/pkg/sync/#Map)中指出，在以下两个场景中使用 sync.Map，会比使用 map+RWMutex 的方式，性能要好得多：

+ **只会增长**的缓存系统中，一个 key **只写入一次而被读很多次**；
+ **多个 goroutine 为不相交**的键集读、写和重写键值对。

这两个场景说得都比较笼统，而且，这些场景中还包含了一些特殊的情况。所以，官方建议你针对自己的场景做**性能评测**，如果确实能够显著提高性能，再使用 sync.Map。

### Map 的实现

sync.Map 的实现有几个优化点:

+ **空间换时间**。通过**冗余**的两个数据结构（**只读的 read 字段、可写的 dirty**），来减少加锁对性能的影响。对只读字段（read）的操作不需要加锁。
+ 优先从 read 字段读取、更新、删除，因为对 read 字段的读取不需要锁。
+ **动态调整**。miss 次数多了之后，将 dirty 数据提升为 read，**避免总是从 dirty 中加锁读取**。
+ **double-checking**。加锁之后先还要再检查 read 字段，确定真的不存在才操作 dirty 字段。
+ **延迟删除**。**删除一个键值只是打标记**，只有在提升 dirty 字段为 read 字段的时候才清理删除的数据。

```go
type Map struct {
    mu Mutex
    // 基本上你可以把它看成一个安全的只读的map
    // 它包含的元素其实也是通过原子操作更新的，但是已删除的entry就需要加锁操作了
    read atomic.Value // readOnly

    // 包含需要加锁才能访问的元素
    // 包括所有在read字段中但未被expunged（删除）的元素以及新加的元素
    dirty map[interface{}]*entry

    // 记录从read中读取miss的次数，一旦miss数和dirty长度一样了，就会把dirty提升为read，并把dirty置空
    misses int
}

type readOnly struct {
    m       map[interface{}]*entry
    amended bool // 当dirty中包含read没有的数据时为true，比如新增一条数据
}

// expunged是用来标识此项已经删掉的指针
// 当map中的一个项目被删除了，只是把它的值标记为expunged，以后才有机会真正删除此项
var expunged = unsafe.Pointer(new(interface{}))

// entry代表一个值
type entry struct {
    p unsafe.Pointer // *interface{}
}
```

如果 dirty 字段非 nil 的话，map 的 read 字段和 dirty 字段会包含相同的非 expunged 的项，所以如果通过 read 字段更改了这个项的值，从 dirty 字段中也会读取到这个项的新值，因为本来它们指向的就是同一个地址。

dirty 包含重复项目的好处就是，一旦 miss 数达到阈值需要将 dirty 提升为 read 的话，只需简单地把 dirty 设置为 read 对象即可。不好的一点就是，当创建新的 dirty 对象的时候，需要逐条遍历 read，把非 expunged 的项复制到 dirty 对象中。

Store、Load 和 Delete 这三个核心函数的操作都是先从 read 字段中处理的，因为读取 read 字段的时候不用加锁。

### Store

Store方法是用来设置一个键值对，或者更新一个键值对的。

```go
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    // 如果read字段包含这个项，说明是更新，cas更新项目的值即可
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // read中不存在，或者cas更新失败，就需要加锁访问dirty了
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    if e, ok := read.m[key]; ok { // 双检查，看看read是否已经存在了
        if e.unexpungeLocked() {
            // 此项目先前已经被删除了，通过将它的值设置为nil，标记为unexpunged
            m.dirty[key] = e
        }
        e.storeLocked(&value) // 更新
    } else if e, ok := m.dirty[key]; ok { // 如果dirty中有此项
        e.storeLocked(&value) // 直接更新
    } else { // 否则就是一个新的key
        if !read.amended { //如果dirty为nil
            // 需要创建dirty对象，并且标记read的amended为true,
            // 说明有元素它不包含而dirty包含
            m.dirtyLocked()
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        m.dirty[key] = newEntry(value) //将新值增加到dirty对象中
    }
    m.mu.Unlock()
}
```

可以看出，Store 既可以是新增元素，也可以是更新元素。如果运气好的话，更新的是已存在的未被删除的元素，直接更新即可，不会用到锁。如果运气不好，需要更新（重用）删除的对象、更新还未提升的 dirty 中的对象，或者新增加元素的时候就会使用到了锁，这个时候，性能就会下降。

所以从这一点来看，**sync.Map 适合那些只会增长的缓存系统，可以进行更新，但是不要删除，并且不要频繁地增加新元素**。

新加的元素需要放入到 dirty 中，如果 dirty 为 nil，那么需要从 read 字段中复制出来一个 dirty 对象：

```go
func (m *Map) dirtyLocked() {
    if m.dirty != nil { // 如果dirty字段已经存在，不需要创建了
        return
    }

    read, _ := m.read.Load().(readOnly) // 获取read字段
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m { // 遍历read字段
        if !e.tryExpungeLocked() { // 把非punged的键值对复制到dirty中
            m.dirty[k] = e
        }
    }
}
```

### Load

Load 方法用来读取一个 key 对应的值。它也是从 read 开始处理，一开始并不需要锁。

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 首先从read处理
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended { // 如果不存在并且dirty不为nil(有新的元素)
        m.mu.Lock()
        // 双检查，看看read中现在是否存在此key
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {//依然不存在，并且dirty不为nil
            e, ok = m.dirty[key]// 从dirty中读取
            // 不管dirty中存不存在，miss数都加1
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if !ok {
        return nil, false
    }
    return e.load() //返回读取的对象，e既可能是从read中获得的，也可能是从dirty中获得的
}
```

如果幸运的话，我们从 read 中读取到了这个 key 对应的值，那么就不需要加锁了，性能会非常好。但是，如果请求的 key 不存在或者是新加的，就需要加锁从 dirty 中读取。所以，读取不存在的 key 会因为加锁而导致性能下降，读取还没有提升的新值的情况下也会因为加锁性能下降。

其中，missLocked 增加 miss 的时候，如果 miss 数等于 dirty 长度，会将 dirty 提升为 read，并将 dirty 置空。

```go
func (m *Map) missLocked() {
    m.misses++ // misses计数加一
    if m.misses < len(m.dirty) { // 如果没达到阈值(dirty字段的长度),返回
        return
    }
    m.read.Store(readOnly{m: m.dirty}) //把dirty字段的内存提升为read字段
    m.dirty = nil // 清空dirty
    m.misses = 0  // misses数重置为0
}
```

### Delete

Delete 方法的核心在 LoadAndDelete 中实现了。

```go
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        // 双检查
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // 这一行长坤在1.15中实现的时候忘记加上了，导致在特殊的场景下有些key总是没有被回收
            delete(m.dirty, key)
            // miss数加1
            m.missLocked()
        }
        m.mu.Unlock()
    }
    if ok {
        return e.delete()
    }
    return nil, false
}

func (m *Map) Delete(key interface{}) {
    m.LoadAndDelete(key)
}
func (e *entry) delete() (value interface{}, ok bool) {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == nil || p == expunged {
            return nil, false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, nil) {
            return *(*interface{})(p), true
        }
    }
}
```

如果 read 中不存在，那么就需要从 dirty 中寻找这个项目。最终，如果项目存在就删除（将它的值标记为 nil）。如果项目不为 nil 或者没有被标记为 expunged，那么还可以把它的值返回。

最后，我补充一点，sync.map 还有一些 LoadAndDelete、LoadOrStore、Range 等辅助方法，但是没有 Len 这样查询 sync.Map 的包含项目数量的方法，并且官方也不准备提供。如果你想得到 sync.Map 的项目数量的话，你可能不得不通过 Range 逐个计数。

当然还有一些扩展其它功能的 map 实现，比如带有过期功能的[timedmap](https://github.com/zekroTJA/timedmap)、使用红黑树实现的 key 有序的[treemap](https://godoc.org/github.com/emirpasic/gods/maps/treemap)等，因为和并发问题没有关系。
