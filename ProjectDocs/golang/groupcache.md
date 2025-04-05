# Go 语言分布式缓存 Groupcache – 用法，源码深度解读

> [groupcache](https://github.com/golang/groupcache)是memcached作者Brad Fitzpatrick的另一kv cache项目，由于作者转 go 并对go语言情有独钟，所以作者本着‘intended as a replacement for memcached in many cases’ 的设计初衷设计了groupcache 。Redis等其他常用cache实现不同，groupcache并不运行在单独的server上，而是作为library和app运行在同一进程中。所以groupcache既是server也是client。

## 为什用，怎么用？

还是老样子，我们先搞清楚为什么要用Groupcache，

Groupcache是一个轻量级的分布式缓存库，他的逻辑非常简单所以保证了他的轻量，但它却可以满足各种缓存中间件的功能，如防击穿，防穿透，防雪崩。我们之后会通过源码来解释他是如何做到这些的。同时因为其简单的逻辑保证其在消耗极少资源的情况下保持高性能。

有利一定会有弊，他在应用上会受一些限制，接下来我们讨论怎么用比较好。

要搞清楚这一点我们需要看下它的设计思路，groupcache 其实就是一个小巧的 kv 存储库，他的特点是：

**只增不删改**，他没有设计让你通过外部调用来删改的接口。你可能觉得这样只能增加的设计很不灵活，那作者为什么不设计的更加灵活？答案是**一致性**。一个分布式的缓存系统需要强一致性，作者使用只增不删改的方式来在不牺牲性能的基础上保持了一致性。groupcache 就是作用在一些高频访问固定数据上，而这种需求也是我们最常见的。如果一个东西你需要改动你完全可以把不变的东西抽离出来引入 groupcache。

例子1：热点快讯，出现之时他就会固定，而且是将会被高频访问。

例子2：缓存高频访问的静态文件。

我们的用例：我们用 groupcache 主要是存储资源固定信息，因为这些信息是提供给多个服务的，所以这些信息访问频率非常高，很难直接调用数据库。而且我们的资源是增量的使用redis这种缓存服务必然会出现容量上限或是并发量过大导致的击穿等情况。

## 主体逻辑

Groupcache 的逻辑思路很简单，我简单的总结为三种获取方式

1. 通过本地缓存获取
2. 通过对应节点缓存获取
3. 通过已定义好的方法获取

以上三种获取资源方式就是该包的核心逻辑，三种获取方式优先级为从上到下：

**查询逻辑**：groupcahce 服务首先查询自己的本地缓存自己本地缓存无法找到再从周边节点搜寻，周边节点的缓存找不到就会调用你已经定义好的资源获取方法来获取资源。

**缓存逻辑**：如果是通过通过本地缓存获取到资源则无需缓存处理，如果是通过其他节点获取到的资源则会缓存到本地的hotcache，如果是通过已定义的方法获取的资源则会缓存到本地的maincache。

下图为一个我从网上找逻辑描述图：

[![groupcache_code](https://marksuper.xyz/picture/file/groupcache/groupcache_code.png)](https://marksuper.xyz/picture/file/groupcache/groupcache_code.png)

可以看到该图简单的描述了我刚刚叙述的groupcache核心逻辑。看了这个核心逻辑你是否可以自己构思一下代码的架构。

## 代码架构

要弄清代码架构，我们先来看下目录结构：

[![image-20220825130947414](https://marksuper.xyz/picture/file/groupcache/groupcache_menu.png)](https://marksuper.xyz/picture/file/groupcache/groupcache_menu.png)

可以看到主目录下有 byteview.go，groupcache.go，http.go，peer.go，sinks.go 这几个文件其实这几个文件的中对应的接口或结构就构成了该代码的主体架构。代码的主体是 groupcache.go 文件中的 Group 结构该结构中包含了节点获取接口 peers，自定义获取资源方法 getter，存储结构mainCache，hotCache，同时并发执行单次结构 loadGroup。

```go
// A Group is a cache namespace and associated data loaded spread over
// a group of 1 or more machines.
type Group struct {
	name       string
	getter     Getter
	peersOnce  sync.Once
	peers      PeerPicker
	cacheBytes int64 // limit for sum of mainCache and hotCache size

	// mainCache is a cache of the keys for which this process
	// (amongst its peers) is authoritative. That is, this cache
	// contains keys which consistent hash on to this process's
	// peer number.
	mainCache cache

	// hotCache contains keys/values for which this peer is not
	// authoritative (otherwise they would be in mainCache), but
	// are popular enough to warrant mirroring in this process to
	// avoid going over the network to fetch from a peer.  Having
	// a hotCache avoids network hotspotting, where a peer's
	// network card could become the bottleneck on a popular key.
	// This cache is used sparingly to maximize the total number
	// of key/value pairs that can be stored globally.
	hotCache cache

	// loadGroup ensures that each key is only fetched once
	// (either locally or remotely), regardless of the number of
	// concurrent callers.
	loadGroup flightGroup

	_ int32 // force Stats to be 8-byte aligned on 32-bit platforms

	// Stats are statistics on the group.
	Stats Stats
}
```

peer.go 定义注册节点选择器的方法，在第一次 group get 时会使用节点注册方法进行一次注册。

byteview.go 定义了基础存储结构，存储结构是groupcache的重要结构下面会单独展开说。

sink.go 中定义资源获取结构。

http.go 中只包含了一个HTTPPool这样的一个结构，但我认为这个文件结构才是理解整体的关键。

这个结构定义了实现分布式的所有方法，如果你是单体部署那你非常好理解这个包，其实就是从 group 结构中取资源，取到后在本地缓存。说白了基本是不需要这个包来实现这个逻辑。所以分布式才是这个包的灵魂。

这个结构定义了远程节点的注册，选取方法。以及作为服务端获取本地资源的方法和作为客户端获取远程节点资源的方法。所以这个HTTPPool结构我们会去重点说。

consistenthash文件夹 中定义Map结构用于节点选取。

lru文件夹 中定义的Cache结构是缓存的主要结构。

singleflight文件夹 中定义了单个资源同时获取只执行一次的获取逻辑。

好了，代码的架构我感觉我说的很清楚，接下来我们就从分Group主体，缓存，分布式三个重点来进行源码解读。

## 源码解读

### Group主体

上面我们看了group主体的代码，描述了主体中的重要结构。

整个groupcache的主体逻辑可以说是就一个Get方法，所以我们先看这个方法，来看作者是如果实现根据不同优先级来执行三种获取方法并缓存数据的逻辑。：

```
func (g *Group) initPeers() {
	if g.peers == nil {
		g.peers = getPeers(g.name)
	}
}

func (g *Group) Get(ctx context.Context, key string, dest Sink) error {
	g.peersOnce.Do(g.initPeers)
	g.Stats.Gets.Add(1)
	if dest == nil {
		return errors.New("groupcache: nil dest Sink")
	}
	value, cacheHit := g.lookupCache(key)

	if cacheHit {
		g.Stats.CacheHits.Add(1)
		return setSinkView(dest, value)
	}

	// Optimization to avoid double unmarshalling or copying: keep
	// track of whether the dest was already populated. One caller
	// (if local) will set this; the losers will not. The common
	// case will likely be one caller.
	destPopulated := false
	value, destPopulated, err := g.load(ctx, key, dest)
	if err != nil {
		return err
	}
	if destPopulated {
		return nil
	}
	return setSinkView(dest, value)
}
```

Get方法先调用了initPeer，这个方法是用你自己注册的一个registerPeer的方法来获取peers，并赋值到group上。注册peer的方法我会在分布式模块中说。

然后调用了g.lookupCache

```
go
func (g *Group) lookupCache(key string) (value ByteView, ok bool) {
	if g.cacheBytes <= 0 {
		return
	}
	value, ok = g.mainCache.get(key)
	if ok {
		return
	}
	value, ok = g.hotCache.get(key)
	return
}
```

这个方法其实就是从mainCache和hotCache也就是本地缓存中尝试获取资源。取到了就把值放入获取数据结构dest中。这就是我们说的从本地获取。

获取不到就继续调用g.load方法：

```
// load loads key either by invoking the getter locally or by sending it to another machine.
func (g *Group) load(ctx context.Context, key string, dest Sink) (value ByteView, destPopulated bool, err error) {
	g.Stats.Loads.Add(1)
	viewi, err := g.loadGroup.Do(key, func() (interface{}, error) {
		// Check the cache again because singleflight can only dedup calls
		// that overlap concurrently.  It's possible for 2 concurrent
		// requests to miss the cache, resulting in 2 load() calls.  An
		// unfortunate goroutine scheduling would result in this callback
		// being run twice, serially.  If we don't check the cache again,
		// cache.nbytes would be incremented below even though there will
		// be only one entry for this key.
		//
		// Consider the following serialized event ordering for two
		// goroutines in which this callback gets called twice for the
		// same key:
		// 1: Get("key")
		// 2: Get("key")
		// 1: lookupCache("key")
		// 2: lookupCache("key")
		// 1: load("key")
		// 2: load("key")
		// 1: loadGroup.Do("key", fn)
		// 1: fn()
		// 2: loadGroup.Do("key", fn)
		// 2: fn()
		if value, cacheHit := g.lookupCache(key); cacheHit {
			g.Stats.CacheHits.Add(1)
			return value, nil
		}
		g.Stats.LoadsDeduped.Add(1)
		var value ByteView
		var err error
		if peer, ok := g.peers.PickPeer(key); ok {
			value, err = g.getFromPeer(ctx, peer, key)
			if err == nil {
				g.Stats.PeerLoads.Add(1)
				return value, nil
			}
			g.Stats.PeerErrors.Add(1)
			// TODO(bradfitz): log the peer's error? keep
			// log of the past few for /groupcachez?  It's
			// probably boring (normal task movement), so not
			// worth logging I imagine.
		}
		value, err = g.getLocally(ctx, key, dest)
		if err != nil {
			g.Stats.LocalLoadErrs.Add(1)
			return nil, err
		}
		g.Stats.LocalLoads.Add(1)
		destPopulated = true // only one caller of load gets this return value
		g.populateCache(key, value, &g.mainCache)
		return value, nil
	})
	if err == nil {
		value = viewi.(ByteView)
	}
	return
}
```

#### 防击穿和穿透

load方法调用了g.loadGroup.Do的执行逻辑，那么这里我穿插着把g.loadGroup.Do说了，这个方法就是前面说的singleflight文件夹中的group结构实现的Do方法：

```
// call is an in-flight or completed Do call
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}

// Group represents a class of work and forms a namespace in which
// units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

虽然没几行代码但他起到了防止**内存穿透**和**内存击穿**的作用，是非常值得我们学习的。获取每个key都建立一个索引连接到一个存值和waitgroup指针的结构，同一时间同一key进行访问时大家将会一起等待并服用第一个请求的结果，相当于同一时间获取同一资源的请求并发量最大只有1。所以不会出现同一时间某热点数据内存失效导致的内存击穿，或是某热点数据不存在于数据库导致的内存穿透。

回到load方法，该方法先再次查询本地缓存，为什么再次查询已经在注释中说明了，防止并发请求同一key A 结束了但 B 还没进入Do。然后进行的节点选取和通过远程节点获取资源，这一步会在之后的分布式中描述。

取到就返回资源值，取不到就调用g.getlocally来通过本地定义的获取方法来获取。

```
func (g *Group) getLocally(ctx context.Context, key string, dest Sink) (ByteView, error) {
	err := g.getter.Get(ctx, key, dest)
	if err != nil {
		return ByteView{}, err
	}
	return dest.view()
}
```

getter其实就是我们之前定义好的获取资源的方法，这一步如果仍然获取不到那说明资源不存在。

你肯定会注意到这个getter方法，这个getter方法才是获取数据的源头，需要我们自己定义并在新建group时进行传入，那么我们现在回过头来看newgroup方法：

```
// The group name must be unique for each getter.
func NewGroup(name string, cacheBytes int64, getter Getter) *Group {
	return newGroup(name, cacheBytes, getter, nil)
}

// If peers is nil, the peerPicker is called via a sync.Once to initialize it.
func newGroup(name string, cacheBytes int64, getter Getter, peers PeerPicker) *Group {
	if getter == nil {
		panic("nil Getter")
	}
	mu.Lock()
	defer mu.Unlock()
	initPeerServerOnce.Do(callInitPeerServer)
	if _, dup := groups[name]; dup {
		panic("duplicate registration of group " + name)
	}
	g := &Group{
		name:       name,
		getter:     getter,
		peers:      peers,
		cacheBytes: cacheBytes,
		loadGroup:  &singleflight.Group{},
	}
	if fn := newGroupHook; fn != nil {
		fn(g)
	}
	groups[name] = g
	return g
}
```

可以看到新建一个group是需要你传入 getter 接口的，没错getter是一个接口他实现了一个Get方法：

```
// A Getter loads data for a key.
type Getter interface {
	// Get returns the value identified by key, populating dest.
	//
	// The returned data must be unversioned. That is, key must
	// uniquely describe the loaded data, without an implicit
	// current time, and without relying on cache expiration
	// mechanisms.
	Get(ctx context.Context, key string, dest Sink) error
}


// A GetterFunc implements Getter with a function.
type GetterFunc func(ctx context.Context, key string, dest Sink) error

func (f GetterFunc) Get(ctx context.Context, key string, dest Sink) error {
	return f(ctx, key, dest)
}
```

我们要做的就是去做一个结构实现这个Get，但贴心的作者显然已经考虑到这个方法的繁琐，他帮我们做了这个GetterFunc函数签名实现了Getter接口，这样我们就可以直接传入方法了。

```
func testSetup() {
	stringGroup = NewGroup(stringGroupName, cacheSize, GetterFunc(func(_ context.Context, key string, dest Sink) error {
		if key == fromChan {
			key = <-stringc
		}
		cacheFills.Add(1)
		return dest.SetString("ECHO:" + key)
	}))

	protoGroup = NewGroup(protoGroupName, cacheSize, GetterFunc(func(_ context.Context, key string, dest Sink) error {
		if key == fromChan {
			key = <-stringc
		}
		cacheFills.Add(1)
		return dest.SetProto(&testpb.TestMessage{
			Name: proto.String("ECHO:" + key),
			City: proto.String("SOME-CITY"),
		})
	}))
}
```

我们看一下作者的调用示例。

Newgroup 方法新建 group 后还会维护一个全局的 groups，以 group.name 为 key，当我们需要获取某个 group 并从中获取资源时，我们只需要传入groupname，和获取要参数的 key 。

```
groups = make(map[string]*Group)
```

到此为止获取的逻辑我估计你已经很清晰了。下面我们看缓存的逻辑。

### 缓存

#### 主要逻辑

你肯定已经发现了在group结构中有mainCache，hotCache。我们在获取本地缓存时是从这两个缓存结构中取得的。而这两个结构其实使用的是同一个cache结构体类型：

```
// values.
type cache struct {
	mu         sync.RWMutex
	nbytes     int64 // of all keys and values
	lru        *lru.Cache
	nhit, nget int64
	nevict     int64 // number of evictions
}
```

这个cache结构是存储结构的最外层。他的内层存储结构其实是 lru 即 lru.Cache 类型的结构。

我们来看下这个缓存结构的增删改查方法：

```
func (c *cache) add(key string, value ByteView) {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.lru == nil {
		c.lru = &lru.Cache{
			OnEvicted: func(key lru.Key, value interface{}) {
				val := value.(ByteView)
				c.nbytes -= int64(len(key.(string))) + int64(val.Len())
				c.nevict++
			},
		}
	}
	c.lru.Add(key, value)
	c.nbytes += int64(len(key)) + int64(value.Len())
}

func (c *cache) get(key string) (value ByteView, ok bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.nget++
	if c.lru == nil {
		return
	}
	vi, ok := c.lru.Get(key)
	if !ok {
		return
	}
	c.nhit++
	return vi.(ByteView), true
}

func (c *cache) removeOldest() {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.lru != nil {
		c.lru.RemoveOldest()
	}
}

func (c *cache) bytes() int64 {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.nbytes
}

func (c *cache) items() int64 {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.itemsLocked()
}

func (c *cache) itemsLocked() int64 {
	if c.lru == nil {
		return 0
	}
	return int64(c.lru.Len())
}
```

可以看到这个内存的主体其实就是lruCache。加入了锁和长度统计。

你在看group主体代码的时候是不是会有疑问，这个 group 结构遇到新的 key 就进行一次存储是不是会导致内存溢出，还是说他有定时清理的逻辑？其实看到 lru 的瞬间就应该悟到并没有定时清理的逻辑清理的逻辑，清理逻辑都在lru.Cache中内置了，这个缓存的淘汰机制是当内存过大时淘汰最老的数据。

还记得上面说过 3 种获取方法中从定义的getter方法中获取和从远端节点获取都会进行本地缓存吗？我们下面回到group Get 方法中结合上面的缓存结构看看他是如何做缓存的：

```
g.populateCache(key, value, &g.mainCache)
```

可以看到在上面贴出的的 group.Get 代码中有以上一行调用了 g.populateCache 其参数是获取资源的 key ，获取到的资源 value 和 group结构中的 mainCahce 缓存：

```
func (g *Group) populateCache(key string, value ByteView, cache *cache) {
	if g.cacheBytes <= 0 {
		return
	}
	cache.add(key, value)

	// Evict items from cache(s) if necessary.
	for {
		mainBytes := g.mainCache.bytes()
		hotBytes := g.hotCache.bytes()
		if mainBytes+hotBytes <= g.cacheBytes {
			return
		}

		// TODO(bradfitz): this is good-enough-for-now logic.
		// It should be something based on measurements and/or
		// respecting the costs of different resources.
		victim := &g.mainCache
		if hotBytes > mainBytes/8 {
			victim = &g.hotCache
		}
		victim.removeOldest()
	}
}
```

如果你回顾刚刚我们说的group.Get 方法你就会发现这个 populateCache 中的 cache 参数在通过外部节点获取到数据时是传入的hotCache，而在本地用已定义的 getter 方法获取到时是传入的 mainCache 。所以 mainCache 保存的是从本地调用getter方法获取到的值，hotCache 是缓存从远端节点获取的资源。

我们继续来看这个 populateCache 这个逻辑，调用 add 方法将资源存储进缓存中，然后通过计算 mainBytes 和 hotBytes 的总和是否大于设置的上限 **g.CacheBytes** 来判断是否要进行缓存清理，这个 CacheBytes 是规定的缓存 group 的最大上限在 newgroup 时传入的参数。如果达到上限就调用 removeOldest 来清理缓存。 这个 mainBytes 和 hotBytes 是在 lrucache 的添加时进行统计的。那么清理谁？是 main 还是 hot ? 逻辑很简单如果 hotBytes 大于 mainBytes 八分之一那么就清理 hotBytes ，控制 hotCache 是 mainCache 的八分之一。

可以看到缓存的主体其实就是 lru.Cahce 所以我们来看lru.Cache ：

#### LruCache

```
// Cache is an LRU cache. It is not safe for concurrent access.
type Cache struct {
	// MaxEntries is the maximum number of cache entries before
	// an item is evicted. Zero means no limit.
	MaxEntries int

	// OnEvicted optionally specifies a callback function to be
	// executed when an entry is purged from the cache.
	OnEvicted func(key Key, value interface{})

	ll    *list.List
	cache map[interface{}]*list.Element
}

// A Key may be any value that is comparable. See http://golang.org/ref/spec#Comparison_operators
type Key interface{}

type entry struct {
	key   Key
	value interface{}
}

// New creates a new Cache.
// If maxEntries is zero, the cache has no limit and it's assumed
// that eviction is done by the caller.
func New(maxEntries int) *Cache {
	return &Cache{
		MaxEntries: maxEntries,
		ll:         list.New(),
		cache:      make(map[interface{}]*list.Element),
	}
}

// Add adds a value to the cache.
func (c *Cache) Add(key Key, value interface{}) {
	if c.cache == nil {
		c.cache = make(map[interface{}]*list.Element)
		c.ll = list.New()
	}
	if ee, ok := c.cache[key]; ok {
		c.ll.MoveToFront(ee)
		ee.Value.(*entry).value = value
		return
	}
	ele := c.ll.PushFront(&entry{key, value})
	c.cache[key] = ele
	if c.MaxEntries != 0 && c.ll.Len() > c.MaxEntries {
		c.RemoveOldest()
	}
}

// Get looks up a key's value from the cache.
func (c *Cache) Get(key Key) (value interface{}, ok bool) {
	if c.cache == nil {
		return
	}
	if ele, hit := c.cache[key]; hit {
		c.ll.MoveToFront(ele)
		return ele.Value.(*entry).value, true
	}
	return
}

// Remove removes the provided key from the cache.
func (c *Cache) Remove(key Key) {
	if c.cache == nil {
		return
	}
	if ele, hit := c.cache[key]; hit {
		c.removeElement(ele)
	}
}

// RemoveOldest removes the oldest item from the cache.
func (c *Cache) RemoveOldest() {
	if c.cache == nil {
		return
	}
	ele := c.ll.Back()
	if ele != nil {
		c.removeElement(ele)
	}
}

func (c *Cache) removeElement(e *list.Element) {
	c.ll.Remove(e)
	kv := e.Value.(*entry)
	delete(c.cache, kv.key)
	if c.OnEvicted != nil {
		c.OnEvicted(kv.key, kv.value)
	}
}

// Len returns the number of items in the cache.
func (c *Cache) Len() int {
	if c.cache == nil {
		return 0
	}
	return c.ll.Len()
}

// Clear purges all stored items from the cache.
func (c *Cache) Clear() {
	if c.OnEvicted != nil {
		for _, e := range c.cache {
			kv := e.Value.(*entry)
			c.OnEvicted(kv.key, kv.value)
		}
	}
	c.ll = nil
	c.cache = nil
}
```

可以看到 lru.Cache 是一个双向链表，add 逻辑是已有数据移动到链表最前面，新数据直接插入到链表最前面。也就是根据用户的查询不断的把最新查询的数据放到链表最前面，也就是说链表的最后是相对这个链表数据最后一次查询时间最晚的，也可以说是最旧的或是不火热的数据。当我们要清理的时候将链表最后一个数据清理掉即可。这就是 lru 淘汰机制。

数据存储这块我们了解了，获取数据的时候，作者为了将方法做的通用也做了一个接口Sink，你再翻回上面的 group.Get 的方法会发现接受数据的 dest 参数类型就是一个 Sink 接口，我们来看下这个接口以及作者设计的几个实现类。

#### Sink

```
// A Sink receives data from a Get call.
//
// Implementation of Getter must call exactly one of the Set methods
// on success.
type Sink interface {
	// SetString sets the value to s.
	SetString(s string) error

	// SetBytes sets the value to the contents of v.
	// The caller retains ownership of v.
	SetBytes(v []byte) error

	// SetProto sets the value to the encoded version of m.
	// The caller retains ownership of m.
	SetProto(m proto.Message) error

	// view returns a frozen view of the bytes for caching.
	view() (ByteView, error)
}
```

接口实现的几个方法就是set 不同类型的 set ，还有 view 。我们先看 view 方法的函数签名，这个方法其实就是将实现 Sink 类中

资源转换为 ByteView 结构。

在看其他几个 set 相关的函数签名，看之前我们回过头再看 group.Get 这个方法在获取数据成功后都会调用：

```
setSinkView(dest, value)
```

使用这个方法将value，赋值给 dest 这个 dest 就是我们之前 group.Get 方法中参数传入的 Sink 接收器。

我们看这个 setSinkView 方法：

```
func setSinkView(s Sink, v ByteView) error {
	// A viewSetter is a Sink that can also receive its value from
	// a ByteView. This is a fast path to minimize copies when the
	// item was already cached locally in memory (where it's
	// cached as a ByteView)
	type viewSetter interface {
		setView(v ByteView) error
	}
	if vs, ok := s.(viewSetter); ok {
		return vs.setView(v)
	}
	if v.b != nil {
		return s.SetBytes(v.b)
	}
	return s.SetString(v.s)
}
```

就是调用不同优先级的set 方法，有 setView 就掉 setView。有byte就掉 setBytes 都没有就掉 SetString。

你会发现另外一个点就是 ByteView 这个结构是 group.Get 的三种不同获取数据的方式的统一输出数据格式，为什么要统一输出格式因为存入缓存的类型要统一：

#### ByteView

```
// A ByteView holds an immutable view of bytes.
// Internally it wraps either a []byte or a string,
// but that detail is invisible to callers.
//
// A ByteView is meant to be used as a value type, not
// a pointer (like a time.Time).
type ByteView struct {
	// If b is non-nil, b is used, else s is used.
	b []byte
	s string
}
```

缓存到这儿就聊完了，作者将缓存套了比较多的层，主要还是为了包的灵活性考虑，多看几遍就可以明白。

### 分布式

分布式设计是 groupcache 库的核心设计也是我认为最巧妙的地方。你是否会有这样的疑问：一个节点查找某一个 Key 对应资源，在本地内存中没有找到，当他要去远程节点查找资源时他如何知道这个资源在哪个节点？接下来聊的就会对这个问题进行解答。

这个分布式的设计都在 http.go 文件中，这个文件中 HTTPPool 结构实现了这样几个功能：

1. 形成缓存服务端，注册获取资源http请求方法。
2. 形成其他节点的客户端，并根据IP对客户端进行索引。
3. 形成节点选择器，在某Key对应资源不存在于本地时，选择正确的节点并通过该节点客户端对节点 进行访问。

#### HTTPPool

先看一下HTTPPool的结构：

```
// HTTPPool implements PeerPicker for a pool of HTTP peers.
type HTTPPool struct {
	// Context optionally specifies a context for the server to use when it
	// receives a request.
	// If nil, the server uses the request's context
	Context func(*http.Request) context.Context

	// Transport optionally specifies an http.RoundTripper for the client
	// to use when it makes a request.
	// If nil, the client uses http.DefaultTransport.
	Transport func(context.Context) http.RoundTripper

	// this peer's base URL, e.g. "https://example.net:8000"
	self string

	// opts specifies the options.
	opts HTTPPoolOptions

	mu          sync.Mutex // guards peers and httpGetters
	peers       *consistenthash.Map
	httpGetters map[string]*httpGetter // keyed by e.g. "http://10.0.0.2:8008"
}

// HTTPPoolOptions are the configurations of a HTTPPool.
type HTTPPoolOptions struct {
	// BasePath specifies the HTTP path that will serve groupcache requests.
	// If blank, it defaults to "/_groupcache/".
	BasePath string

	// Replicas specifies the number of key replicas on the consistent hash.
	// If blank, it defaults to 50.
	Replicas int

	// HashFn specifies the hash function of the consistent hash.
	// If blank, it defaults to crc32.ChecksumIEEE.
	HashFn consistenthash.Hash
}
```

接下来我们一点一点的来看看

#### 服务端

分布式的groupcache缓存服务的每个节点都即是服务端又是客户端，那么这个节点如何形成一个服务端？我们先看 NewHTTPPool 这个方法：

```
// NewHTTPPool initializes an HTTP pool of peers, and registers itself as a PeerPicker.
// For convenience, it also registers itself as an http.Handler with http.DefaultServeMux.
// The self argument should be a valid base URL that points to the current server,
// for example "http://example.net:8000".
func NewHTTPPool(self string) *HTTPPool {
	p := NewHTTPPoolOpts(self, nil)
	http.Handle(p.opts.BasePath, p)
	return p
}

var httpPoolMade bool

// NewHTTPPoolOpts initializes an HTTP pool of peers with the given options.
// Unlike NewHTTPPool, this function does not register the created pool as an HTTP handler.
// The returned *HTTPPool implements http.Handler and must be registered using http.Handle.
func NewHTTPPoolOpts(self string, o *HTTPPoolOptions) *HTTPPool {
	if httpPoolMade {
		panic("groupcache: NewHTTPPool must be called only once")
	}
	httpPoolMade = true

	p := &HTTPPool{
		self:        self,
		httpGetters: make(map[string]*httpGetter),
	}
	if o != nil {
		p.opts = *o
	}
	if p.opts.BasePath == "" {
		p.opts.BasePath = defaultBasePath
	}
	if p.opts.Replicas == 0 {
		p.opts.Replicas = defaultReplicas
	}
	p.peers = consistenthash.New(p.opts.Replicas, p.opts.HashFn)

	RegisterPeerPicker(func() PeerPicker { return p })
	return p
}
```

熟悉HTTP包的朋友肯定一眼就看出来这个 HTTPPool 肯定是实现了 Handler 接口的 SeveHTTP 的方法，在外部对 p.opts.BasePath 的请求过来时将会直接调用这个 SeveHTTP 方法进行资源请求，完全没错，那么我们直接来看这个SeveHTTP方法，如果你不熟悉http包也无所谓你只要知道外部节点需要请求这个 p.opts.BasePath 路径，然后服务端会调用这个 SeveHTTP 方法进行响应。

```
func (p *HTTPPool) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Parse request.
	if !strings.HasPrefix(r.URL.Path, p.opts.BasePath) {
		panic("HTTPPool serving unexpected path: " + r.URL.Path)
	}
	parts := strings.SplitN(r.URL.Path[len(p.opts.BasePath):], "/", 2)
	if len(parts) != 2 {
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}
	groupName := parts[0]
	key := parts[1]

	// Fetch the value for this group/key.
	group := GetGroup(groupName)
	if group == nil {
		http.Error(w, "no such group: "+groupName, http.StatusNotFound)
		return
	}
	var ctx context.Context
	if p.Context != nil {
		ctx = p.Context(r)
	} else {
		ctx = r.Context()
	}

	group.Stats.ServerRequests.Add(1)
	var value []byte
	err := group.Get(ctx, key, AllocatingByteSliceSink(&value))
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Write the value to the response body as a proto message.
	body, err := proto.Marshal(&pb.GetResponse{Value: value})
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", "application/x-protobuf")
	w.Write(body)
}
```

其实也是一样的调用了 group.Get 的逻辑。这样只要新建了这个 httppool 在进行一个监听我们就可以形成一个服务端。

#### 客户端

每个 groupcache 的节点都需要和其他节点进行联系形成客户端，如何形成客户端：

```
// Set updates the pool's list of peers.
// Each peer value should be a valid base URL,
// for example "http://example.net:8000".
func (p *HTTPPool) Set(peers ...string) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.peers = consistenthash.New(p.opts.Replicas, p.opts.HashFn)
	p.peers.Add(peers...)
	p.httpGetters = make(map[string]*httpGetter, len(peers))
	for _, peer := range peers {
		p.httpGetters[peer] = &httpGetter{transport: p.Transport, baseURL: peer + p.opts.BasePath}
	}
}
```

新建 HTTPPool 结构后使用 set 方法进行节点添加，这个 peer 就是类似http://example.net:8000的一个http协议加IP地址和端口，当然这个一定是对应了你其他节点形成的服务端的监听。把所有的IP都Set进去然后生成一个httpGetter结构并和其对应的peer生成索引关系。这个httpGetter 结构就是所谓的链接对应节点的客户端。下面我们来看下这个结构：

```
type httpGetter struct {
	transport func(context.Context) http.RoundTripper
	baseURL   string
}

var bufferPool = sync.Pool{
	New: func() interface{} { return new(bytes.Buffer) },
}

func (h *httpGetter) Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error {
	u := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(in.GetGroup()),
		url.QueryEscape(in.GetKey()),
	)
	req, err := http.NewRequest("GET", u, nil)
	if err != nil {
		return err
	}
	req = req.WithContext(ctx)
	tr := http.DefaultTransport
	if h.transport != nil {
		tr = h.transport(ctx)
	}
	res, err := tr.RoundTrip(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	if res.StatusCode != http.StatusOK {
		return fmt.Errorf("server returned: %v", res.Status)
	}
	b := bufferPool.Get().(*bytes.Buffer)
	b.Reset()
	defer bufferPool.Put(b)
	_, err = io.Copy(b, res.Body)
	if err != nil {
		return fmt.Errorf("reading response body: %v", err)
	}
	err = proto.Unmarshal(b.Bytes(), out)
	if err != nil {
		return fmt.Errorf("decoding response body: %v", err)
	}
	return nil
}
```

可以看到这个结构吧调用地址缓存起来并实现了Get方法而这个方法就是客户端与请求对应节点资源的方法。

这时你一定想到了从远端节点获取资源肯定是选择到资源所在的节点然后取到这个 httpGetter 然后掉用 Get 方法。没错你又想对了，接下来我们再回到group.Get这个逻辑里，看从远端节点获取资源的方法 group.getFromPeer：

```
func (g *Group) getFromPeer(ctx context.Context, peer ProtoGetter, key string) (ByteView, error) {
	req := &pb.GetRequest{
		Group: &g.name,
		Key:   &key,
	}
	res := &pb.GetResponse{}
	err := peer.Get(ctx, req, res)
	if err != nil {
		return ByteView{}, err
	}
	value := ByteView{b: res.Value}
	// TODO(bradfitz): use res.MinuteQps or something smart to
	// conditionally populate hotCache.  For now just do it some
	// percentage of the time.
	if rand.Intn(10) == 0 {
		g.populateCache(key, value, &g.hotCache)
	}
	return value, nil
}
```

可以看到入参 peer 是一个 ProtoGetter 的接口，我们来看这个接口：

```
// ProtoGetter is the interface that must be implemented by a peer.
type ProtoGetter interface {
	Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error
}
```

其实就是只要实现了Get方法就可以，这个peer.Get 入参其实就是 httpGetter 结构的入参。那么这个peer应该就是刚刚我们注册的客户端 httpGetter。我们的猜想是否正确，我们往调用 getFromPeer 方法的上面回溯，看这个peer到低是什么：

```
if peer, ok := g.peers.PickPeer(key); ok {
	value, err = g.getFromPeer(ctx, peer, key)
	if err == nil {
		g.Stats.PeerLoads.Add(1)
		return value, nil
	}
	g.Stats.PeerErrors.Add(1)
	// TODO(bradfitz): log the peer's error? keep
	// log of the past few for /groupcachez?  It's
	// probably boring (normal task movement), so not
	// worth logging I imagine.
}
```

可以看到这个 peer 参数其实是通过g.peers.PickPeer这个方法获取到的，那我们想看这个方法就要先看这个 group 中的 peers 结构是啥：

```
peers      PeerPicker
```

又是一个接口 PeerPicker 。。。不要急我们回溯一下就会发现新建 group 时这个 peers 就是之间通过入参数传入的。那么这个peer是个啥的线索就断了。我们看下 http_test.go 作者居然直接调用的 NewGroup 来新建 group：

```
// The group name must be unique for each getter.
func NewGroup(name string, cacheBytes int64, getter Getter) *Group {
	return newGroup(name, cacheBytes, getter, nil)
}

// If peers is nil, the peerPicker is called via a sync.Once to initialize it.
func newGroup(name string, cacheBytes int64, getter Getter, peers PeerPicker) *Group {
```

那这个peer 是个nil ？？其实看到这里我也很懵逼，但你注意注释，作者考虑到了你会蒙，特意告诉你有一个peerPicker会调用一个初始化的方法来注入他。我们看这个peerPicker接口确实有一个注册的方法：

```
// RegisterPeerPicker registers the peer initialization function.
// It is called once, when the first group is created.
// Either RegisterPeerPicker or RegisterPerGroupPeerPicker should be
// called exactly once, but not both.
func RegisterPeerPicker(fn func() PeerPicker) {
	if portPicker != nil {
		panic("RegisterPeerPicker called more than once")
	}
	portPicker = func(_ string) PeerPicker { return fn() }
}
```

这个方法原来在HewHTTPPool时调用了，我们回溯NewHTTPPoolOpts这个方法发现这个 portPicker 注册返回的原来就是这个新建的HTTPPool结构。看到这里又一个疑问出现了这个 注册的portPicker方法是何时赋值给了group的peers，经过我的不懈努力终于找到了:

```
func (g *Group) initPeers() {
	if g.peers == nil {
		g.peers = getPeers(g.name)
	}
}

func (g *Group) Get(ctx context.Context, key string, dest Sink) error {
	g.peersOnce.Do(g.initPeers)
go
func getPeers(groupName string) PeerPicker {
	if portPicker == nil {
		return NoPeers{}
	}
	pk := portPicker(groupName)
	if pk == nil {
		pk = NoPeers{}
	}
	return pk
}
```

原来是在第一group.Get时调用了一次 initPeers ，而initPeers又调用了 getPeers ，getPeers 调用了刚刚注册的 portPicker 并把新建的 HTTPPool 赋值给了 group.peers 。原来这个peers也是HTTPPool。那么这个HTTPPool一定是实现了pickpeer 这个方法。看一下果然没错：

```
func (p *HTTPPool) PickPeer(key string) (ProtoGetter, bool) {
	p.mu.Lock()
	defer p.mu.Unlock()
	if p.peers.IsEmpty() {
		return nil, false
	}
	if peer := p.peers.Get(key); peer != p.self {
		return p.httpGetters[peer], true
	}
	return nil, false
}
```

在这里我们的猜想终于闭环了。这个 protoGetter 的接口果然是我们刚刚注册的客户端httpGetter。

我自己看这里的时候很绕，希望我的解释可以让你梳理的更快。。。

#### 节点选择

好了我们就快完成了groupCache的解析，现在还差一点就是我一开始抛出的疑问，如何通过资源的取得其所在的节点IP呢？这个资源的Key其实和节点是没有任何联系的，我们怎么给他们建立一个关系并让所有不同的Key都稳定的在某一个节点缓存且我可以通过Key去找到其所在的节点并访问获取Key对应的资源？

在上面的 PickPeer 方法中，可以看到是通过 peers.Get 获取到了节点peer，那么奥秘一定是在这个Get方法中，我们来看这个方法：

```
// Get gets the closest item in the hash to the provided key.
func (m *Map) Get(key string) string {
	if m.IsEmpty() {
		return ""
	}

	hash := int(m.hash([]byte(key)))

	// Binary search for appropriate replica.
	idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })

	// Means we have cycled back to the first replica.
	if idx == len(m.keys) {
		idx = 0
	}

	return m.hashMap[m.keys[idx]]
}
```

这个httppool中的 peers 结构是 *consistenthash.Map 结构，这个结构实现了节点和资源Key的绑定与选择。通过Get方法我们无法看出其逻辑我们就整体看这个Map结构：

```
type Hash func(data []byte) uint32

type Map struct {
	hash     Hash
	replicas int
	keys     []int // Sorted
	hashMap  map[int]string
}

func New(replicas int, fn Hash) *Map {
	m := &Map{
		replicas: replicas,
		hash:     fn,
		hashMap:  make(map[int]string),
	}
	if m.hash == nil {
		m.hash = crc32.ChecksumIEEE
	}
	return m
}

// IsEmpty returns true if there are no items available.
func (m *Map) IsEmpty() bool {
	return len(m.keys) == 0
}

// Add adds some keys to the hash.
func (m *Map) Add(keys ...string) {
	for _, key := range keys {
		for i := 0; i < m.replicas; i++ {
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			m.keys = append(m.keys, hash)
			m.hashMap[hash] = key
		}
	}
	sort.Ints(m.keys)
}
```

这个Add方法的入参 keys 其实就是所有节点的访问地址类似http://example.net:8000刚刚说过了。每个节点建立50个副本，将副本的index和节点key 相加利用哈希算法形成一个随机数这个数建立和节点的索引，每个节点都会生成50个随机数并索引到这个节点，那么4个节点就是200个随机数，每个节点都会对应50个随机数。

好了了解了这些我们再回溯Get方法是不是豁然开朗。每获取到一个请求资源key用同样的方式来形成一个随机数，然后用二分查找的方法选择一个最接近他的节点随机数副本，这个key就和节点关联了起来，以后我每次请求这个资源都将选择到相同的节点上面。

这回我们的groupcache源码就完全读完了，接下来我们看一下测试用例，这个用例一定要重视，看会了这个用例那么你就会用了这个groupcache，这个用例就在http_test.go中我把它贴在这里：

### 测试范例

```
var (
	peerAddrs = flag.String("test_peer_addrs", "", "Comma-separated list of peer addresses; used by TestHTTPPool")
	peerIndex = flag.Int("test_peer_index", -1, "Index of which peer this child is; used by TestHTTPPool")
	peerChild = flag.Bool("test_peer_child", false, "True if running as a child process; used by TestHTTPPool")
)

func TestHTTPPool(t *testing.T) {
	if *peerChild {
		beChildForTestHTTPPool()
		os.Exit(0)
	}

	const (
		nChild = 4
		nGets  = 100
	)

	var childAddr []string
	for i := 0; i < nChild; i++ {
		childAddr = append(childAddr, pickFreeAddr(t))
	}

	var cmds []*exec.Cmd
	var wg sync.WaitGroup
	for i := 0; i < nChild; i++ {
		cmd := exec.Command(os.Args[0],
			"--test.run=TestHTTPPool",
			"--test_peer_child",
			"--test_peer_addrs="+strings.Join(childAddr, ","),
			"--test_peer_index="+strconv.Itoa(i),
		)
		cmds = append(cmds, cmd)
		wg.Add(1)
		if err := cmd.Start(); err != nil {
			t.Fatal("failed to start child process: ", err)
		}
		go awaitAddrReady(t, childAddr[i], &wg)
	}
	defer func() {
		for i := 0; i < nChild; i++ {
			if cmds[i].Process != nil {
				cmds[i].Process.Kill()
			}
		}
	}()
	wg.Wait()

	// Use a dummy self address so that we don't handle gets in-process.
	p := NewHTTPPool("should-be-ignored")
	p.Set(addrToURL(childAddr)...)

	// Dummy getter function. Gets should go to children only.
	// The only time this process will handle a get is when the
	// children can't be contacted for some reason.
	getter := GetterFunc(func(ctx context.Context, key string, dest Sink) error {
		return errors.New("parent getter called; something's wrong")
	})
	g := NewGroup("httpPoolTest", 1<<20, getter)

	for _, key := range testKeys(nGets) {
		var value string
		if err := g.Get(context.TODO(), key, StringSink(&value)); err != nil {
			t.Fatal(err)
		}
		if suffix := ":" + key; !strings.HasSuffix(value, suffix) {
			t.Errorf("Get(%q) = %q, want value ending in %q", key, value, suffix)
		}
		t.Logf("Get key=%q, value=%q (peer:key)", key, value)
	}
}
```

这个用例建立三个假的本地节点，每个节点都mock了获取key 对应资源的getter方法。然后取获取key 时会将key 和节点序号返回。这个用例就自己看吧，相信读完本文你看这个用例将毫不费力。

## 优化

这个groupcache的节点注册是静态的，也就是说你要预先定义好所有节点的IP。

这样一来就失去了高可用性：一旦某一个节点挂了将会出现某些Key缓存无法访问的问题。

也完全失去了拓展性：在缓存压力大的情况下我希望可以横向拓展，这个设计就无法支持，在现在这个微服务的时代可拓展性可以说是一个服务必须具备的。

无法支持节点IP变动：在这个容器的时代我们基本不会去将一个服务的IP固定起来，基于k8s的pod的IP是不断变动的。

那么我们该如何优化呢？我们可使用etcd来缓存我们的所有节点IP（每启动一个服务都获取自己IP并注册到etcd，注意同一集群前缀相同），不断地获取前缀相同的key并进行对应节点IP获取，并不断监听Key的变化在根据变化不断更新上文的节点选择器。

## 总结

groupcache可以提升我们的程序性能并防止击穿，穿透，雪崩等服务问题，而且轻量化易于上手节省资源，希望这篇文章对你有帮助，有疑问可以留下你的评论我们共同进步，最后这篇文章实在是太长了，我不想回头看那些话啰嗦，那些话语法有问题或是错别字了，遇到类似问题希望见谅见谅。