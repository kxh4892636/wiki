---
id: 5cfdd006-7b84-49c7-b287-2b46bbf52fa7
anki: true
---

# Go 的 channel 与并发安全如何保证？

## 概念索引

- channel: goroutine 之间传递值的通信管道;
- 缓冲: 决定发送是否立即完成;
- 关闭: 表示不会再发送值;
- 线程安全: 多个 goroutine 同时访问时的正确性;
- 同步原语: mutex、RWMutex、atomic、WaitGroup;

## 有缓冲与无缓冲 channel 有什么区别？

| 类型 | 创建 | 发送完成条件 | 同步语义 |
| --- | --- | --- | --- |
| 无缓冲 | make(chan T) | 有接收者配对 | 发送与接收同步交接 |
| 有缓冲 | make(chan T, n) | 缓冲区未满 | 先入缓冲，解耦双方 |
| nil channel | var ch chan T | 永远阻塞 | 不能发送或接收 |

```go
unbuffered := make(chan int)
buffered := make(chan int, 2)

buffered <- 1 // 缓冲未满，不阻塞
buffered <- 2
fmt.Println(<-buffered, <-buffered)
```

- 无缓冲更像交接点，不是队列;
- 有缓冲容量满时发送阻塞，空时接收阻塞;
- 关闭后仍可读出缓冲值，读完后返回零值与 false;

## channel 如何保证线程安全？

| 操作 | 安全性 |
| --- | --- |
| 多 goroutine 同时发送 | channel 内部同步，安全 |
| 多 goroutine 同时接收 | 安全，每个值只被一个接收者拿到 |
| 关闭与发送并发 | 可能 panic，需由协调者保证 |
| 通过 channel 传指针 | 只传递指针，底层数据仍可能被并发修改 |

```go
ch := make(chan int)
go func() { ch <- 1 }()
fmt.Println(<-ch) // channel 保证这次交接同步
```

- channel 只保护“传递”这个动作;
- 发送方在发送后继续修改同一份数据，仍会与接收方竞争;
- 关闭原则: 由发送方关闭，且确认不会再有发送;

## map 是线程安全的吗？

- 不是: 并发读写 map 会触发 data race，甚至直接 panic;
- 方案一: `sync.Mutex` / `sync.RWMutex` 保护整张 map;
- 方案二: `sync.Map`，适合读多写少或键集合稳定的场景;
- 检查: `go test -race ./...` 只能发现已执行路径上的竞争;

```go
var (
    mu sync.RWMutex
    m  = map[string]int{}
)

func Get(k string) int {
    mu.RLock()
    defer mu.RUnlock()
    return m[k]
}
```

## goroutine 之间有哪些通信方式？

| 方式 | 适用 |
| --- | --- |
| channel | 传递数据、事件、完成信号 |
| mutex / RWMutex | 保护共享可变状态 |
| atomic | 简单计数、标志位 |
| context | 取消、超时、跨调用链传值 |
| WaitGroup | 等待一组 goroutine 完成 |

- 优先通过通信共享内存，减少共享可变状态;
- 必须共享时，给数据指定唯一写入者，或所有读写走同一套锁;
- 不要让一部分路径加锁、另一部分绕开;
