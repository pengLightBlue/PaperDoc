# 算法就像搭乐高：手撸 LRU 算法

LRU 算法就是一种缓存淘汰策略，原理不难，但是面试中写出没有 bug 的算法比较有技巧，需要对数据结构进行层层抽象和拆解，本文就带你写一手漂亮的代码。

LRU 算法用到的关键数据结构是哈希链表 `LinkedHashMap`，数据结构基础章节的 [手把手带你实现哈希链表](https://labuladong.online/algo/data-structure-basic/hashtable-with-linked-list/) 专门讲解了哈希链表的原理及代码实现。如果你没看过也没关系，本文会再次讲解哈希链表的核心原理，以便实现 LRU 算法。

计算机的缓存容量有限，如果缓存满了就要删除一些内容，给新内容腾位置。但问题是，删除哪些内容呢？我们肯定希望删掉哪些没什么用的缓存，而把有用的数据继续留在缓存里，方便之后继续使用。那么，什么样的数据，我们判定为「有用的」的数据呢？

LRU 缓存淘汰算法就是一种常用策略。LRU 的全称是 Least Recently Used，也就是说我们认为最近使用过的数据应该是是「有用的」，很久都没用过的数据应该是无用的，内存满了就优先删那些很久没用过的数据。

举个简单的例子，安卓手机都可以把软件放到后台运行，比如我先后打开了「设置」「手机管家」「日历」，那么现在他们在后台排列的顺序是这样的：

![](https://labuladong.online/algo/images/lru/1.jpg)

但是这时候如果我访问了一下「设置」界面，那么「设置」就会被提前到第一个，变成这样：

![](https://labuladong.online/algo/images/lru/2.jpg)

假设我的手机只允许我同时开 3 个应用程序，现在已经满了。那么如果我新开了一个应用「时钟」，就必须关闭一个应用为「时钟」腾出一个位置，关那个呢？

按照 LRU 的策略，就关最底下的「手机管家」，因为那是最久未使用的，然后把新开的应用放到最上面：

![](https://labuladong.online/algo/images/lru/3.jpg)

现在你应该理解 LRU（Least Recently Used）策略了。当然还有其他缓存淘汰策略，比如不要按访问的时序来淘汰，而是按访问频率（LFU 策略）来淘汰等等，各有应用场景。本文讲解 LRU 算法策略，我会在 [LFU 算法详解](https://labuladong.online/algo/frequency-interview/lfu/) 中讲解 LFU 算法。

## [一、LRU 算法描述](https://labuladong.online/algo/data-structure/lru-cache/#%E4%B8%80%E3%80%81lru-%E7%AE%97%E6%B3%95%E6%8F%8F%E8%BF%B0)

力扣第 146 题「[LRU缓存机制](https://leetcode.cn/problems/lru-cache/)」就是让你设计数据结构：

首先要接收一个 `capacity` 参数作为缓存的最大容量，然后实现两个 API，一个是 `put(key, val)` 方法存入键值对，另一个是 `get(key)` 方法获取 `key` 对应的 `val`，如果 `key` 不存在则返回 -1。

注意哦，`get` 和 `put` 方法必须都是 O(1)O(1)O(1) 的时间复杂度，我们举个具体例子来看看 LRU 算法怎么工作。

```go
// 创建一个LRU Cache，容量为2
cache := Constructor(2)

// 你可以将cache看作是一个队列
// 假设左边是队头，右边是队尾
// 最近使用的排在队头，久未使用的排在队尾
// 圆括号代表键值对 (key, val)

cache.Put(1, 1)
// cache = [(1, 1)]

cache.Put(2, 2)
// cache = [(2, 2), (1, 1)]

// 返回 1
cache.Get(1)
// cache = [(1, 1), (2, 2)]
// 解释：因为最近访问了键 1，所以将其提前到队头
// 返回键 1 对应的值 1

cache.Put(3, 3)
// cache = [(3, 3), (1, 1)]
// 解释：缓存容量已满，需要删除最久未使用的元素腾出位置
// 优先删除最久未使用的数据，也就是队尾的数据
// 然后将新的数据插入到队头

// 返回 -1 (未找到)
cache.Get(2)
// cache = [(3, 3), (1, 1)]
// 解释：cache 中不存在键为 2 的数据

cache.Put(1, 4)
// cache = [(1, 4), (3, 3)]
// 解释：键 1 已经存在，将原始值 1 覆盖为 4
// 不要忘记还要将键值对提前到队头
```

## [二、LRU 算法设计](https://labuladong.online/algo/data-structure/lru-cache/#%E4%BA%8C%E3%80%81lru-%E7%AE%97%E6%B3%95%E8%AE%BE%E8%AE%A1)

分析上面的操作过程，要让 `put` 和 `get` 方法的时间复杂度为 O(1)，我们可以总结出 `cache` 这个数据结构必要的条件：

1、显然 `cache` 中的元素必须有时序，以区分最近使用的和久未使用的数据，当容量满了之后要删除最久未使用的那个元素腾位置。

2、我们要在 `cache` 中快速找某个 `key` 是否已存在并得到对应的 `val`；

3、每次访问 `cache` 中的某个 `key`，需要将这个元素变为最近使用的，也就是说 `cache` 要支持在任意位置快速插入和删除元素。

那么，什么数据结构同时符合上述条件呢？哈希表查找快，但是数据无固定顺序；链表有顺序之分，插入删除快，但是查找慢。所以结合一下，形成一种新的数据结构：哈希链表 `LinkedHashMap`。

LRU 缓存算法的核心数据结构就是哈希链表，双向链表和哈希表的结合体。这个数据结构长这样：

![](https://labuladong.online/algo/images/lru/4.jpg)

借助这个结构，我们来逐一分析上面的 3 个条件：

1、如果我们每次默认从链表尾部添加元素，那么显然越靠尾部的元素就是最近使用的，越靠头部的元素就是最久未使用的。

2、对于某一个 `key`，我们可以通过哈希表快速定位到链表中的节点，从而取得对应 `val`。

3、链表显然是支持在任意位置快速插入和删除的，改改指针就行。只不过传统的链表无法按照索引快速访问某一个位置的元素，而这里借助哈希表，可以通过 `key` 快速映射到任意一个链表节点，然后进行插入和删除。

**也许读者会问，为什么要是双向链表，单链表行不行？另外，既然哈希表中已经存了 `key`，为什么链表中还要存 `key` 和 `val` 呢，只存 `val` 不就行了**？

想的时候都是问题，只有做的时候才有答案。这样设计的原因，必须等我们亲自实现 LRU 算法之后才能理解，所以我们开始看代码吧～

## [三、代码实现](https://labuladong.online/algo/data-structure/lru-cache/#%E4%B8%89%E3%80%81%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0)

很多编程语言都有内置的哈希链表或者类似 LRU 功能的库函数，但是为了帮大家理解算法的细节，我们先自己造轮子实现一遍 LRU 算法，然后再使用 Java 内置的 `LinkedHashMap` 来实现一遍。

首先，我们把 [双链表](https://labuladong.online/algo/data-structure-basic/linkedlist-basic/) 的节点类写出来，为了简化，`key` 和 `val` 都认为是 int 类型：

```go
type Node struct {
    key, val int
    next, prev *Node
}

func NewNode(k, v int) *Node {
    return &Node{key: k, val: v}
}
```

然后依靠我们的 `Node` 类型构建一个双链表，实现几个 LRU 算法必须的 API：

```go
type DoubleList struct {
    head, tail *Node
    size       int
}

func NewDoubleList() *DoubleList {
    // 初始化双向链表的数据
    head := &Node{key: 0, val: 0}
    tail := &Node{key: 0, val: 0}
    head.next, tail.prev = tail, head
    return &DoubleList{head: head, tail: tail, size: 0}
}

// 在链表尾部添加节点 x，时间 O(1)
func (this *DoubleList) AddLast(x *Node) {
    x.prev = this.tail.prev
    x.next = this.tail
    this.tail.prev.next = x
    this.tail.prev = x
    this.size++
}

// 删除链表中的 x 节点（x 一定存在）
// 由于是双链表且给的是目标 Node 节点，时间 O(1)
func (this *DoubleList) Remove(x *Node) {
    x.prev.next = x.next
    x.next.prev = x.prev
    this.size--
}

// 删除链表中第一个节点，并返回该节点，时间 O(1)
func (this *DoubleList) RemoveFirst() *Node {
    if this.head.next == this.tail {
        return nil
    }
    first := this.head.next
    this.Remove(first)
    return first
}

// 返回链表长度，时间 O(1)
func (this *DoubleList) Size() int {
    return this.size
}
```

如果对链表的操作不熟悉，可以看前文 [手把手带你实现双链表](https://labuladong.online/algo/data-structure-basic/linkedlist-basic/)。

到这里就能回答刚才「为什么必须要用双向链表」的问题了，因为我们需要删除操作。删除一个节点不光要得到该节点本身的指针，也需要操作其前驱节点的指针，而双向链表才能支持直接查找前驱，保证操作的时间复杂度 O(1)。

Important

注意我们实现的双链表 API 只能从尾部插入，也就是说靠尾部的数据是最近使用的，靠头部的数据是最久未使用的。

有了双向链表的实现，我们只需要在 LRU 算法中把它和哈希表结合起来即可，先搭出代码框架：

```go
type LRUCache struct {
    // key -> Node(key, val)
    _map map[int]*Node
    // Node(k1, v1) <-> Node(k2, v2)...
    cache *DoubleList
    // 最大容量
    cap int
}

func Constructor(capacity int) LRUCache {
    return LRUCache{
        map_:  make(map[int]*Node),
        cache: NewDoubleList(),
        cap:   capacity,
    }
}
```

先不慌去实现 LRU 算法的 `get` 和 `put` 方法。由于我们要同时维护一个双链表 `cache` 和一个哈希表 `map`，很容易漏掉一些操作，比如说删除某个 `key` 时，在 `cache` 中删除了对应的 `Node`，但是却忘记在 `map` 中删除 `key`。

**解决这种问题的有效方法是：在这两种数据结构之上提供一层抽象 API**。

就是尽量让 LRU 的主方法 `get` 和 `put` 避免直接操作 `map` 和 `cache` 的细节。我们可以先实现下面几个函数：

```go
// 为了节约篇幅，省略上文给出的代码部分...

// 将某个 key 提升为最近使用的
func (this *LRUCache) makeRecently(key int) {
    x := this._map[key]
    // 先从链表中删除这个节点
    this.cache.Remove(x)
    // 重新插到队尾
    this.cache.AddLast(x)
}

// 添加最近使用的元素
func (this *LRUCache) addRecently(key, val int) {
    x := NewNode(key, val)
    // 链表尾部就是最近使用的元素
    this.cache.AddLast(x)
    // 别忘了在 map 中添加 key 的映射
    this._map[key] = x
}

// 删除某一个 key
func (this *LRUCache) deleteKey(key int) {
    x := this._map[key]
    // 从链表中删除
    this.cache.Remove(x)
    // 从 map 中删除
    delete(this._map, key)
}

// 删除最久未使用的元素
func (this *LRUCache) removeLeastRecently() {
    // 链表头部的第一个元素就是最久未使用的
    deletedNode := this.cache.RemoveFirst()
    // 同时别忘了从 map 中删除它的 key
    deletedKey := deletedNode.key
    delete(this._map, deletedKey)
}
```

这里就能回答之前的问答题「为什么要在链表中同时存储 key 和 val，而不是只存储 val」，注意 `removeLeastRecently` 函数中，我们需要用 `deletedNode` 得到 `deletedKey`。

也就是说，当缓存容量已满，我们不仅仅要删除最后一个 `Node` 节点，还要把 `map` 中映射到该节点的 `key` 同时删除，而这个 `key` 只能由 `Node` 得到。如果 `Node` 结构中只存储 `val`，那么我们就无法得知 `key` 是什么，就无法删除 `map` 中的键，造成错误。

上述方法就是简单的操作封装，调用这些函数可以避免直接操作 `cache` 链表和 `map` 哈希表，下面我先来实现 LRU 算法的 `get` 方法：

```go
// 为了节约篇幅，省略上文给出的代码部分...

func (this *LRUCache) Get(key int) int {
    if _, ok := this._map[key]; !ok {
        return -1
    }
    // 将该数据提升为最近使用的
    this.makeRecently(key)
    return this._map[key].val
}
```

`put` 方法稍微复杂一些，我们先来画个图搞清楚它的逻辑：

![](https://labuladong.online/algo/images/lru/put.jpg)

这样我们可以轻松写出 `put` 方法的代码：

```go
// 为了节约篇幅，省略上文给出的代码部分...

func (this *LRUCache) Put(key, val int) {
    if _, ok := this._map[key]; ok {
        // 删除旧的数据
        this.deleteKey(key)
        // 新插入的数据为最近使用的数据
        this.addRecently(key, val)
        return
    }

    if this.cap == this.cache.Size() {
        // 删除最久未使用的元素
        this.removeLeastRecently()
    }
    // 添加为最近使用的元素
    this.addRecently(key, val)
}
```

至此，你应该已经完全掌握 LRU 算法的原理和实现了。看下完整的实现：

```go
// 双向链表节点
type Node struct {
    key, val int
    next, prev *Node
}

// 双向链表
type DoubleList struct {
    // 头尾虚节点
    head, tail *Node
    // 链表元素数
    size int
}

func NewNode(k, v int) *Node {
    return &Node{key: k, val: v}
}

func NewDoubleList() *DoubleList {
    // 初始化双向链表的数据
    head := NewNode(0, 0)
    tail := NewNode(0, 0)
    head.next = tail
    tail.prev = head
    return &DoubleList{head: head, tail: tail, size: 0}
}

// 在链表尾部添加节点 x，时间 O(1)
func (this *DoubleList) AddLast(x *Node) {
    x.prev = this.tail.prev
    x.next = this.tail
    this.tail.prev.next = x
    this.tail.prev = x
    this.size++
}

// 删除链表中的 x 节点（x 一定存在）
// 由于是双链表且给的是目标 Node 节点，时间 O(1)
func (this *DoubleList) Remove(x *Node) {
    x.prev.next = x.next
    x.next.prev = x.prev
    this.size--
}

// 删除链表中第一个节点，并返回该节点，时间 O(1)
func (this *DoubleList) RemoveFirst() *Node {
    if this.head.next == this.tail {
        return nil
    }
    first := this.head.next
    this.Remove(first)
    return first
}

// 返回链表长度，时间 O(1)
func (this *DoubleList) Size() int {
    return this.size
}

type LRUCache struct {
    // key -> Node(key, val)
    _map map[int]*Node
    // Node(k1, v1) <-> Node(k2, v2)...
    cache *DoubleList
    // 最大容量
    cap int
}

func Constructor(capacity int) LRUCache {
    return LRUCache{
        _map:  make(map[int]*Node),
        cache: NewDoubleList(),
        cap:   capacity,
    }
}

func (this *LRUCache) Get(key int) int {
    if _, ok := this._map[key]; !ok {
        return -1
    }
    // 将该数据提升为最近使用的
    this.makeRecently(key)
    return this._map[key].val
}

func (this *LRUCache) Put(key, val int) {
    if _, ok := this._map[key]; ok {
        // 删除旧的数据
        this.deleteKey(key)
        // 新插入的数据为最近使用的数据
        this.addRecently(key, val)
        return
    }

    if this.cap == this.cache.Size() {
        // 删除最久未使用的元素
        this.removeLeastRecently()
    }
    // 添加为最近使用的元素
    this.addRecently(key, val)
}

func (this *LRUCache) makeRecently(key int) {
    x := this._map[key]
    // 先从链表中删除这个节点
    this.cache.Remove(x)
    // 重新插到队尾
    this.cache.AddLast(x)
}

func (this *LRUCache) addRecently(key, val int) {
    x := NewNode(key, val)
    // 链表尾部就是最近使用的元素
    this.cache.AddLast(x)
    // 别忘了在 map 中添加 key 的映射
    this._map[key] = x
}

func (this *LRUCache) deleteKey(key int) {
    x := this._map[key]
    // 从链表中删除
    this.cache.Remove(x)
    // 从 map 中删除
    delete(this._map, key)
}

func (this *LRUCache) removeLeastRecently() {
    // 链表头部的第一个元素就是最久未使用的
    deletedNode := this.cache.RemoveFirst()
    // 同时别忘了从 map 中删除它的 key
    deletedKey := deletedNode.key
    delete(this._map, deletedKey)
}
```

你也可以用 Java 的内置类型 `LinkedHashMap` 或者 [手把手带你实现哈希链表](https://labuladong.online/algo/data-structure-basic/hashtable-with-linked-list/) 实现的 `MyLinkedHashMap` 来实现 LRU 算法，逻辑和之前完全一致：

```java
class LRUCache {
    int cap;
    LinkedHashMap<Integer, Integer> cache = new LinkedHashMap<>();
    public LRUCache(int capacity) { 
        this.cap = capacity;
    }
    
    public int get(int key) {
        if (!cache.containsKey(key)) {
            return -1;
        }
        // 将 key 变为最近使用
        makeRecently(key);
        return cache.get(key);
    }
    
    public void put(int key, int val) {
        if (cache.containsKey(key)) {
            // 修改 key 的值
            cache.put(key, val);
            // 将 key 变为最近使用
            makeRecently(key);
            return;
        }
        
        if (cache.size() >= this.cap) {
            // 链表头部就是最久未使用的 key
            int oldestKey = cache.keySet().iterator().next();
            cache.remove(oldestKey);
        }
        // 将新的 key 添加链表尾部
        cache.put(key, val);
    }
    
    private void makeRecently(int key) {
        int val = cache.get(key);
        // 删除 key，重新插入到队尾
        cache.remove(key);
        cache.put(key, val);
    }
}
```

至此，LRU 算法就没有什么神秘的了。