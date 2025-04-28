# sync.Map实现原理

## 简介

Go 的内建 `map` 是不支持并发写操作的，原因是 `map` 写操作不是并发安全的，当你尝试多个 Goroutine 操作同一个 `map`，会产生报错：`fatal error: concurrent map writes`。

因此官方另外引入了 `sync.Map` 来满足并发编程中的应用。

`sync.Map` 的实现原理可概括为：

-   通过 read 和 dirty 两个字段将读写分离，读的数据存在只读字段 read 上，将最新写入的数据则存在 dirty 字段上
-   读取时会先查询 read，不存在再查询 dirty，写入时则只写入 dirty
-   读取 read 并不需要加锁，而读或写 dirty 都需要加锁
-   另外有 misses 字段来统计 read 被穿透的次数（被穿透指需要读 dirty 的情况），超过一定次数则将 dirty 数据同步到 read 上
-   对于删除数据则直接通过标记来延迟删除

## 数据结构

`Map` 的数据结构如下：

 ```go
type Map struct {
    // 加锁作用，保护 dirty 字段
    mu Mutex
    // 只读的数据，实际数据类型为 readOnly
    read atomic.Value
    // 最新写入的数据
    dirty map[interface{}]*entry
    // 计数器，每次需要读 dirty 则 +1
    misses int
}
 ```

其中 readOnly 的数据结构为：

```go
type readOnly struct {
    // 内建 map
    m  map[interface{}]*entry
    // 表示 dirty 里存在 read 里没有的 key，通过该字段决定是否加锁读 dirty
    amended bool
}
```

`entry` 数据结构则用于存储值的指针：

```go
type entry struct {
    p unsafe.Pointer  // 等同于 *interface{}
}
```

属性 p 有三种状态：

-   `p == nil`: 键值已经被删除，且 `m.dirty == nil`
-   `p == expunged`: 键值已经被删除，但 `m.dirty!=nil` 且 `m.dirty` 不存在该键值（expunged 实际是空接口指针）
-   除以上情况，则键值对存在，存在于 `m.read.m` 中，如果 `m.dirty!=nil` 则也存在于 `m.dirty`

`Map` 常用的有以下方法：

-   `Load`：读取指定 key 返回 value
-   `Store`： 存储（增或改）key-value
-   `Delete`： 删除指定 key

## 源码解析

### Load

```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 首先尝试从 read 中读取 readOnly 对象
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]

    // 如果不存在则尝试从 dirty 中获取
    if !ok && read.amended {
        m.mu.Lock()
        // 由于上面 read 获取没有加锁，为了安全再检查一次
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]

        // 确实不存在则从 dirty 获取
        if !ok && read.amended {
            e, ok = m.dirty[key]
            // 调用 miss 的逻辑
            m.missLocked()
        }
        m.mu.Unlock()
    }

    if !ok {
        return nil, false
    }
    // 从 entry.p 读取值
    return e.load()
}

func (m *Map) missLocked() {
    m.misses++
    if m.misses < len(m.dirty) {
        return
    }
    // 当 miss 积累过多，会将 dirty 存入 read，然后 将 amended = false，且 m.dirty = nil
    m.read.Store(readOnly{m: m.dirty})
    m.dirty = nil
    m.misses = 0
}
```

### Store

```go
func (m *Map) Store(key, value interface{}) {
    read, _ := m.read.Load().(readOnly)
    // 如果 read 里存在，则尝试存到 entry 里
    if e, ok := read.m[key]; ok && e.tryStore(&value) {
        return
    }

    // 如果上一步没执行成功，则要分情况处理
    m.mu.Lock()
    read, _ = m.read.Load().(readOnly)
    // 和 Load 一样，重新从 read 获取一次
    if e, ok := read.m[key]; ok {
        // 情况 1：read 里存在
        if e.unexpungeLocked() {
            // 如果 p == expunged，则需要先将 entry 赋值给 dirty（因为 expunged 数据不会留在 dirty）
            m.dirty[key] = e
        }
        // 用值更新 entry
        e.storeLocked(&value)
    } else if e, ok := m.dirty[key]; ok {
        // 情况 2：read 里不存在，但 dirty 里存在，则用值更新 entry
        e.storeLocked(&value)
    } else {
        // 情况 3：read 和 dirty 里都不存在
        if !read.amended {
            // 如果 amended == false，则调用 dirtyLocked 将 read 拷贝到 dirty（除了被标记删除的数据）
            m.dirtyLocked()
            // 然后将 amended 改为 true
            m.read.Store(readOnly{m: read.m, amended: true})
        }
        // 将新的键值存入 dirty
        m.dirty[key] = newEntry(value)
    }
    m.mu.Unlock()
}

func (e *entry) tryStore(i *interface{}) bool {
    for {
        p := atomic.LoadPointer(&e.p)
        if p == expunged {
            return false
        }
        if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
            return true
        }
    }
}

func (e *entry) unexpungeLocked() (wasExpunged bool) {
    return atomic.CompareAndSwapPointer(&e.p, expunged, nil)
}

func (e *entry) storeLocked(i *interface{}) {
    atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

func (m *Map) dirtyLocked() {
    if m.dirty != nil {
        return
    }

    read, _ := m.read.Load().(readOnly)
    m.dirty = make(map[interface{}]*entry, len(read.m))
    for k, e := range read.m {
        // 判断 entry 是否被删除，否则就存到 dirty 中
        if !e.tryExpungeLocked() {
            m.dirty[k] = e
        }
    }
}

func (e *entry) tryExpungeLocked() (isExpunged bool) {
    p := atomic.LoadPointer(&e.p)
    for p == nil {
        // 如果有 p == nil（即键值对被 delete），则会在这个时机被置为 expunged
        if atomic.CompareAndSwapPointer(&e.p, nil, expunged) {
            return true
        }
        p = atomic.LoadPointer(&e.p)
    }
    return p == expunged
}
```

### Delete

```go
func (m *Map) Delete(key interface{}) {
    m.LoadAndDelete(key)
}

// LoadAndDelete 作用等同于 Delete，并且会返回值与是否存在
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool) {
    // 获取逻辑和 Load 类似，read 不存在则查询 dirty
    read, _ := m.read.Load().(readOnly)
    e, ok := read.m[key]
    if !ok && read.amended {
        m.mu.Lock()
        read, _ = m.read.Load().(readOnly)
        e, ok = read.m[key]
        if !ok && read.amended {
            e, ok = m.dirty[key]
            m.missLocked()
        }
        m.mu.Unlock()
    }
    // 查询到 entry 后执行删除
    if ok {
        // 将 entry.p 标记为 nil，数据并没有实际删除
        // 真正删除数据并被被置为 expunged，是在 Store 的 tryExpungeLocked 中
        return e.delete()
    }
    return nil, false
}
```

## 总结

可见，通过这种读写分离的设计，解决了并发情况的写入安全，又使读取速度在大部分情况可以接近内建 `map`，非常适合读多写少的情况。关键要点如下：

**dirty 是 read 的超集：**

* 在迁移前，dirty 一定包含所有 read 中已有的 key（即使值被修改），因此覆盖后 read 会得到最新值。

**amended 标志的作用**

* 如果 read 和 dirty 不一致，amended=true。迁移后 amended=false（因为 read 和 dirty 一致）。

**并发读写冲突**

* 如果迁移过程中有其他协程 同时写入，新数据会直接写入 dirty，但此时 dirty 已被清空，需要重建；重建后的 dirty 会包含 read 的最新数据，保证一致性。

`sync.Map` 还有一些其他方法：

-   `Range`：遍历所有键值对，参数是回调函数
-   `LoadOrStore`：读取数据，若不存在则保存再读取

这里就不再详解了