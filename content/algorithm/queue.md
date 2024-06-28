+++
title = '队列'
date = 2024-06-28T16:51:51+08:00
+++

队列跟栈一样，也是一种**操作受限的线性表数据结构**。

具有先进先出的特性，支持在队尾插入元素，在队头删除元素

## 顺序队列和链式队列
队列可以用数组来实现，也可以用链表来实现。用数组实现的队列叫作顺序队列， 用链表实现的队列叫作链式队列。

```go
type ArrayQueue struct {
	items    []string
	capacity int
	head     int // 下一个出队的下标
	tail     int // 下一个入队的下标
}

func NewArrayQueue(capacity int) *ArrayQueue {
	return &ArrayQueue{
		items:    make([]string, capacity, capacity),
		capacity: capacity,
		head:     0,
		tail:     0,
	}
}

// 入队
func (q *ArrayQueue) Enqueue(value string) bool {
	if q.tail == q.capacity {
		// 判断是否可以搬迁，集中搬迁
		if q.head == 0 { // 没有空闲空间，不能入队
			return false
		}
		// 所有元素前移 q.head 位
		// 改变 head 以及 tail
		for i := q.head; i < q.tail; i++ {
			q.items[i-q.head] = q.items[i]
		}
		q.tail = q.tail - q.head
		q.head = 0
	}
	q.items[q.tail] = value
	q.tail++
	return true
}

// 出队，出队不做任何数据搬迁
func (q *ArrayQueue) Dequeue() (string, bool) {
	if q.head == q.tail {
		return "", false
	}
	item := q.items[q.head]
	q.head++
	return item, true
}
```
我们在出队时可以不用搬移数据。如果没有空闲空间了，我们只需要在入队时，再**集中触发一次数据的搬移操作**。这种实现思路中，**出队操作的时间复杂度仍然是O(1)**。
![queue-1](/images/algorithm/algo-queue-1.png)

基于链表的实现，我们同样需要两个指针：head指针和tail指针。它们分别指向链表的第一个结点和最后一个结点。如图所示，入队时，tail->next= new_node, tail = tail->next；出队时，head = head->next。

```go
//TODO:CODE
```

![img.png](/images/algorithm/algo-queue-5.png)

## 循环队列

![queue-2](/images/algorithm/algo-queue-2.png)

上图这个队列的大小为8，当前head=4，tail=7。当有一个新的元素a入队时，我们放入下标为7的位置。但这个时候，我们并不把tail更新为8，而是将其在**环中后移一位**，到下标为0的位置。当再有一个元素b入队时，我们将b放入下标为0的位置，然后tail加1更新为1。所以，在a，b依次入队之后，循环队列中的元素就变成了下面的样子：

![queue-3](/images/algorithm/algo-queue-3.png)

要想写出没有bug的循环队列的实现代码，最关键的是，**确定好队空和队满的判定条件**。

在用数组实现的循环队列中，队列为空的判断条件仍然是head == tail。但队列满的判断条件就稍微有点复杂了。

![queue-4](/images/algorithm/algo-queue-4.png)

**当队满时，(tail+1)%n=head。**

**当队列满时，图中的tail指向的位置实际上是没有存储数据的。所以，循环队列会浪费一个数组的存储空间。**

这个空的存储空间就是一个哨兵位，作用如下：

+ 区分队列是满的还是空的。当队列中没有元素时，队头指针和队尾指针都指向哨兵位置。当队列中有元素入队时，队尾指针向前移动，当队尾指针到达队列尾部时，如果队列还有空余位置，队尾指针将回到队列的开头，继续存储元素。当队列中只有一个空位置时，队头指针和队尾指针相遇，此时队列被认为是满的。因此，哨兵位置可以方便地区分队列是满的还是空的。
+ 简化队列的实现。如果没有哨兵位置，在循环队列中需要特殊处理队列是满的情况，否则队列会发生溢出。有了哨兵位置，可以将队列的长度定义为实际存储元素的最大数量加一，这样就可以避免队列溢出的情况，同时也简化了循环队列的实现。

```go
//TODO:CODE
```

## 阻塞队列和并发队列
阻塞队列：在队列为空的时候，从队头取数据会被阻塞。因为此时还没有数据可取，直到队列中有了数据才能返回；如果队列已经满了，那么插入数据的操作就会被阻塞，直到队列中有空闲位置后再插入数据，然后再返回。

![img.png](/images/algorithm/algo-queue-6.png)

![img.png](/images/algorithm/algo-queue-7.png)

在多线程情况下，会有多个线程同时操作队列，这个时候就会存在线程安全问题。线程安全的队列我们叫作并发队列。最简单直接的实现方式是直接在enqueue()、dequeue()方法上加锁，但是锁粒度大并发度会比较低，同一时刻仅允许一个存或者取操作。实际上，**基于数组的循环队列，利用CAS原子操作，可以实现非常高效的并发队列**。这也是循环队列比链式队列应用更加广泛的原因。


