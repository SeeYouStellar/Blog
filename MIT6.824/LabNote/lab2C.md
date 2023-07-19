## lab2C Persistence

### 注意点
1. 2C的测试代码中cfg.crash(server)表示的是使server宕机，而cfg.unconnect(server)表示的是形成网络分区，将该server与其他server断开网络连接，crash表示server内存中数据丢失需要使用持久化恢复，而unconnect后数据不会丢失。
2. 日志回退机制改进
   - 优化点1
    如果follower.log不存在prevLog，让Leader下一次从follower.log的末尾开始同步日志。
   - 优化点2
    如果是因为prevLog.Term不匹配，记follower.prevLog.Term为conflictTerm。
      +  如果leader.log找不到Term为conflictTerm的日志，则下一次从follower.log中conflictTerm的第一个log的位置开始同步日志。
      +  如果leader.log找到了Term为conflictTerm的日志，则下一次从leader.log中conflictTerm的最后一个log的下一个位置开始同步日志。

3. 持久化发生在一切改变voteFor,currentTerm,logs的操作之后
4. 选举逻辑的bug:
   
   ![Alt text](../image/image12.png)
   
   S0,S4 index=3处的两条日志任期也相同，代表着S4发起AppendEntries后，日志冲突检查时不会检查出冲突点，故也不会截断覆盖S0的这条日志，并且之后也会在S4更新了commitIndex后将自己的commitIndex更新为3，那么这条日志将会被提交，显然在相同的index处提交了不同的日志，无法保证一致性了。
   
    问题出现在正常情况下，S4为什么能在重连后当上leader呢？显然是获得了大多数的选票。说明选举逻辑有问题。并且不是RequestVote RPC 和 sendRequestVote这两个函数的逻辑出问题，而是一个论文中提到的很容易忽视的心跳的作用没有完全实现。

    论文中的心跳作用描述为：
    "It then sends heartbeat messages to all of the other servers to establish its authority and prevent new elections."

    如果心跳实现时只是重置选举时间，而没有将votefor改为leaderid，那么只要在S0宕机后，各个集群里的follower的选举计时剩余时间比S4重启后随机获取的选举计时多，S4就可以率先进行选举并成为leader。
    我的理解是，establish its authority的意思其实是需要将votefor改为leaderid，那么S4就无法获取多数的选票，因为follower都将votefor改为了S0