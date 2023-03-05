## 任务分析

### 前置信息：

1. 整个 MR 框架由一个 Coordinator 进程及若干个 Worker 进程构成
2. Coordinator 进程与 Worker 进程间通过本地 Socket 进行 Golang RPC 通信
3. 由 Coordinator 协调整个 MR 计算的推进，并分配 Task 到 Worker 上运行
4. 在启动 Coordinator 进程时指定 输入文件名 及 Reduce Task 数量
5. 在启动 Worker 进程时指定所用的 MR APP 动态链接库文件
6. Coordinator 需要留意 Worker 可能无法在合理时间内完成收到的任务（Worker 卡死或宕机），在遇到此类问题时需要重新派发任务
7. Coordinator 进程的入口文件为 main/mrcoordinator.go，Worker 进程的入口文件为 main/mrworker.go，需要补充实现 mr/coordinator.go、mr/worker.go、mr/rpc.go 这三个文件

### 关于测试

#### early_exit.sh

测试时会将把第一个worker退出时的mr-out-*作为mr的最终输出。

测试目的在于reduce任务没做完前，所有worker是不能退出的。

所以只要保证在最后一个reduce任务被做完前所有worker都不能退出即可。

我一开始实现时没过1s master 都会去检查每个任务的状态，根据任务的状态（所有maptask时否做完，或者所有reducetask是否做完）来改变整个mr任务运行的状态。出现worker提前退出的情况：比如上一次master检查是第9s（进入睡眠状态），执行最后(时间上)一个maptask的worker向master发起TaskDone的rpc请求，更新了这个maptask的状态（比如在第9.3s）。此时master可能没有马上发现。master要在第10s才会再次检查，然后把mr的状态改为1（0表示可获取maptask, 1表示可获取reducetask，2表示mr任务结束），但是在这1s内，worker可能会多次发起GetTask的rpc请求（假设申请maptask），此时明显无法获取任务（就不会获取到filepath），但是rpc请求是成功的，那么worker执行时，打开文件时就会有运行时报错（can't open file），worker就被动退出了。

解决方法：
一开始我是想直接在执行文件打开操作时加try-catch的，但是后来想想太蠢了这样。所以我给mr任务设定了第四种状态3，**表示上一个状态结束master还未来得及改变**，worker知道这个状态后直接执行等待，然后再发起请求

#### crash.sh

