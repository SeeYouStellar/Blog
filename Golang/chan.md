
### `chan`

`chan` 表示一个双向通道，可以用来发送和接收数据。创建和使用通道的基本语法如下：

```go
ch := make(chan int)  // 创建一个可以发送和接收 int 类型数据的通道
ch <- 42              // 向通道发送数据
value := <-ch         // 从通道接收数据
```

### `<-chan`

`<-chan` 表示一个只读通道，只能从通道接收数据，不能向通道发送数据。这通常用于函数参数，明确限制该函数只能读取通道数据，不能修改通道状态。

```go
func receiveOnly(ch <-chan int) {
    value := <-ch  // 只能从通道接收数据
    fmt.Println(value)
}
```

### `chan<-`

与 `<-chan` 相对，`chan<-` 表示一个只写通道，只能向通道发送数据，不能从通道接收数据。同样，这种用法通常用于函数参数，限制该函数只能向通道发送数据。

```go
func sendOnly(ch chan<- int) {
    ch <- 42  // 只能向通道发送数据
}
```

### 具体示例

以下是一个完整的示例，展示了 `chan`、`<-chan` 和 `chan<-` 的用法：

```go
package main

import (
    "fmt"
)

func sendOnly(ch chan<- int) {
    ch <- 42
}

func receiveOnly(ch <-chan int) {
    value := <-ch
    fmt.Println("Received:", value)
}

func main() {
    ch := make(chan int)

    go sendOnly(ch)
    receiveOnly(ch)
}
```

在这个示例中：

1. `sendOnly` 函数接受一个只写通道 `chan<- int`，只能向通道发送数据。
2. `receiveOnly` 函数接受一个只读通道 `<-chan int`，只能从通道接收数据。
3. `main` 函数创建了一个双向通道 `ch`，并分别将其传递给 `sendOnly` 和 `receiveOnly` 函数，以演示数据从发送端到接收端的传递过程。
