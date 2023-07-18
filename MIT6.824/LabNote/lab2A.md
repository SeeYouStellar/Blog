## lab2A LeaderElection

要完成的任务：
- Raft instance data structure design
- RequestVote RPC design
- AppendEntries RPC design
- Make func implement
- ticker func implement
- GetState func implement


### 1. something useful

1. [make you debug easier](https://blog.josejg.com/debugging-pretty/)
2. [students-guide-to-raft](https://thesquareplanet.com/blog/students-guide-to-raft/)

### 2. code

#### 2.1 Raft data structure

内置timer用golang的time.Timer实现，这个timer有两个功能，当machine state是follower和candidate状态时，timer用来实现election timeout，当machine state是leader时，timer用来实现heartbeart timeout。

time.Timer内置一个chan（缓冲区为1），当定时结束，就会向chan里放一个time。那么用select就可以实现定时结束执行相应操作（后续make函数实现里会涉及）。Timer结构如下：
```go
type Timer struct {
   C <-chan Time 
   r runtimeTimer
}
```
Raft 实例如下：
```go
type Raft struct {
	mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	currentTerm int32
	votedFor    int
	state       int
	timer       *time.Timer
	overTime    time.Duration
	votedNum    int32

	// log         []LogEntry   // 2B 
	// commitIndex int          // 2B 
	// lastApplied int          // 2B
	// nextIndex   []int        // 2B
	// matchIndex  []int        // 2B
}

// heartbeat timeout 心跳间隔时间不变
var HeartBeatTimeout = 100 * time.Millisecond

// three states
const (
	STATE_FOLLOWER = iota
	STATE_CANDIDATE
	STATE_LEADER
)

```
#### 2.2 RequestVote RPC design

rpc.reply rpc.args的数据结构完全采用paper figure2。rpc的实现逻辑与lab1一样，即某个server调用sendxxxrpc函数，在sendxxxrpc函数里callxxxrpc调用另外一个server的xxxrpc。这里不再赘述。

RequestVote逻辑是(只是2A的逻辑，在2B需要改动)：
- 当candidate的任期（args.Term）小于当前machine的任期（rf.currentTerm），就不投票并且通过返回的reply.term来提醒调用者candidate转为follower
- 当candidate的任期（args.Term）大于当前machine的任期（rf.currentTerm），**直接投票**，并且执行**转为follower**的操作，也不需要判断是否已投给了其他人，因为如果投给了其他candidate，那么rf.term必然跟那个candidate一样，那么投给了term小的candidate相当于没投，raft只有最大的term的candidate才有机会成为leader。并且需要将选举时间重置
- 当candidate的任期（args.Term）等于当前machine的任期（rf.currentTerm），此时这个machine state有两种可能，candidate或follower，**首先如果是candidate**，因为follower在成为candidate时已经把票投给了自己，又因为term相同，显然这个candidate是与当前call rpc的candidate形成竞争关系的，所以不能投给它。**然后如果是follower的话**，同个term,需要判断有没有投给其他人，如果投了其他人就不能再投当前的candidate
  
注意：
- 任期的新旧程度是判断的第一准则，当candidate的任期大于某个server的任期时，直接执行下述代码。然后在2B中对最终是否投票还有一个选举限制，2A的话判断完任期就可以投票了。
	```go
		rf.state = STATE_FOLLOWER
		rf.currentTerm = args.Term
		rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond 
		rf.timer.Reset(rf.overTime)
	```
- 投票成功了需要重置选举时间，没有成功就不需要重置。2A没成功可能是投给了其他人或自己，投给其他则在另外的rpc中已经重置过了，投给自己当然希望自己能当选，不需要重置。
- 
```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {

	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.VoteGranted = false
	reply.Term = rf.currentTerm
	lastRfLogsIndex := len(rf.logs) - 1

	if args.Term < rf.currentTerm {
		return
	} else if args.Term > rf.currentTerm {
		// 2B添加了选举限制，2A不需要判断直接执行里面的代码
		if args.LastLogsTerm > rf.logs[lastRfLogsIndex].Term || (args.LastLogsTerm == rf.logs[lastRfLogsIndex].Term && args.LastLogsIndex >= lastRfLogsIndex) {
			rf.state = STATE_FOLLOWER
			rf.voteFor = args.CandidateId
			rf.voteNum = 0
			rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond
			rf.timer.Reset(rf.overTime)
			rf.currentTerm = args.Term
			reply.VoteGranted = true
		} else {
			rf.state = STATE_FOLLOWER
			rf.voteFor = -1
			rf.voteNum = 0
			rf.currentTerm = args.Term
			reply.Term = args.Term
			// 投票失败不需要重置选举时间
		}
	} else {
		if rf.voteFor == -1 || rf.voteFor == args.CandidateId {
			if args.LastLogsTerm > rf.logs[lastRfLogsIndex].Term || (args.LastLogsTerm == rf.logs[lastRfLogsIndex].Term && args.LastLogsIndex >= lastRfLogsIndex) {
				rf.voteFor = args.CandidateId
				rf.voteNum = 0
				rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond
				rf.timer.Reset(rf.overTime)
				reply.VoteGranted = true
			} else {
				// 投票失败不需要重置选举时间
				return
			}
		}
	}
}
```
sendRequestVote的实现逻辑：
1. 先要call rpc（rpc调用），这是与lab1一样的地方，并且这一步是不用上锁的（students-guide-to-raft里有说过，**rpc调用不上锁，rpc内部上锁**），可能是因为想尽可能地实现follower转为candidate**立刻**进行投票选举（个人猜测，其实在我的代码中rpc内部也加了锁，还是做不到同步进行投票选举），另外还有可能是rpc需要一直发送直到rpc请求成功，如果这一过程一直上了锁，自然会造成资源浪费，而如果仅仅在rpc内部上锁，调用失败马上可以释放锁，增加了并行性。
2. 比较"candidate"(引号是因为此时有可能不是candidate了)的currentTerm和之前call rpc时的term（args.Term）大小关系。如果后者大，说明这已经是之前任期的rpc请求了，因为某些原因现在才执行完返回（比如在发送rpc请求过程中已经收到了leader的心跳转为了follower，同时同步了term），**过期的rpc请求是没用的**。
3. 判断reply.VoteGranted。**并行**统计得票结果。
4. 对reply.term进行处理。其实就是对应paper中"If RPC request or response contains term T > currentTerm:set currentTerm = T, convert to follower"，这里的rpc request or response中的response其实指的就是在处理reply时也要通过reply.term判断rpc请求是否已过期（或者说当时call rpc的candidate是否已过期）
```go
func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	// 可以不用
	for ok == false {
		if rf.killed() == false {
			ok = rf.peers[server].Call("Raft.RequestVote", args, reply)
		} else {
			break
		}
	}

	rf.mu.Lock()
	defer rf.mu.Unlock()

	if args.Term < rf.currentTerm {
		return false
	}
	if reply.VoteGranted == true {
		// 获得了选票
		rf.voteNum += 1
		if int(rf.voteNum) >= (len(rf.peers)/2)+1 {
			rf.state = STATE_LEADER
			rf.timer.Reset(0)
			for j := 0; j < len(rf.peers); j++ {
				rf.nextIndex[j] = len(rf.logs)
				rf.matchIndex[j] = 0
			}
		}
	} else {
		// 未获得选票
		if reply.Term > rf.currentTerm {
			rf.state = STATE_FOLLOWER
			rf.currentTerm = reply.Term
			rf.voteFor = -1
			rf.voteNum = 0
		} else {
			// (2B)日志不够新 ->do nothing, 只要和有大多数比够新就行，和一个比不够新还不足以让这个candidate变为follower
			// (2A)投给其他人 ->do nothing
		}
	}
	return true
}
```
#### 2.3 AppendEntries RPC design

逻辑和RequestVote是一样的。实现的细节看注释。

注意：
- AppendEntries RPC有两个功能，2A只实现心跳，2B会实现日志复制。两个功能是一起完成的，意思是每个AppendEntries RPC都会起到心跳作用，有些还能起到日志复制作用。
- 只要任期判断通过，那么就需要重置选举时间并切换state为follower
  
```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	reply.Success = false

	// (2A)判断任期新旧
	// 接下来判断的所有分支都需要重置选举时间以及改变state为follower
	if args.Term < rf.currentTerm {
		return
	}
	
	// (2B) 检测args.Entries日志块的前一条日志是否匹配
	if (args.PrevLogsIndex < len(rf.logs) && rf.logs[args.PrevLogsIndex].Term != args.PrevLogsTerm) || args.PrevLogsIndex >= len(rf.logs) {
		rf.state = STATE_FOLLOWER
		rf.voteNum = 0
		rf.voteFor = -1
		// 虽然不匹配，但是因为任期比leader小，所以还是要重置选举时间
		// 相当于这个AppendEntries RPC只起到了心跳包的作用
		rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond 
		rf.timer.Reset(rf.overTime)
		rf.currentTerm = args.Term
		reply.Term = args.Term
		return
	}

	// (2B) 按照paper的要求，按下标遍历寻找冲突点，找到后进行截断覆盖
	i := 0
	for i = 1; i <= len(args.Entries); i++ {
		if len(rf.logs) > args.PrevLogsIndex+i && rf.logs[args.PrevLogsIndex+i].Term != args.Entries[i-1].Term {
			rf.logs = rf.logs[:args.PrevLogsIndex+i] to 1
			tmp := args.Entries[i-1:]
			rf.logs = append(rf.logs, tmp...)
			break
		} else if len(rf.logs) == args.PrevLogsIndex+i {
			tmp := args.Entries[i-1:]
			rf.logs = append(rf.logs, tmp...)
			break
		}
	}

	// (2B) 更新rf.commitIndex
	if args.LeaderCommit > rf.commitIndex {
		if args.LeaderCommit > len(rf.logs)-1 {
			rf.commitIndex = len(rf.logs) - 1
		} else {
			rf.commitIndex = args.LeaderCommit
		}
	}

	rf.state = STATE_FOLLOWER
	rf.currentTerm = args.Term
	rf.voteNum = 0
	rf.voteFor = -1
	rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond
	rf.timer.Reset(rf.overTime)
	reply.Success = true
}

func (rf *Raft) sendAppendEntries(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	for ok == false {
		if rf.killed() == false {
			ok = rf.peers[server].Call("Raft.AppendEntries", args, reply)
		} else {
			break
		}
	}

	rf.mu.Lock()
	defer rf.mu.Unlock()
	if args.Term < rf.currentTerm {
		return false
	}

	if reply.Success == false {
		if reply.Term > rf.currentTerm {
			// (2A) 遇到更新的term
			rf.state = STATE_FOLLOWER
			rf.currentTerm = reply.Term
			rf.voteFor = -1
			rf.voteNum = 0
			rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond
			rf.timer.Reset(rf.overTime)
		} else {
			// (2B) 前一条日志不匹配，需要向前寻找匹配点
			rf.nextIndex[server] -= 1
		}
	} else {
		// (2B)日志复制成功，更新leader的相关配置
		rf.nextIndex[server] = rf.nextIndex[server] + len(args.Entries)
		rf.matchIndex[server] = rf.nextIndex[server] - 1

		for i := len(rf.logs) - 1; i >= rf.commitIndex+1; i-- {
			cnt := 0
			for j := range rf.peers {
				if rf.matchIndex[j] >= i && rf.logs[i].Term == rf.currentTerm {
					cnt += 1
				}
			}
			if cnt >= (len(rf.peers)/2)+1 {
				rf.commitIndex = i
				break
			}
		}
	}
	return true
}
```
#### 2.4 Make func

创建Raft实例的入口函数。包含基本的初始化工作以及启动定时检测的函数ticker。

```go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {

	rand.Seed(time.Now().UnixNano())

	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	// (2A)
	rf.currentTerm = 0
	rf.voteFor = -1
	rf.state = STATE_FOLLOWER
	rf.overTime = time.Duration(200+rand.Intn(250)) * time.Millisecond
	rf.timer = time.NewTimer(rf.overTime)

	// (2B)
	rf.nextIndex = make([]int, len(rf.peers))
	rf.matchIndex = make([]int, len(rf.peers))
	rf.logs = make([]LogEntry, 1)
	rf.commitIndex = 0
	rf.lastApplied = 0
	rf.applyMsgchan = applyCh

	go rf.ticker()
	// (2B) 日志提交线程
	go rf.ApplyLogs()
	return rf
}
```

#### 2.5 ticker func

ticker函数是定时检测函数，作为一个长期存在的协程对每个machine进行超时检测。

三个state的machine在每个state都执行相同的操作。**因为整个过程都是上锁的，所以不用担心在执行相应操作时发生状态改变**。想了很久，这里也只能通过整个上锁来处理。

代码基本是按照paper的figure2来实现的，特别是rules for server部分。

```go
func (rf *Raft) ticker() {

	time.Sleep(5 * time.Millisecond) // wait tester to write "test(2A)"

	for rf.killed() == false {

		// Your code here to check if a leader election should
		// be started and to randomize sleeping time using
		// time.Sleep().
		select {
		case <-rf.timer.C:
			rf.mu.Lock()
			switch rf.state {
			case STATE_FOLLOWER:
				rf.state = STATE_CANDIDATE
				fallthrough // 保证选举时限一到就转变为candidate进行选举
			case STATE_CANDIDATE:
				rf.currentTerm += 1
				rf.overTime = time.Duration(350+rand.Intn(150)) * time.Millisecond
				rf.timer.Reset(rf.overTime)
				rf.voteFor = rf.me
				rf.voteNum = 1
				for i := 0; i < len(rf.peers); i++ {
					if i == rf.me {
						continue
					}
					reply := RequestVoteReply{}
					args := RequestVoteArgs{
						Term:          rf.currentTerm,
						CandidateId:   rf.me,
						LastLogsIndex: 0,
						LastLogsTerm:  0,
					}
					// (2B)
					if len(rf.logs) > 1 {
						args.LastLogsIndex = rf.logs[len(rf.logs)-1].Index
						args.LastLogsTerm = rf.logs[len(rf.logs)-1].Term
					}
					go rf.sendRequestVote(i, &args, &reply)
				}
			case STATE_LEADER:
				rf.timer.Reset(HeartBeatTimeout)
				for j := range rf.peers {
					if j == rf.me {
						continue
					}
					// commom heartbeat
					reply := AppendEntriesReply{}
					args := AppendEntriesArgs{
						Term:          rf.currentTerm,
						LeaderId:      rf.me,
						PrevLogsIndex: 0,
						PrevLogsTerm:  0,
						Entries:       nil,
						LeaderCommit:  rf.commitIndex,
					}
					// (2B)
					if len(rf.logs) > 1 {

						if rf.nextIndex[j] >= 1 && rf.nextIndex[j] <= len(rf.logs) {
							args.PrevLogsIndex = rf.nextIndex[j] - 1
							args.PrevLogsTerm = rf.logs[args.PrevLogsIndex].Term
							if rf.nextIndex[j] < len(rf.logs) {
								args.Entries = rf.logs[rf.nextIndex[j]:]
							} else {
								args.Entries = nil
							}
						} else if rf.nextIndex[j] < 1 {
							args.PrevLogsIndex = 0
							args.PrevLogsTerm = 0
							args.Entries = rf.logs
						} else if rf.nextIndex[j] > len(rf.logs) {
							args.PrevLogsIndex = rf.logs[len(rf.logs)-1].Index
							args.PrevLogsTerm = rf.logs[len(rf.logs)-1].Term
							args.Entries = nil
						}
					}
					go rf.sendAppendEntries(j, &args, &reply)
				}

			}
			rf.mu.Unlock()
		}
		time.Sleep(10 * time.Millisecond)
	}
}
```

### 3. about test

这部分的测试感觉比较水。主要考察的是正常情况下集群是否能很快选出leader和如果leader宕机集群能否快速选出leader，调整好选举时间，心跳时间的范围就可以

**单次测试**
![single test](../image/image9.png)

**多次测试**
![multiple test](../image/image10.png)

### 4. problem

1. 两个candidate（term相同）几乎同时结束election timeout，其中一个成为了leader, 其在sendappendentries过程中，如果还未来得及抢占到锁并对rpc reply进行处理（假设正好要将另一个candidate转化为follower），而恰巧此时是这个candidate正在用锁，那么它就会以为是选举超时未成为leader, 就会把term++，此时他就拥有最大的term。那么这种是不是不被允许的，是代码逻辑的问题吗？
raft算法需要考虑因为代码实现时，**共享内存必然会造成的线程竞争问题**。你永远不知道哪个线程先抢到锁。这个问题描述的情况就是：线程在sendappendentries函数中call appendeneries rpc，在rpc中执行心跳发送机制，也就是将所有term<=leader.term的machine转为follower状态，整个rpc都是上锁的，同时candidate重新选举也是需要加锁，那么这两部分代码都需要上锁，自然谁先谁后都有可能，那么上述的问题也会出现。但是我后来觉得，这种情况其实就是raft允许的，**此时那个term大的candidate有可能成为leader，而那个term小的leader再发送完心跳并收到reply后，就知道有更大的term，就会转变为follower，也即是说，raft是对这种情况有防备的。**
