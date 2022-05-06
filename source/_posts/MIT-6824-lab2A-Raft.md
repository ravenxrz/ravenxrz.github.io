---
title: MIT-6824-lab2A-Raft
categories: MIT6824
date: 2022-02-22 14:50:31
tags:
---

## 0. 要求

本次lab为raft的第一分部(PartA), 原文要求[在这里](http://nil.csail.mit.edu/6.824/2020/labs/lab-raft.html)

简述lab2A的要求为：

> 实现Raft中leader election和心跳，保证一个term内只有一个leader，并且leader可通过心跳保证其 leader 角色。 如果发生leader crash或者leader网络丢失，需要达到新leader自动take over旧leader。

所有修改均在 `raft.go` 文件内。

这里简述下raft目录下的其他文件的作用：

```
.
├── config.go		-- 测试时会使用到该文件，用于模拟各种场景
├── persister.go	-- 持久化raft state
├── raft.go			-- raft算法
├── test_test.go	-- 单元测试
└── util.go			-- 工具类，目前只包含了一个 DPrintf, 辅助调试
```

<!--more-->

## 1. 实现

**本文并非最终实现，最终实现在[这里](https://ravenxrz.github.io/archives/d16f195.html)**

raft实现是相对复杂的，不过好在**raft paper figure 2**给出了算法的详细描述。lab2A的所有实现，都参考自paper的figure:

<img src="https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220222145423023.png" alt="image-20220222145423023" style="zoom:50%;" />

### 状态转换

![](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/raft状态转换.svg)

### 数据结构

raft相关的数据结构，paper中已经写出。只用转化为go代码即可：

```go
type RaftRole uint8

// Raft role for each raft server
const (
	FOLLOWER RaftRole = 1 << iota
	CANDIDATE
	LEADER
)

// log entry contains command and term information
type LogEntry struct {  // NOTE: may be modified in next labs
	Term    int // term when entry was received by leader
	Command interface{}
}

//
// A Go object implementing a single Raft peer.
//
type Raft struct {
	mu             sync.Mutex          // Lock to protect shared access to this peer's state
	roleChangeCond *sync.Cond          // used for election thread, wait role be changed to `FOLLOWER` or 'CANDIDATE`
	peers          []*labrpc.ClientEnd // RPC end points of all peers
	persister      *Persister          // Object to hold this peer's persisted state
	me             int                 // this peer's index into peers[]
	dead           int32               // set by Kill()
	role           RaftRole            // who am i?

	// Persistent states
	currentTerm int
	votedFor    int // vote for which candidate? (represent in `candidateId`), -1 means no vote before
	log         []LogEntry

	// Volatile states
	// on all servers
	commitIndex      int // index of hightest log entry that known to be committed
	lastAppliedIndex int // index of highest log entry applied to state machine

	// on leaders
	nextIndex  []int // for peers, index of the next log entry to send to that server
	matchIndex []int // for peers, index of highest og entry known to be replicated on server. NOTE: "2022-2-21: 目前认为是每次心跳replica log后，peer ack后的确定id"

	// timer settings
	electionTimeout int // ms, changed in `Candidate`(send request and reset timer), `Followers when receive heartbeat, or grant vote for candidate in "RequestVote RCP"`
}
```

其它的和RPC相关结构体就不贴出了，只用将paper中的说明转为struct即可。

### 定时器

在lab2A中，每个raft instance可能含有两个定时器，一个是electionTimer, 一个是成为leader后，需要定时发送心跳的 heartbeatTimer. 

#### electionTimer

每个follower和candidate在定时器到时候，会重新开启一轮选举（对于follow来说，需要先转化为candidate）， 如果在定时器还未超时内，收到了来自leader的心跳信息，需要重置定时器时间，重新开始定时。 另外，为了避免 **split vote** 反复发生，对每个raft instance来说，需要采用随机化定时器超时时间。

ok，**第一个问题是，如何在定时器还没有到达时间时，重置定时器时间？**官方的推荐做法时是使用 `time.Sleep`来实现，根据这一个提示，我的做法是：假设总超时时间为1s， 单次sleep的时间设定为200ms，如果睡眠了5次没有收到任何leader的心跳信息，则发起选举，否则重置睡眠时间。

此部分代码如下：

```go
for {
    ...
    if rf.electionTimeout <= 0 {
        // fire election
        rf.fireElection()  // will reset inside this func
        continue
	}
    rf.electionTimeout -= napTime

    time.Sleep(napTime * time.Millisecond)
    ...
}

```

**第二个问题是，只有follower和candidate需要发起选举，而raft instance的role是在改变的，如果变成leader后，就不再需要选举。**所以加入条件变量：

```go
for {
    ...
    rf.mu.Lock()
    for rf.role == LEADER && !rf.killed() {
        rf.roleChangeCond.Wait()
    }
    if rf.killed() {
        rf.mu.Unlock()
        break
    }
    ...
}

```

除此外，这里还加入了 `killed`后，退出routine的逻辑。

**第三个问题是，如何随机化时间?**

我采用如下公式来随机化：

```go
 The real electiontimeout = baseElectionTimeout + rand.Int(minRandDis, maxRandDis)
```

具体参数设定如下：

```go
const baseElectionTimeout = 80 // ms, this this as network latency
// random Disturbance for election timeout.
const minRandDis = 400
const maxRandDis = 700
```

下面是electionTimer的代码：

```go
func (rf *Raft) electionTimer() {
	const napTime = 100
	if napTime >= rf.electionTimeout {
		log.Panicf("electionTimer napTime(%v) shouldn't be less than rf.electionTimeout(%v)", napTime, rf.electionTimeout)
	}
	DPrintf("[%d] electionTimer start...\n", rf.me)
	timerId := time.Now().Unix()
	for {
		rf.mu.Lock()
		for rf.role == LEADER && !rf.killed() {
			rf.roleChangeCond.Wait()
		}
		if rf.killed() {
			rf.mu.Unlock()
			break
		}
		// assert rf.role == FOLLOWER || rf.role == CANDIDATE
		if rf.electionTimeout <= 0 {
			// fire election
			rf.fireElection()
			rf.mu.Unlock()
			continue
		}
		rf.electionTimeout -= napTime
		rf.mu.Unlock()

		time.Sleep(napTime * time.Millisecond)
	}
	DPrintf("[electionTimeId %d] exits\n", timerId)
}

```

#### heartBeatTimer

heartBeatTimer相对electionTimer来说，整体逻辑类似，代码如下：

```go
func (rf *Raft) heartBeatTimer() {
	DPrintf("[%d] heartBeat start...\n", rf.me)
	timerId := time.Now().Unix()
	for {
		rf.mu.Lock()
		for rf.role != LEADER && !rf.killed() {
			rf.roleChangeCond.Wait()
		}
		rf.mu.Unlock()
		if rf.killed() {
			break
		}

		time.Sleep(heartBeatTimeout * time.Millisecond)
		// fire
		DPrintf("[hb id %d] fireAppendEntriesRPC\n", timerId)
		rf.fireAppendEntiresRPC()
	}
	DPrintf("[hb id %d] exits\n", timerId)
}
```

### 选举leader实现

现在回到electionTimer中的`fireElection`, 该函数为具体选举的具体实现， 算法步骤如下：

1. 如果是follower，首先转换状态为candidate
2. 增加currentTerm
3. 为自己投票
4. 重置election Timer
5. 给所有peers发起 `RequsstVote` RPC

对应论文中的：

![image-20220222152021461](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20220222152021461.png)

所以代码如下：

```go
// NOTE: must be guarded by rf.mu
func (rf *Raft) fireElection() {
	if rf.role == FOLLOWER {
		rf.changeRoleTo(CANDIDATE)
	}
	DPrintf("[%d] current role %v, fire a new election\n", rf.me, rf.role)

	// start new election
	rf.currentTerm++
	DPrintf("[%d] currentTerm++", rf.me)
	rf.votedFor = rf.me
	rf.resetElectionTimer()
	// send RequestVote to peers
	go func() {
		var curTerm int
		// send
		// build args, actually if we move `build args` into outside, we can avoid one lock acquire
		rf.mu.Lock()
		curTerm = rf.currentTerm
		lastLogTerm := -1
		if len(rf.log) != 0 {
			lastLogTerm = rf.log[len(rf.log)-1].Term
		}
		args := &RequestVoteArgs{
			Term:         rf.currentTerm,
			CandidateId:  rf.me,
			LastLogIndex: len(rf.log),
			LastLogTerm:  lastLogTerm,
		}
		rf.mu.Unlock()

		ch := make(chan RequestVoteReply)
		go rf.doSendRequestVote(args, ch)

		// receive
		voteNum := 1 // vote for myself
		maxTermFromRsp := 0
		done := make(chan bool)
		// receiver
		go rf.doReceiveRequestVoteReply(ch, done, &maxTermFromRsp, &voteNum)

		// wait
		<-done
		if rf.killed() {
			return
		}
		// update if need
		rf.mu.Lock()
		if rf.currentTerm < maxTermFromRsp {
			rf.currentTerm = maxTermFromRsp
			rf.changeRoleTo(FOLLOWER)
		}
		if voteNum*2 > len(rf.peers) {
			// double check whether curTerm is the same with rf.currentTerm to avoid while executing `RequestVote`, the candidate had started a new election
			if curTerm != rf.currentTerm || rf.role != CANDIDATE {
				rf.mu.Unlock()
				return
			}
			rf.changeRoleTo(LEADER)
			DPrintf("[%d] becomes leader\n", rf.me)
			// NOTE: leader routine
		}
		rf.mu.Unlock()
	}()
}
```

这里的核心代码为中间的协程。

逻辑如下图所示：

![raft_fireElectoin.excalidraw](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/raft_fireElectoin.excalidraw-1645516183030.svg)

定时器到时后， 开启多个 子routine, 多个routine通过rpc发起 `RequstVote` RPC请求，当routine收到回复后，通过 ch 通道返回到 collection routine， 当collection routine收到半数以上的票数后，立即通过 done 通道返回给外层 go routine (也就是fireElection函数中的go关键中的匿名函数)。

之所以做得相对复杂的原因在于，fireElection无需等待所有发起rpc请求的go routine都返回后，再做核算。只用收集到一半以上的票数后，就可以立即转变为leader。

发起 RequestVote RPC的routine代码如下：

```go
func (rf *Raft) doSendRequestVote(args *RequestVoteArgs, ch chan<- RequestVoteReply) {
	var chCloseCnt int32 = int32(len(rf.peers) - 1) // use this to close `ch`
	// sender
	for i := 0; i < len(rf.peers); i++ { // peers is read-only once created, so no mutex needed
		// if rf.killed() { // if rf is killed, return immediately
		// 	return
		// }
		if i == rf.me { // rf.me is read-only once created
			continue
		}
		go func(svrId int) {
			var reply RequestVoteReply
			rf.sendRequestVote(svrId, args, &reply)
			DPrintf("[%d] --> [%d] RequestVote Done\n", rf.me, svrId)
			ch <- reply
			cnt := atomic.AddInt32(&chCloseCnt, -1)
			if cnt == 0 {
				// receive all reply, close channel now
				DPrintf("close ch channel in fireElection\n")
				close(ch)
			}
		}(i)
	}
}
```

这里采用了原子变量的方式来确定何时需要关闭通道—最后一个收到回复的routine负责关闭。

collection routine如下：

```go
func (rf *Raft) doReceiveRequestVoteReply(replyCh <-chan RequestVoteReply, done chan<- bool, maxTermFromRsp *int, voteNum *int) {
	var once sync.Once
	voted := 1
	tmpMaxTerm := 0
	for {
		// if rf.killed() {
		// 	close(done) // actually we don't need this, now that rf is killed, blocked routines will be reclaimed by GC or OS
		// 	return
		// }
		reply, ok := <-replyCh
		if !ok {
			close(done)
			return
		}
		if reply.Term > *maxTermFromRsp {
			tmpMaxTerm = reply.Term
		}
		if reply.VoteGranted { // no mutex needed, `done chanel` will sync `*voteNum` and `maxTermFromRsp` for us
			voted++
			if 2*voted > len(rf.peers) {
				once.Do(func() { // we can't break loop because we need to receive all data from the channel, otherwise some goroutine will be blocked forever
					*voteNum = voted
					*maxTermFromRsp = tmpMaxTerm
					done <- true
				})
			}
		}
	}
}
```

采用 sync.Once来保证一旦收到半数以上的票后，只用反馈一次给外层routine。

最后还有一个注意点为：

在fireElection中的routine的最后几行中有如下：

```go
// double check whether curTerm is the same with rf.currentTerm to avoid while executing `RequestVote`, the candidate had started a new election
if curTerm != rf.currentTerm || rf.role != CANDIDATE {
    rf.mu.Unlock()
    return
}
```
为什么这里要做curTerm的二次检验？因为有可能外层routine还没有收到来自collection routine的返回信息（通过done通道），此时electionTimer再次超时，发起了第二次选举，只有最新的currentTerm能够用于状态转换。

> 后期更新，实际上这里的done通道多余了，直接在collection routine执行 fireElection中的后续逻辑即可。后期的逻辑图如下：
>
> ![raft_fireElectoin.excalidraw](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/raft_fireElectoin.excalidraw.png)

### RequestVote handler

看完了发送端，再看RPC handler

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()         // maybe receive RequestVote by multiple candidates at the same time, so we need this guard
	defer rf.mu.Unlock() // this procedure isn't expensive, so give it a big latch

	reply.Term = rf.currentTerm
	reply.VoteGranted = false

	if args.Term < rf.currentTerm {
		return
	}
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.votedFor = -1
	}

	if rf.votedFor != -1 { // voted before
		DPrintf("[%d] grant vote for [%d] failed: voted for [%d] before \n", rf.me, args.CandidateId, rf.votedFor)
		return
	}
	// check log is at least up-to-date or not
	if len(rf.log) != 0 {
		lastLogEntry := rf.log[len(rf.log)-1]
		if lastLogEntry.Term > args.LastLogTerm ||
			(lastLogEntry.Term == args.LastLogTerm && len(rf.log) > args.LastLogIndex) {
			DPrintf("[%d] vote for [%d] log check check failed\n", rf.me, args.CandidateId)
			return
		}
	}
	rf.votedFor = args.CandidateId
	rf.resetElectionTimer()
	// ok, check pass, grant vote for it
	DPrintf("[%d] grant vote for [%d]\n", rf.me, args.CandidateId)
	reply.VoteGranted = true
}
```

两个注意点：

1. 一旦发现更新的currentTerm，需要更新自己的currentTerm
2. 一旦投票成功，需要重置选举timer

### 心跳实现

相对选举来说，心跳简单很多。代码如下：

```go
// replica log entires and used as heart beat
func (rf *Raft) fireAppendEntiresRPC() {
	go func() {
		var curTerm int
		replies := make([]AppendEntriesReply, len(rf.peers))
		var waitGroup sync.WaitGroup

		rf.mu.Lock()
		curTerm = rf.currentTerm
		waitGroup.Add(len(rf.peers) - 1)
		DPrintf("[%d] curTerm:%d ,AppendEntires\n", rf.me, rf.currentTerm)
		rf.mu.Unlock()

		// send requests
		go rf.doSendAppendEntires(&waitGroup, replies)
		// wait until having received all replies
		waitGroup.Wait()
		if rf.killed() {
			return
		}

		// update  leaders' term if need
		maxTermFromRsp := 0
		for i := 0; i < len(rf.peers); i++ {
			// if rf.killed() {
			// 	return
			// }
			if i == rf.me {
				continue
			}
			if replies[i].Term > maxTermFromRsp {
				maxTermFromRsp = replies[i].Term
			}
		}
		rf.mu.Lock()
		DPrintf("[%d] %+v, me term %d, maxTermFromRsp %d", rf.me, replies, rf.currentTerm, maxTermFromRsp)
		if rf.currentTerm != curTerm {
			// double check, because maybe `currentTerm` has been changed by other routine
			rf.mu.Unlock()
			return
		}
		if rf.currentTerm < maxTermFromRsp {
			rf.currentTerm = maxTermFromRsp
		}
		rf.mu.Unlock()
	}()
}

func (rf *Raft) doSendAppendEntires(waitGroup *sync.WaitGroup, replies []AppendEntriesReply) {
	rf.mu.Lock()
	prevLogTerm := -1
	if len(rf.log) >= 2 {
		prevLogTerm = rf.log[len(rf.log)-2].Term // NOTE: maybe a bug
	}
	args := &AppendEntriesArg{
		Term:         rf.currentTerm,
		LeaderId:     rf.me,
		PrevLogIndex: len(rf.log) - 1,
		PrevLogTerm:  prevLogTerm,
		Entries:      make([]LogEntry, 0),
		LeaderCommit: rf.commitIndex,
	}
    rf.mu.Unlock()

	for i := 0; i < len(rf.peers); i++ {
		// if rf.killed() {
		// 	return
		// }
		if i == rf.me {
			continue
		}
		go func(svrId int) {
			rf.sendAppendEntires(svrId, args, &replies[svrId])
			DPrintf("[%d] --> [%d] AppendEntries Done\n", rf.me, svrId)
			waitGroup.Done()
		}(i)
	}
}
```

注意点和选举类似，包含 currentTerm check. 和选举不同的是，这里采用waitGroup等待所有子routines（发起rpc）完成后，才继续处理后续逻辑。

### AppendEntires实现

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArg, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	reply.Term = rf.currentTerm
	reply.Success = false
	if args.Term < rf.currentTerm {
		return
	}
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.votedFor = -1
	}

	/*
		TODO:
		1. Reply false if term < currentTerm (§5.1)
		2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)
		3. If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it (§5.3)
		4. Append any new entries not already in the log
		5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
	*/
	// TODO: step 2-5 are ignored by now
	if rf.role != FOLLOWER {
		rf.changeRoleTo(FOLLOWER)
	}
	rf.resetElectionTimer()
	reply.Term = rf.currentTerm
	reply.Success = true
	DPrintf("[%d] get an AppendEntiresRPC from leader [%d]\n", rf.me, args.LeaderId)
}
```

注意状态转换：

```go
	if rf.role != FOLLOWER {
		rf.changeRoleTo(FOLLOWER)
	}
```

如果一个leader crash后重新加入网络，那么它是可能收到来自其它leader的 `AppendEntries` RPC, 此时需要将自己重新变为FOLLOWER。

## 2. 公共的注意点

paper中提到对于所有SVR：

- If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1). 除此外，还应该将voteFor设置为nil(我设置为-1），因为已经进入到新的一轮term

## 3. 其它

1. 关于测试：第一次跑通后，并不一定是真的pass了。 最好跑超过10次(实际上我跑了100次，最后发现go的test包会超时，超10分钟后自动crash，并打印所有routine的堆栈，一度让我以为我的实现有什么bug)。

```shell
go test -race -run 2A -count 10
```

2. 另外，在我实现过程fireElection函数中，其实我非常纠结什么时候关闭通道，因为对每个for循环都加入了 rf.killed 检测后，关闭通道会变得非常麻烦，所以后来 我直接放弃了在每个for循环中对 `rf.killed`的检测, 现在关闭channel会变得简单很多。
3. 结合1和2点，会发现在通过log来debug时，有可能出现下一轮测试的日志中，出现上一轮测试的日志。这是因为上一轮go routine还没有完全关闭的情况下，下一轮又开始了。
4. 每个RPC请求是一定会返回的，要么正常，要么超时，所以不用担心channel或者waitGroup卡死，造成go routine一直在后台不退出，除非在RPC handler中，故意不返回，但这是不可能的。

## 4. 总结

本lab是raft的第一部分，主要集中在两个定时器与两个rpc handler的实现，整体相对简单，不过一旦出现问题，在分布式的环境下，debug确实比较困难。

## 5. 参考

- [Raft动画演示](http://thesecretlivesofdata.com/raft/)
- [Students’ Guide to Raft](https://thesquareplanet.com/blog/students-guide-to-raft/)
- [Raft paper](http://nil.csail.mit.edu/6.824/2020/labs/lab-raft.html)