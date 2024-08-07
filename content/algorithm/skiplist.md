---
title: '跳表'
date: 2024-07-04T21:09:55+08:00
params:
  math: true
---

对于一个单链表来讲，即便链表中存储的数据是有序的，如果我们要想在其中查找某个数据，也只能从头到尾遍历链表。这样查找效率就会很低，时间复杂度会很高，是O(n)。

![img.png](/images/algorithm/algo-skiplist-1.png)

那怎么来提高查找效率呢？如果像下图中那样，对链表建立一级“索引”，查找起来是不是就会更快一些呢？每两个结点提取一个结点到上一级，我们把抽出来的那一级叫作**索引或索引层**。下图中的down表示down指针，指向下一级结点。

![img.png](/images/algorithm/algo-skiplist-2.png)

如果我们现在要查找某个结点，比如16。我们可以先在索引层遍历，当遍历到索引层中值为13的结点时，我们发现下一个结点是17，那要查找的结点16肯定就在这两个结点之间。然后我们通过索引层结点的down指针，下降到原始链表这一层，继续遍历。这个时候，我们只需要再遍历2个结点，就可以找到值等于16的这个结点了。这样，原来如果要查找16，需要遍历10个结点，现在只需要遍历7个结点。

加了一层索引之后，查找一个结点需要遍历的结点个数减少了，也就是说查找效率提高了。那如果我们再加一级索引呢？效率会不会提升更多呢？

跟前面建立第一级索引的方式相似，我们在第一级索引的基础之上，每两个结点就抽出一个结点到第二级索引。现在我们再来查找16，只需要遍历6个结点了，需要遍历的结点数量又减少了。

![img.png](/images/algorithm/algo-skiplist-3.png)

为了明显感觉到二级索引的效果，我们提升下数据集大小。

![img.png](/images/algorithm/algo-skiplist-4.png)

从上图中我们可以看出，原来没有索引的时候，查找62需要遍历62个结点，现在只需要遍历11个结点，速度是不是提高了很多？所以，当链表的长度n比较大时，比如1000、10000的时候，在构建索引之后，查找效率的提升就会非常明显。

这种链表加多级索引的结构，就是跳表。现在我们定量地分析一下，用跳表查询到底有多快。

## 时间复杂度

如果链表里有n个结点，会有多少级索引呢？每两个结点会抽出一个结点作为上一级索引的结点，那第一级索引的结点个数大约就是n/2，第二级索引的结点个数大约就是n/4，第三级索引的结点个数大约就是n/8，依次类推，也就是说，第k级索引的结点个数是第k-1级索引的结点个数的1/2，那第k级索引结点的个数就是$n/(2^k)$。

假设索引有h级，最高级的索引有2个结点。通过上面的公式，我们可以得到$n/(2^h)=2$，从而求得$h=\log_2 n-1$。如果包含原始链表这一层，整个跳表的高度就是$\log_2 n$。我们在跳表中查询某个数据的时候，如果每一层都要遍历m个结点，那在跳表中查询一个数据的时间复杂度就是O(m*logn)。

那这个m的值是多少呢？按照前面这种索引结构，我们每一级索引都最多只需要遍历3个结点，也就是说m=3，为什么是3呢？我来解释一下。

假设我们要查找的数据是x，在第k级索引中，我们遍历到y结点之后，发现x大于y，小于后面的结点z，所以我们通过y的down指针，从第k级索引下降到第k-1级索引。在第k-1级索引中，y和z之间只有3个结点（包含y和z），所以，我们在K-1级索引中最多只需要遍历3个结点，依次类推，每一级索引都最多只需要遍历3个结点。

![img.png](/images/algorithm/algo-skiplist-5.png)

通过上面的分析，我们得到m=3，所以在跳表中查询任意数据的**时间复杂度就是O(logn)**。这个查找的时间复杂度跟二分查找是一样的。换句话说，我们其实是基于单链表实现了二分查找。但是这是有代价的，我们建立了多级索引，这就是典型的空间换时间。

那到底需要消耗多少额外的存储空间呢？我们来分析一下跳表的空间复杂度。

## 空间复杂度

跳表的空间复杂度分析并不难，假设原始链表大小为n，那第一级索引大约有n/2个结点，第二级索引大约有n/4个结点，以此类推，每上升一级就减少一半，直到剩下2个结点。如果我们把每层索引的结点数写出来，就是一个等比数列。

![img.png](/images/algorithm/algo-skiplist-6.png)

这几级索引的结点总和就是n/2+n/4+n/8…+8+4+2=n-2。所以，**跳表的空间复杂度是O(n)**。也就是说，如果将包含n个结点的单链表构造成跳表，我们需要额外再用接近n个结点的存储空间。那我们有没有办法降低索引占用的内存空间呢？

我们前面都是每两个结点抽一个结点到上级索引，如果我们每三个结点或五个结点，抽一个结点到上级索引，是不是就不用那么多索引结点了呢？我画了一个每三个结点抽一个的示意图，你可以看下。

![img.png](/images/algorithm/algo-skiplist-7.png)

第一级索引需要大约n/3个结点，第二级索引需要大约n/9个结点。每往上一级，索引结点个数都除以3。为了方便计算，我们假设最高一级的索引结点个数是1。我们把每级索引的结点个数都写下来，也是一个等比数列。

![img.png](/images/algorithm/algo-skiplist-8.png)

通过等比数列求和公式，总的索引结点大约就是n/3+n/9+n/27+…+9+3+1=n/2。**尽管空间复杂度还是O(n)**，但比上面的每两个结点抽一个结点的索引构建方法，**要减少了一半的索引结点存储空间**。

在实际的软件开发中，原始链表中存储的有可能是很大的对象，而索引结点只需要存储关键值和几个指针，并不需要存储对象，所以当对象比索引结点大很多时，那索引占用的额外空间就可以忽略了。

## 高效的插入、删除
实际上，跳表这个动态数据结构，不仅支持查找操作，还支持动态的插入、删除操作，而且**插入、删除操作的时间复杂度也是O(logn)**。

在单链表中，一**旦定位好要插入的位置，插入结点的时间复杂度是很低的，就是O(1)**。但是，这里为了保证原始链表中数据的有序性，我们需要先找到要插入的位置，这个查找操作就会比较耗时。

对于纯粹的单链表，需要遍历每个结点，来找到插入的位置。但是，对于跳表来说，我们讲过查找某个结点的的时间复杂度是O(logn)，所以这里查找某个数据应该插入的位置，方法也是类似的，时间复杂度也是O(logn)。

如果这个结点在索引中也有出现，我们除了要删除原始链表中的结点，还要删除索引中的。因为单链表中的删除操作需要拿到要删除结点的前驱结点，然后通过指针操作完成删除。所以在查找要删除的结点的时候，一定要获取前驱结点。当然，如果我们用的是双向链表，就不需要考虑这个问题了。

## 跳表索引动态更新
当我们不停地往跳表中插入数据时，如果我们不更新索引，就有可能出现某2个索引结点之间数据非常多的情况。极端情况下，跳表还会退化成单链表。

![img.png](/images/algorithm/algo-skiplist-9.png)

作为一种动态数据结构，我们需要某种手段来维护索引与原始链表大小之间的平衡，也就是说，如果链表中结点多了，索引结点就相应地增加一些，避免复杂度退化，以及查找、插入、删除操作性能下降。

跳表是通过**随机函数**来维护二者的“平衡性”。

当我们往跳表中插入数据的时候，我们可以选择**同时将这个数据插入到部分索引层**中。如何选择加入哪些索引层呢？

我们通过一个随机函数，来决定将这个结点插入到哪几级索引中，比如**随机函数生成了值K**，那我们就将这个**结点添加到第一级到第K级这K级索引中**。

![img.png](/images/algorithm/algo-skiplist-10.png)

随机函数的选择很有讲究，从概率上来讲，能够保证跳表的索引大小和数据大小平衡性，不至于性能过度退化。至于随机函数的选择，这里就不多做关注。

## 跳表的Go语言实现
+ https://github.com/ryszard/goskiplist
+ https://github.com/MauriceGit/skiplist
+ https://github.com/sean-public/fast-skiplist
+ https://github.com/sean-public/skiplist-survey
