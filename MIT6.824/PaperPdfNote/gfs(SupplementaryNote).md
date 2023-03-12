### 分布式存储系统设计的困境

要提升访问的效率（网络带宽是瓶颈），就需要分片存储，将文件的不同部分放在不同的机器上，这样多个客户端就可以并发的进行读写。但是一旦其中某个机器宕机，那么文件的某部分就消失了，所以需要使用多副本的技术，将文件的每个部分的多个副本存在多个机架中（注意这里如果存在同个机架，同时宕机的概率还是很大）。一旦多副本技术引入后，数据的一致性就会出现问题，比如其中存储其中一个副本的机器宕机了，再次重启后，副本之间很明显数据不一致了，那么一旦你需要去考虑数据一致性的问题，就不得不通过网络的交互，比如通过一些轮询的方式获取到副本失效的信息，并通过一些方式重新克隆一个副本，这将不可避免地降低系统的性能。

以上描述的就是分布式系统性能与一致性的相互制衡。


### master节点中的数据结构

#### data structure in memory 
* **map1**: filename->chunk handles(可以理解为chunk ID)
* **map2**: chunk handles->chunk message
  * **chunk message** :
      * replica locations\[3]: chunk的每个副本位于哪个chunk server
      * chunk version: chunk的版本号，在chunk和master中都要存储这个信息，并且在每次 grant a lease 后version number会自增。主要用来Stale Replica Detection(section 4.5)
      * primary chunk location: primary chunk就是签订了租约的chunk。mutation order发生时，client需要对master发起primary chunk 位置请求（当然大多数情况client会缓存primary chunk location），这是因为gfs独特的控制流数据流分离策略要求控制流要经过primary chunk
      * lease expiration: 租约过期时限
* **map3-namespace**: filename->file metadata。paper里说master里没有全局路径树数据结构，但有一个类似这样的映射。并且每个
这些数据既然存储在内存中，那么当master宕机时这些数据就会消失，所以所有metadata更改操作都会记录在operation log中，operation log 到达一定大小就会生成checkpoint

#### data structure in local disk

* **map1** 要保存在磁盘上。我给它标记成NV（non-volatile, 非易失），这个标记表示对应的数据会写入到磁盘上。
* **Chunk replica locations**不用保存到磁盘上。因为Master节点重启之后可以与所有的Chunk服务器通信，并查询每个Chunk服务器存储了哪些Chunk，所以我认为它不用写入磁盘。所以这里标记成V（volatile），
* **chunk version** 要不要写入磁盘取决于GFS是如何工作的，我认为它需要写入磁盘。我们之后在讨论系统是如何工作的时候再详细讨论这个问题。这里先标记成NV。
* **primary chunk location** 可以不用写入磁盘，因为Master节点重启之后会忘记谁是主Chunk，它只需要等待60秒租约到期，那么它知道对于这个Chunk来说没有主Chunk，这个时候，Master节点可以安全指定一个新的主Chunk。所以这里标记成V。
* **lease expiration**也不用写入磁盘，所以这里标记成V。


### read opearation

![read operation](../image/image5.png)

首先要理清一个前提，就是读操作不需要去修改master中的metadata，所以operation log不记录读操作。但是读的chunk的read lock还是要锁上的。

### mutation

![mutation](../image/image6.png)
**mutation** is an operation that changes the contentes or metadata of a chunk such as a write or an append operation(section 3.1)

### when to write the operation log 

metadata modify

chunk modify
  
### tolerant

#### master

#### chunk 

#### chunk server

### read/write locking 

只保证