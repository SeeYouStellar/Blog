在 Go 语言中，有多种用于多线程编程的工具和并发原语。以下是一些常用的工具：

### 1. Goroutines

Goroutine 是 Go 语言中的轻量级线程。通过简单的 `go` 关键字可以启动一个新的 goroutine。它们由 Go 运行时进行管理，具有低开销。

```go
package main

import (
    "fmt"
    "time"
)

func say(s string) {
    for i := 0; i < 5; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}

func main() {
    go say("world")
    say("hello")
}
```

### 2. Channels

Channels 是 Go 语言中用于在 goroutine 之间传递数据的管道。通过 `make` 函数创建，可以使用 `<-` 操作符进行发送和接收。

```go
package main

import "fmt"

func main() {
    ch := make(chan int)
    
    // 启动一个新的 goroutine
    go func() {
        ch <- 42
    }()
    
    // 从通道接收数据
    value := <- ch
    fmt.Println(value)
}
```

### 3. `sync.WaitGroup`

`sync.WaitGroup` 用于等待一组 goroutine 完成。它有三个方法：`Add`，`Done` 和 `Wait`。

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("Worker %d starting\n", id)
    // 模拟工作
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }

    wg.Wait()
}
```

### 4. `sync.Mutex` 和 `sync.RWMutex`

`sync.Mutex` 用于提供互斥锁，保护共享资源。`sync.RWMutex` 是读写锁，允许多个读操作并发，但写操作需要独占锁。

```go
package main

import (
    "fmt"
    "sync"
)

var (
    mu sync.Mutex
    balance int
)

func Deposit(amount int, wg *sync.WaitGroup) {
    defer wg.Done()
    mu.Lock()
    balance += amount
    mu.Unlock()
}

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go Deposit(100, &wg)
    }

    wg.Wait()
    fmt.Println("Final balance:", balance)
}
```

### 5. `sync.Cond`

`sync.Cond` 是条件变量，用于 goroutine 之间的通知和等待。

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var (
    mu sync.Mutex
    cond = sync.NewCond(&mu)
)

func waitForCondition() {
    mu.Lock()
    cond.Wait() // 等待条件满足
    fmt.Println("Condition met, proceeding")
    mu.Unlock()
}

func signalCondition() {
    mu.Lock()
    cond.Signal() // 唤醒一个等待的 goroutine
    mu.Unlock()
}

func main() {
    go waitForCondition()
    
    time.Sleep(1 * time.Second)
    
    go signalCondition()
    
    time.Sleep(1 * time.Second)
}
```

### 6. `sync.Once`

`sync.Once` 用于确保某些操作只执行一次。

```go
package main

import (
    "fmt"
    "sync"
)

var once sync.Once

func initialize() {
    fmt.Println("Initialized")
}

func main() {
    for i := 0; i < 10; i++ {
        go once.Do(initialize)
    }

    time.Sleep(1 * time.Second)
}
```

### 7. `context` 包

`context` 包提供了在 goroutine 树之间传递截止日期、取消信号和其他请求范围值的功能。

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    go func() {
        select {
        case <-time.After(2 * time.Second):
            fmt.Println("Operation completed")
        case <-ctx.Done():
            fmt.Println("Operation cancelled")
        }
    }()

    time.Sleep(1 * time.Second)
    cancel()

    time.Sleep(1 * time.Second)
}
```

这些工具使得 Go 语言在处理并发编程时非常强大和灵活，可以应对各种并发场景和需求。