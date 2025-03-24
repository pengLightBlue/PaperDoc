# Go select使用与底层原理

## 1. select的使用

> select 是 Go 提供的 IO 多路复用机制，可以用多个 case 同时监听多个 channel 的读写状态：

- case: 可以监听 channel 的读写信号
- default：声明默认操作，有该字段的 select 不会阻塞

```go
select {
case chan <-:
    // TODO
case <- chan:
    // TODO
default:
    // TODO
}
```

## 2. 底层原理

1. 每一个 case 对应的 channel 都会被封装到一个结构体中；
2. 当第一次执行到 select 时，会锁住所有的 channel 并且，打乱 case 结构体的顺序；
3. 按照打乱的顺序遍历，如果有就绪的信号，就直接走对应 case 的代码段，之后跳出 select；
4. 如果没有就绪的代码段，但是有 default 字段，那就走 default 的代码段，之后跳出 select；
5. 若没有 `case` 就绪且无 `default`，当前 goroutine 会被挂起，并加入所有涉及通道的等待队列。
6. 当任意一个通道就绪时，唤醒该 goroutine，重新检查所有 `case`。再次锁住所有的 channel，遍历一遍，将所有等待队列中的 goroutine 取出，之后执行就绪的代码段，跳出select。

> 怎么判断case是否就绪？
>
> - **接收操作就绪条件**：
>   - 通道非 `nil`，且（缓冲区有数据 | 有发送者在等待 | 通道已关闭）。
> - **发送操作就绪条件**：
>   - 通道非 `nil`，且（缓冲区未满 | 有接收者在等待）。
> - **优先级**：
>   - 若多个 `case` 同时就绪，随机选择一个执行。
>   - `default` 仅在所有 `case` 未就绪时执行。

## 3. 数据结构

每一个 case 对应的数据结构如下：

```Go
type scase struct {
    c           *hchan         // chan
    elem        unsafe.Pointer // 读或者写的缓冲区地址
    kind        uint16   //case语句的类型，是default、传值写数据(channel <-) 还是  取值读数据(<- channel)
    pc          uintptr // race pc (for race detector / msan)
    releasetime int64
}
```

## 4. 几种常见 case

学习了 select 的使用与原理，我们就能更轻松地分辨不同情况下的输出情况了。

### case 1

```Go
package main

import (
  "fmt"
  "time"
)

func main() {
  chan1 := make(chan int)
  chan2 := make(chan int)
  
  go func() {
    chan1 <- 1
    time.Sleep(5 * time.Second)
  }()
  go func() {
    chan2 <- 1
    time.Sleep(5 * time.Second)
  }()
  
  select {
    case <- chan1:
      fmt.Println("chan1")
    case <- chan2:
      fmt.Println("chan2")
    default:
      fmt.Println("default")
  }
}
```

三种输出都有可能。

### case2

```Go
package main

import (
  "fmt"
  "time"
)

func main() {
  chan1 := make(chan int)
  chan2 := make(chan int)
  
  select {
    case <- chan1:
      fmt.Println("chan1")
    case <- chan2:
      fmt.Println("chan2")
  }
  fmt.Println("main exit.")
}
```

上述程序会一直阻塞。

### case3

```Go
package main

import (
  "fmt"
)

func main() {
  chan1 := make(chan int)
  chan2 := make(chan int)
  
  go func() {
    close(chan1)
  }()
  go func() {
    close(chan2)
  }()
  
  select {
    case <- chan1:
      fmt.Println("chan1")
    case <- chan2:
      fmt.Println("chan2")
  }
  fmt.Println("main exit.")
}
```

随机执行1或者2.

### case4

```Go
func main() {
  select {
  }
}
```

对于空的 select 语句，程序会被阻塞，确切的说是当前协程被阻塞，同时 Go 自带死锁检测机制，当发现当前协程再也没有机会被唤醒时，则会发生 panic。所以上述程序会 panic。