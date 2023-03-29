### about test
TestBasicAgree2B()：最基础的追加日志测试。先使用nCommitted()检查有多少的server认为日志已经提交（在执行Start()函数之前，所有的服务器都不应该提交日志），若满足条件则调用cfg.one()，其通过调用rf.Start(cmd)来追加日志。rf.Start(cmd)用于模拟Raft实例从Client接收实例的情况。
TestRPCBytes2B：基于RPC的字节数检查保证每个cmd都只对每个peer发送一次。
TestFailAgree2B：断连小部分，不影响整体Raft集群的情况检测追加日志。
TestFailNoAgree2B：断连过半数节点，保证无日志可以正常追加。然后又重新恢复节点，检测追加日志情况。
TestConcurrentStarts2B：模拟客户端并发发送多个命令
TestRejoin2B：Leader 1断连，再让旧leader 1接受日志，再给新Leader 2发送日志，2断连，再重连旧Leader 1，提交日志，再让2重连，再提交日志。
TestBackup2B：先给Leader 1发送日志，然后断连3个Follower（总共1Ledaer 4Follower），网络分区。提交大量命令给1。然后让leader 1和其Follower下线，之前的3个Follower上线，向它们发送日志。然后在对剩下的仅有3个节点的Raft集群重复上面网络分区的过程。
TestCount2B：检查无效的RPC个数，不能过多。