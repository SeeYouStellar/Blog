## Lab3A Key/value service without snapshots

### 一、运行原理
![Alt text](../image/image19.png)

### 二、请求的运行链路
图中绿色部分代表的是一个请求从clerk到leader节点，经过预写日志同步，再应用到状态机，最后返回的过程。文字描述如下：

1. clerk 发起请求，调用 put/append/get 的 rpc handler，rpc handler 的返回是同步的，故clerk等待其返回
2. rpc handler 将 put/append/get 指令与 clerk 信息结合，生成日志条目，调用 raft 的 start(cmd) 接口。start 返回日志 index 信息，rpc handler 根据生成属于该 index 的 agreech, 并阻塞等待，等待时间超过2000ms即返回超时，告知 clerk 请求失败
3. start(cmd) 将日志写入 leader 的 WAL(预写日志)中
4. leader 节点在发心跳(appendEntries)的过程中，将新的日志传送给 follower 节点
5. 日志同步逻辑：半数follower日志同步完成(写入 WAL 中)，appendEntries 返回成功，leader节点根据多数派原则更新 commitIndex 和 lastApplied ，在下次发心跳时，将 leaderCommit 告知 follower ，follower也会更新commitIndex 和 lastApplied 。
异步applyLog:发现 commitIndex < lastApplied, 将日志已提交信息 applymsg 发送至管道 applych 
异步waitAgreee:监听到管道内的 applymsg ，将其中的日志部分提取出后，在保证**客户端请求幂等性**的前提下，将日志应用到状态机中。并往属于该下标日志的管道 agreeCh 中写入应用结果（实际上传回日志即可）
6. 阻塞等待的 rpc handler 返回请求成功，clerk收到后即表示一个完整的请求过程结束

图中玫红色部分代表的是 follower 节点中的日志同步、提交、应用的过程。注意 follower 节点是没有 put/append/get rpc 的调用过程的

### 三、接口的幂等性

同一个客户端重复请求同一个接口，server 要将这多个请求看成一个请求。比如，当某个请求因为日志没有即时同步，客户端重复发出这个请求。那么也会同步多个内容相同日志。

**处理方法：**

- 每个客户端保存自己的客户端编号clerkid和请求数seqid。每个client发送请求时，附带上clerkid和seqid
- server在将请求转换为日志时，日志内容附带上clerkid和seqid
- 每个server保存所有客户端已提交日志的最大seqid，用```lastSeq map[int]int```实现
- 在日志提交信息通过 applych 传送给 server 时，判断日志内容里的seqid是否大于lastSeq[clerkid]，若大于则给与应用，若等于则代表这是这条指令已经应用了，是重复指令，不予应用。不会发生小于的情况。这一步称为幂等性判断

### 四、多客户端请求


针对但客户端：
```go
_, _, isLeader := kv.rf.Start(op) // push op to raft layer to reach the agreement
if !isLeader {
    reply.Err = ErrWrongLeader
    return
}
<- applyCh
// agreed! apply to statemachine
kv.apply(op)
```
单客户端这样做是没有问题的。但是单个客户端请求是串行发送，多个客户端时请求是并行发送。而 ```applych``` 只有一个。多个请求发起的rpc handler都在等待applych传回日志提交信息，但是他们无法知道传回的到底是不是之前start(op)里的op同步成功了。

那是否可以通过循环等待applych传回的信息并判断，这个想法大错特错，因为你要判断，就要接受管道里的信息，这个信息一旦接受，如果判断为不是符合要求的op，那么真正需要这个信息的rpc handler就永远无法获取这个信息了。管道的理解不够深才会有这个想法。我太菜了。

**处理方法：**

每个日志index建立一个管道，用来通知发送该下标日志请求的rpc handler。rpc handler在start(op)后就等待这个管道。

另起一个waitAgree线程用来接受applych中的数据，然后判断幂等性，然后应用。**注意应用日志必须与接受applymsg串行进行，如果并行的话（比如说把应用日志放在rpc handler里完成），有可能会日志应用的顺序出错，即后发送applymsg的日志比前发送的日志先进行了日志应用操作（applyLogToStateMachine）**


### 五、代码实现

#### 5.1 client

1）客户端结构体：

```go
type Clerk struct {
	servers []*labrpc.ClientEnd
	lastLeader int  // 上一个请求成功时leaderId
	clerkId    int  // 客户端id
	lastSeq    int  // 该客户端发起的最后一个写请求号，从0开始
	mu         sync.Mutex 
}
```

2）发起get请求：
如果没有响应成功，需要一直发起请求知道成功。

```go
func (ck *Clerk) Get(key string) string {
	args := GetArgs{
		Key:     key,
		ClerkId: ck.clerkId,
		Seq:     ck.lastSeq,
	}
	reply := GetReply{}
	i := ck.lastLeader
	defer func() {
		ck.mu.Lock()
		ck.lastLeader = i
		ck.mu.Unlock()
	}()
	for {
		ok := ck.servers[i].Call("KVServer.Get", &args, &reply)
		if !ok || reply.Err == ErrWrongLeader {
			i = (i + 1) % len(ck.servers)
			time.Sleep(10 * time.Millisecond)
			continue
		}
		if reply.Err == ErrNoKey {
			return ""
		}
		return reply.Value
	}
}
```

3）发起put/append请求：
```go
func (ck *Clerk) PutAppend(key string, value string, op string) {
	// You will have to modify this function.
	ck.mu.Lock()
	ck.lastSeq += 1
	ck.mu.Unlock()

	args := PutAppendArgs{
		Key:     key,
		Value:   value,
		OpPA:    op,
		ClerkId: ck.clerkId,
		Seq:     ck.lastSeq,
	}
	reply := PutAppendReply{}

	i := ck.lastLeader
	defer func() {
		ck.mu.Lock()
		ck.lastLeader = i
		ck.mu.Unlock()
	}()

	for {
		// fmt.Printf("clerk %d send cmd to server层 %d\n", ck.clerkId, i)
		// fmt.Println(args)
		ok := ck.servers[i].Call("KVServer.PutAppend", &args, &reply)
		if !ok || reply.Err == ErrWrongLeader {
			i = (i + 1) % len(ck.servers)
			time.Sleep(10 * time.Millisecond)
			continue
		}
		return
	}
}
```

#### 5.2 server

1）日志条目Op
```go
type Op struct {
	OpPA    string
	Key     string
	Val     string
	ClerkId int
	Seq     int
}
```

2）KVServer
```go
type KVServer struct {
	mu      sync.Mutex
	me      int
	rf      *raft.Raft
	applyCh chan raft.ApplyMsg
	dead    int32 // set by Kill()
	maxraftstate int // snapshot if log grows this big
	// Your definitions here.
	kvdb     map[string]string
	lastSeq  map[int]int     // 记录每个clerk已经提交的日志的序列号（op中的序列号，与日志下标不一样）
	agreeChs map[int]chan Op // 每个日志下标对应的通知rpc结束的管道
}
```

3）Get rpc
```go
func (kv *KVServer) Get(args *GetArgs, reply *GetReply) {
	// Your code here.
	cmd := Op{
		OpPA:    "Get",
		Key:     args.Key,
		ClerkId: args.ClerkId,
		Seq:     args.Seq,
	}

	index, _, isLeader := kv.rf.Start(cmd)
	if !isLeader {
		reply.Err = ErrWrongLeader
		return
	}
	ch := kv.getAgreeCh(index)
	var cmd_ Op
	select {
	case cmd_ = <-ch:
		close(ch)
	case <-time.After(1000 * time.Millisecond): // 超时重试
		reply.Err = ErrWrongLeader
		return
	}
	if !isSameOp(cmd_, cmd) {
		reply.Err = ErrWrongLeader
		return
	}
	reply.Value = kv.kvdb[args.Key] // 读请求不涉及数据库的更新，直接读
	reply.Err = OK
	return
}
```

4）PutAppend rpc
```go
func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
	cmd := Op{
		OpPA:    args.OpPA,
		Key:     args.Key,
		Val:     args.Value,
		ClerkId: args.ClerkId,
		Seq:     args.Seq,
	}
	index, _, isLeader := kv.rf.Start(cmd)
	if !isLeader {
		reply.Err = ErrWrongLeader
		return
	}
	ch := kv.getAgreeCh(index)
	var cmd_ Op
	select {
	case cmd_ = <-ch:
		close(ch)
	case <-time.After(1000 * time.Millisecond): // 超时重试
		reply.Err = ErrWrongLeader
		return
	}
	// 网络分区修复前，某个term更小的leader正好start(cmd)，然后cmd还未被同步完成即还未提交，
	// 网络分区修复后，新的cmd覆盖了这个index上原有的cmd，然后通过applych传回了该cmd，
	// 此时该rpc handler发起时的需同步的cmd与applych传回的不是同一个cmd，之前的需同步的cmd实际未同步，所以需要告知clerk重试
	if !isSameOp(cmd_, cmd) { // 所以需要判断cmd_与cmd是否相同
		reply.Err = ErrWrongLeader
		return
	}
	reply.Err = OK
	return
}
```

5）WaitAgree，getAgreeCh，ApplyToStateMachine
```go
func (kv *KVServer) WaitAgree() {
	for !kv.killed() {
		select {
		case msg := <-kv.applyCh:
			cmd := msg.Command.(Op)
			kv.mu.Lock()
			lastSeq, ok := kv.lastSeq[cmd.ClerkId]
			if !ok || lastSeq < cmd.Seq {
				kv.ApplyToStateMachine(cmd) // 同步进行
				kv.lastSeq[cmd.ClerkId] = cmd.Seq
			}
			kv.mu.Unlock()
			kv.getAgreeCh(msg.CommandIndex) <- cmd
		}
	}
}
// 获取每个index的channel，没有则新建一个
func (kv *KVServer) getAgreeCh(idx int) chan Op {
	kv.mu.Lock()
	defer kv.mu.Unlock()

	ch, ok := kv.agreeChs[idx]
	if !ok {
		ch = make(chan Op, 1)
		kv.agreeChs[idx] = ch
	}
	return ch
}
func (kv *KVServer) ApplyToStateMachine(cmd Op) {
	op := cmd.OpPA // 命令操作类型
	switch op {
	case "Put":
		kv.kvdb[cmd.Key] = cmd.Val
	case "Append":
		kv.kvdb[cmd.Key] += cmd.Val
	}
}
```

### 六、总结

在做lab2时，有很多时间在看测试代码，看的头疼，其实lab2的测试代码里的请求发送代码逻辑与lab3A实现的差不多。做完3A确实对整个基于raft的应用如何运行有了全面的理解。虽然这里的kv只是一个map。