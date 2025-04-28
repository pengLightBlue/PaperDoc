# 深入理解 sync.RWMutex 实现原理

## 1 前言

读写锁sync.RWMutex，是互斥锁Mutex的一个改进版，适用于**读取数据频率远大于写数据频率**的场景

例如，程序中写操作少而读操作多，如果执行过程是1次写然后N次读的话，使用Mutex这个过程将是串行的，带来的问题是：即便N次读操作互相之间并不影响，也需要获取到Mutex后才可以操作。如果使用读写锁，多个读操作可以同时持有锁，将大大**提升并发能力**

读写锁的基本性质为：

1.  一个协程拥有写锁时，其他协程获取写锁需要阻塞
2.  一个协程拥有写锁时，其他协程获取读锁需要阻塞
3.  一个协程拥有读锁时，其他协程获取写锁需要阻塞
4.  一个协程拥有读锁时，其他协程也可以获取读锁

即：**写写，读写互斥，读读不互斥**

相比于java的ReentrantReadWriteLock，go的读写锁不支持可重入，也没用锁降级这种复杂的机制，就是一个纯粹的读写锁，非常适合用于学习如何优雅设计一个读写锁

## 2 数据结构

在源码/src/sync/rwmutex.go中定义了读写锁的数据结构：

```go
type RWMutex struct {
    w           Mutex  // 保证写锁之间互斥
    writerSem   uint32 // 写协程阻塞的信号
    readerSem   uint32 // 读协程阻塞的信号
    readerCount int32  // 已经持有读锁的协程数量
    readerWait  int32  // 需要等到多少读锁释放后可以加写锁
}
```

读写锁提供了4个方法：

-   RLock()：获取读锁
-   RUnlock()：释放读锁
-   Lock(): 获取写锁
-   Unlock()：释放解锁

接下来分析这4个方法的具体流程：

## 3 执行流程

### 3.1 获取写锁

```go
func (rw *RWMutex) Lock() {
    // 获取锁w，告知其他要写的协程，这里要加写锁
	rw.w.Lock()
	// 告诉其他读的协程，这里有个协程要加写锁
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	
    // 如果此时有其他协程在读，阻塞等待
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}
```

该方法做了以下几件事：

1.  获取互斥锁，达到写锁之间的互斥效果
2.  将readerCount减去一个很大的值，这样如果后续有协程相加读锁时，发现readerCount是一个负数，就会加锁失败，乖乖在后面排队
3.  将此时的readerCount加到readerWait字段中，表示要**等前面这么多个读锁释放后，自己才能加上写锁**
4.  陷入阻塞

### 3.2 释放写锁

```go
func (rw *RWMutex) Unlock() {
	
	// 告诉其他读协程，现在没有协程持有写锁了
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
 
	// 将后面阻塞的读协程全部唤醒
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// 释放w
	rw.w.Unlock()
}
```

释放写锁的流程为：

1.  将之前在readerCount中减的值加回来，告诉其他读协程，现在没有协程持有写锁了，这样后续的读请求可以加锁成功
2.  将此时阻塞的读请求全部唤醒
3.  释放w，这样其他的写请求可以加锁成功  

### 3.3 获取读锁

```go
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```

加读锁的流程比较简单，往readerCount+1，如果此时该值为负数，表示前面有协程想要加写锁，或者已经获取到写锁，就阻塞当前协程

### 3.4 释放读锁

```go
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		rw.rUnlockSlow(r)
	}
}
 
func (rw *RWMutex) rUnlockSlow(r int32) {
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

释放读锁的流程为：将readerCount-1，如果此时该值为负数，说明后面有协程准备写写锁，进而检查写协程需要等待的读协程是否全部释放完毕，若是，就唤醒写协程

需要注意的是，并不是所有的解除读锁定操作都会唤醒该协程，而是最后一个解除读锁定的协程才会释放信号量将该协程唤醒，因为只有当所有读操作的协程释放锁后才可以唤醒写协程

## 4 场景分析

### 4.1 写操作是如何阻止写操作的

读写锁包含一个互斥锁(Mutex)，获取写锁时必须要先获取该互斥锁，意味着如果协程A获取了互斥锁，那么协程B只能阻塞等待该互斥锁

所以，**写操作依赖互斥锁阻止其他的写操作**

### 4.2 写操作是如何阻止读操作的

当写锁定进行时，会先将readerCount减去2^30，从而readerCount变成了负值，此时再有读锁定到来时检测到readerCount为负值，便知道有写操作在进行，只好阻塞等待。而真实的读操作个数并不会丢失，只需要将readerCount加上2^30即可获得。

所以，**写操作将readerCount变成负值来阻止读操作的**。

### 4.3 读操作是如何阻止写操作的

读锁定会先将RWMutex.readerCount加1，此时写操作到来时发现读者数量不为0，会阻塞等待所有读操作结束。

所以，**读操作通过readerCount来将来阻止写操作的**。

### 4.4 为什么写锁定不会被饿死

我们知道，写操作要等待读操作结束后才可以获得锁，但写操作等待期间可能还有新的读操作持续到来，如果写操作等待所有读操作结束，很可能被饿死。然而，通过RWMutex.readerWait可完美解决这个问题。

写操作到来时，会把RWMutex.readerCount值拷贝到RWMutex.readerWait中，用于标记排在写操作前面的读者个数。

前面的读操作结束后，除了会递减RWMutex.readerCount，还会递减RWMutex.readerWait值，当RWMutex.readerWait值变为0时唤醒写操作，因此**写操作只用等待前面readerWait个读操作完成，即可获取到写锁，而不是无限制等待**

写操作就相当于把一段连续的读操作划分成两部分，前面的读操作结束后唤醒写操作，写操作结束后唤醒后面的读操作