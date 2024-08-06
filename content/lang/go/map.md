+++
title = '映射'
date = 2024-04-27T10:54:38+08:00

+++

在官方标准编译器和运行时中，映射是使用哈希表算法来实现的。所以**一个映射中的所有元素也均存放在一块连续的内存中，但是映射中的元素并不一定紧挨着存放**。 另外一种常用的映射实现算法是二叉树算法。无论使用何种算法，**一个映射中的所有元素的键值也存放在此映射值（的间接部分）中**。

即一个映射也是由一个**直接部分**和一个可能的被此直接部分引用着的**间接部分**组成。

映射元素访问消耗的时长要**数倍**于数组和切片元素访问消耗的时长。 但是映射相对于数组和切片有两个优点：

+ 映射的键值类型可以是任何可比较类型。
+ **对于大多数元素为零值的情况，使用映射可以节省大量的内存**。

> Q: 为什么映射可以节省大量的内存。

## 数据结构
```go
// runtime/map.go/hmap
type hmap struct {
    count     int // 当前保存的元素个数
	flags     uint8
	B         uint8  // 指示bucket数组的大小
	noverflow uint16
    hash0     uint32
    
    buckets    unsafe.Pointer // bucket数组指针，数组的大小为2^B
    oldbuckets unsafe.Pointer
	nevacuate  uintptr

	extra *mapextra
}

type mapextra struct {
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    nextOverflow *bmap
}
```
+ count 表示当前哈希表中的元素数量；
+ B 表示当前哈希表持有的 buckets 数量，但是因为哈希表中桶的数量都 2 的倍数，所以该字段会存储对数，也就是 len(buckets) == 2^B；
+ hash0 是哈希的种子，它能为哈希函数的结果引入随机性，这个值在创建哈希表时确定，并在调用哈希函数时作为参数传入；
+ oldbuckets 是哈希在扩容时用于保存之前 buckets 的字段，它的大小是当前 buckets 的一半；

**所有映射类型的尺寸也都是一致的（一个指针大小）**。映射类型属于**不可比较类型**，任意两个映射值是不能相互比较的。但是一个映射值可以和预声明的`nil`标识符进行比较以检查此映射值是否为一个零值。


![img.png](/images/lang/go/map-1.png)

如上图所示哈希表 runtime.hmap 的桶是 runtime.bmap。每一个 runtime.bmap 都能存储 8 个键值对，当哈希表中存储的数据过多，单个桶已经装满时就会使用 extra.nextOverflow 中桶存储溢出的数据。

上述两种不同的桶在内存中是连续存储的，我们在这里将它们分别称为正常桶和溢出桶，上图中黄色的 runtime.bmap 就是正常桶，绿色的 runtime.bmap 是溢出桶，溢出桶是在 Go 语言还使用 C 语言实现时使用的设计3，由于它能够减少扩容的频率所以一直使用至今。

桶的结构体 runtime.bmap ：

```go
type bmap struct {
    tophash [8]uint8 //存储哈希值的高8位
    data    byte[1]  //key value数据:key/key/key/.../value/value/value...
    overflow *bmap   //溢出bucket的地址
}
```
每个bucket可以存储8个键值对。
+ tophash是个长度为8的数组，哈希值相同的键（准确的说是哈希值低位相同的键）存入当前bucket时会将哈希值的高位存储在该数组中，以方便后续匹配以减少访问键值对次数以提高性能。
+ data区存放的是key-value数据，存放顺序是key/key/key/...value/value/value，如此存放是为了节省字节对齐带来的空间浪费。
+ overflow 指针指向的是下一个bucket，据此将所有冲突的键连接起来。

注意：上述中data和overflow**并不是在结构体中显示定义的，而是直接通过指针运算进行访问的**。

随着哈希表存储的数据逐渐增多，我们会扩容哈希表或者使用额外的桶存储溢出的数据，不会让单个桶中的数据超过 8 个，不过溢出桶只是临时的解决方案，创建过多的溢出桶最终也会导致哈希的扩容。

下图展示bucket存放8个key-value对：


![img.png](/images/lang/go/map-2.png)


## 声明及初始化
```go
var m map[string]string  // m = nil
m["name"] = "yogurt" // panic
```
不同于切片的nil， 初值为 nil 的 map 无法“零值可用“。必须对 map 类型变量进行显式初始化后才能使用。

**一个映射类型的键值类型必须为一个可比较类型**。一个映射值类型的容器值中的元素关联的键值可以是任何此映射类型的**键值类型的任何值**。

这里有一点需要注意，如果使用 struct 类型做 key 其实是有坑的，因为**如果 struct 的某个字段值修改了，查询 map 时无法获取它 add 进去的值**，如下面的例子：

```go
type mapKey struct {
    key int
}

func main() {
    var m = make(map[mapKey]string)
    var key = mapKey{10}


    m[key] = "hello"
    fmt.Printf("m[key]=%s\n", m[key])


    // 修改key的字段的值后再次查询map，无法获取刚才add进去的值
    key.key = 100
    fmt.Printf("再次查询m[key]=%s\n", m[key])
}
```

如果要使用 struct 作为 key，我们要保证 struct 对象在**逻辑上是不可变的**，这样才会保证 map 的逻辑没有问题。

**map 是无序的，所以当遍历一个 map 对象的时候，迭代的元素的顺序是不确定的，无法保证两次遍历的顺序是一样的，也不能保证和插入的顺序一致**。

### 复合字面值
```go
m := map[int]string{}
```
我们显式初始化了 map 类型变量 m。不过，你要注意，虽然此时 map 类型变量 m 中没有任何键值对，但变量 m 也不等同于初值为 nil 的 map 变量。这个时候，我们对 m 进行键值对的插入操作，不会引发运行时异常。

```go
hash := map[string]int{
	"1": 2,
	"3": 4,
	"5": 6,
}
```
map 组合字面量中元素对应的键值不可缺失，并且它们**可以为非常量**。

```go
var a uint = 1
var _ = map[uint]int {a : 123} // 没问题
```

> Q: 如果改变了这个变量的值，还能取到value？
>
> A: 

一个容器（array slice map）**组合字面量**中**常量键值**（包括索引下标）是**不能重复**的，

```go
// error: 重复的索引：1
var a = []bool{0: false, 1: true, 1: true}
// error: 重复的索引：0
var b = [...]string{0: "foo", 1: "bar", 0: "foo"}
// error: 重复的键值："foo"
var c = map[string]int{"foo": 1, "foo": 2}
```

> Q: 那么map的组合字面量是可以为变量的，map变量重复的话，会有什么影响？



`map[K]T{}`和`map[K]T(nil)`是不等价的。

我们需要在初始化哈希时声明键值对的类型，这种使用字面量初始化的方式最终都会通过 `cmd/compile/internal/gc.maplit` 初始化，我们来分析一下该函数初始化哈希的过程：

```go
func maplit(n *Node, m *Node, init *Nodes) {
	a := nod(OMAKE, nil, nil)
	a.Esc = n.Esc
	a.List.Set2(typenod(n.Type), nodintconst(int64(n.List.Len())))
	litas(m, a, init)

	entries := n.List.Slice()
	if len(entries) > 25 {
		...
		return
	}

	// Build list of var[c] = expr.
	// Use temporaries so that mapassign1 can have addressable key, elem.
	...
}
```

当哈希表中的元素数量少于或者等于 25 个时，编译器会将字面量初始化的结构体转换成以下的代码，**将所有的键值对一次加入到哈希表中**：

```go
hash := make(map[string]int, 3)
hash["1"] = 2
hash["3"] = 4
hash["5"] = 6
```

这种初始化的方式与的数组和切片几乎完全相同，由此看来集合类型的初始化在 Go 语言中有着相同的处理逻辑。

一旦哈希表中元素的数量超过了 25 个，编译器会创建两个数组分别存储键和值，这些键值对会通过如下所示的 for 循环加入哈希：

```go
hash := make(map[string]int, 26)
vstatk := []string{"1", "2", "3", ... ， "26"}
vstatv := []int{1, 2, 3, ... , 26}
for i := 0; i < len(vstak); i++ {
    hash[vstatk[i]] = vstatv[i]
}
```

不过无论使用哪种方法，使用字面量初始化的过程都会使用 Go 语言中的关键字 make 来创建新的哈希并通过最原始的 [] 语法向哈希追加元素。

### make 关键字
```go
m1 := make(map[int]string) // 未指定初始容量
m2 := make(map[int]string, 8) // 指定初始容量为8
```
第一个函数调用形式创建了一个可以容纳一个小数目的条目而无需再次开辟内存的空映射值。此**小数目的值取决于具体编译器实现**。第二个函数调用形式创建了一个可以容纳至少`n`个条目而无需再次开辟内存的空映射值。 

注意：第二个参数`n`可以为负或者零，这时对应的调用将被视为上述第一种调用形式。

map 类型的容量不会受限于它的初始容量值，当其中的键值对数量超过初始容量后，Go 运行时会自动增加 map 类型的容量，保证后续键值对的正常插入。

当创建的哈希被分配到栈上并且其容量小于 `BUCKETSIZE = 8` 时，Go 语言在编译阶段会使用如下方式快速初始化哈希，这也是编译器对小容量的哈希做的优化：
```go
var h *hmap
var hv hmap
var bv bmap
h := &hv
b := &bv
h.buckets = b
h.hash0 = fashtrand0()
```

除了上述特定的优化之外，无论 make 是从哪里来的，只要我们使用 make 创建哈希，Go 语言编译器都会在**类型检查**期间将它们转换成 `runtime.makemap`，使用字面量初始化哈希也只是语言提供的辅助工具，最后调用的都是 `runtime.makemap`：
```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
	return h
}
```
1. 计算哈希占用的内存是否溢出或者超出能分配的最大值；
2. 调用 runtime.fastrand 获取一个随机的哈希种子； 
3. 根据传入的 hint 计算出需要的最小需要的桶的数量； 
4. 使用 runtime.makeBucketArray 创建用于保存桶的数组；

runtime.makeBucketArray 会根据传入的 B 计算出的需要创建的桶数量并在内存中分配**一片连续的空间用于存储数据**：
```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	if b >= 4 {
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
+ 当桶的数量小于 24 时，由于数据较少、使用溢出桶的可能性较低，会省略创建的过程以减少额外开销； 
+ 当桶的数量多于 24 时，会额外创建 2^(B−4) 个溢出桶；

根据上述代码，我们能确定在正常情况下，正常桶和溢出桶在内存中的存储空间是连续的，只是被 runtime.hmap 中的不同字段引用。


## 地址
同切片一样，map也是由直接值部和间接值部组成的，所以map值的地址不可能等于某个key值对应的地址。

## 添加和删除容器元素

向一个映射中添加一个条目的语法和修改一个映射元素的语法是一样的。 比如，对于一个非零映射值`m`，如果当前`m`中尚未存储条目`(k, e)`，则下面的语法形式将把此条目存入`m`；否则，下面的语法形式将把键值`k`对应的元素值更新为`e`。

```go
m[k] = e
```

内置函数`delete`用来从一个映射中删除一个条目。比如，下面的`delete`调用将把键值`k`对应的条目从映射`m`中删除。 如果映射`m`中未存储键值为`k`的条目，则此调用为一个空操作，它**不会产生一个panic，即使`m`是一个nil零值映射**。

```go
delete(m, k)
```

注意，在Go 1.12之前，映射打印结果中的条目顺序并不固定，两次打印结果可能并不相同。

## 赋值
在 Go 语言中所有的**赋值操作均是**源值被复制给了目标值。精确地说，**源值的直接部分复制**给了目标值。 map 也不例外。

**在任何赋值中，源值的底层间接部分不会被复制。 换句话说，当一个赋值结束后，一个含有间接部分的源值和目标值将共享底层间接部分**。 

**当一个映射赋值语句执行完毕之后，目标映射值和源映射值将共享底层的元素**。 向其中一个映射中添加（或从中删除）元素将体现在另一个映射中。

## 哈希冲突
当有两个或以上数量的键被哈希到了同一个bucket时，我们称这些键发生了冲突。Go使用 **链地址法** 来解决键冲突。 由于每个bucket可以存放8个键值对，所以同一个bucket存放超过8个键值对时就会再创建一个键值对，用类似链表的方式将bucket连接起来。

下图展示产生冲突后的map：


![img.png](/images/lang/go/map-3.png)


bucket数据结构指示下一个bucket的指针称为overflow bucket，意为当前bucket盛不下而溢出的部分。事实上哈希冲突并不是好事情，它降低了存取效率，好的哈希算法可以保证哈希值的随机性，但冲突过多也是要控制的。

## 负载因子
负载因子用于衡量一个哈希表冲突情况，公式为：

> 负载因子 = 键数量/bucket数量

例如，对于一个bucket数量为4，包含4个键值对的哈希表来说，这个哈希表的负载因子为1.

哈希表需要将负载因子控制在合适的大小，超过其阀值需要进行rehash，也即键值对重新组织：
+ 哈希因子过小，说明空间利用率低
+ 哈希因子过大，说明冲突严重，存取效率低

每个哈希表的实现对负载因子容忍程度不同，比如Redis实现中负载因子大于1时就会触发rehash，而Go则在在负载因子达到6.5时才会触发rehash，因为Redis的每个bucket只能存1个键值对，而Go的bucket可能存8个键值对，所以Go可以容忍更高的负载因子。



