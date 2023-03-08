### 设计初衷

在多台机器上存储文件，并且获得巨大的性能(performance)提升

#### Performance: 
1. read
2. overwrite
3. append

#### 循环：
performance(性能) -> sharding(分片) -> fault(出错) -> tolerance(容错) -> replication(副本) -> inconsistence(不一致性)

inconsistence -> consistence 需要花费额外的网络交互资源 -> low performance
