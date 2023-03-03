## 任务分析

### 大致思路：

1. 整个 MR 框架由一个 Coordinator 进程及若干个 Worker 进程构成
2. Coordinator 进程与 Worker 进程间通过本地 Socket 进行 Golang RPC 通信
3. 由 Coordinator 协调整个 MR 计算的推进，并分配 Task 到 Worker 上运行
4. 在启动 Coordinator 进程时指定 输入文件名 及 Reduce Task 数量
5. 在启动 Worker 进程时指定所用的 MR APP 动态链接库文件
6. Coordinator 需要留意 Worker 可能无法在合理时间内完成收到的任务（Worker 卡死或宕机），在遇到此类问题时需要重新派发任务
7. Coordinator 进程的入口文件为 main/mrcoordinator.go
8. Worker 进程的入口文件为 main/mrworker.go
我们需要补充实现 mr/coordinator.go、mr/worker.go、mr/rpc.go 这三个文件

### 细节：

Coordinator 需要有以下功能：

1. 在启动时根据指定的输入文件数及 Reduce Task 数，生成 Map Task 及 Reduce Task
2. 响应 Worker 的 Task 申请 RPC 请求，分配可用的 Task 给到 Worker 处理
3. 追踪 Task 的完成情况，在所有 Map Task 完成后进入 Reduce 阶段，开始派发 Reduce Task；在所有 Reduce Task 完成后标记作业已完成并退出

而 Worker 的功能则相对简单，只需要保证在空闲时通过 RPC 向 Coordinator 申请 Task 并运行，再不断重复该过程即可。