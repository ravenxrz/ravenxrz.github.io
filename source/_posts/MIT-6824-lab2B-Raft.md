---
title: MIT-6824-lab2B-Raft
categories: MIT6824
date: 2022-02-23 22:10:31
tags:
---

## 0. 要求

上文lab2A是本次lab的一个框架，而lab2B则是一些更为细节的东西。

本次lab为raft的第二分部(PartB), 原文要求[在这里](http://nil.csail.mit.edu/6.824/2020/labs/lab-raft.html)

简述lab2B的要求为：

> 实现 leader append log，并通过AppendEntries将log replicate 到其他server中。

看起来相对简单，但实际上实现比lab2A更复杂，debug也很难调。

<!--more-->

## 1. 问题

**本文并非最终实现，最终实现在[这里](https://ravenxrz.github.io/archives/d16f195.html)**

要实现lab2B，至少需要思考如下问题：

1. log是怎么传入到svr的， 接收的接口在哪儿？
2. 如何判定是否接收到了重复的log？
3. leader通过什么机制将自己的log传递给其它svrs, 又是如何传递的？
4. leader什么时候可对之间加入的log确认commit？commit后又该如何apply？
5. **如果网络发生过分区，如何最终的log一致？ 这是本次lab的重点**
6. **如何加快log最终一致性的过程？** 即如何快速的更新 **nextIndex**
7. ...



## 2. 过程描述

整个通信流程图如下：

![lab2b-2](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/lab2b-2.svg)

一次命令的提交，至少需要两个阶段，才能保证所有service的运行状态相同。

先说阶段1：

1. client发起请求到leader的上层服务
2. 上层服务通过raft暴露的Start接口，将命令发送给raft， raft将命令打包为log entry，append到log中
3. 通过心跳（AppendEntries）发送给其他svr
4. 其他raft实例收到新log后，将新log保存下来，并响应到leader。 **这一步其实还有很多工作，比如log冲突了该如何处理**
5. leader收到半数以上的响应后，标记log为committed，并apply到service
6. service至此可以响应给client。

再说下一阶段，因为上一阶段只能保证leader侧的log committed，其他svr并不知道刚才接收的log是否已经被提交，所以需要阶段2，将leader认为的commitIndex传递给其他svrs。

7. leader将commitIndex等信息传递给其他svr
8. 其他svr更新自己的commitIndex，响应给leader，并开始apply log



##  3. 实现

下面分别针对一中的几个问题，一一解答。 

### 1. log如何传递到leader？如何判定重复log？

client发起的请求只能传递给leader，leader通过 **Start接口** 接收上层服务传递进来的命令，append到log后，立即返回，**不必等到这个log被committed**。

除此外，为了避免leader重复接收到相同的log（比如网络延迟等问题造成client重发命令），每次传递进来的command，leader都需要做重复性检测，之类我的做法是直接从后往前扫描log，判定是否有重复command。 **之所以可以这样做，是因为假设每个command都是unique的，paper中也提到可以为每个command生成一个全局唯一的id**

下面为我个人的Start实现：

```go
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	index := -1
	term := -1
	isLeader := true

	term, isLeader = rf.GetState()
	if !isLeader || rf.killed() {
		return index, term, isLeader
	}

	// check whether command appears in the log or not
	rf.mu.Lock()
	for i := len(rf.log) - 1; i >= 0; i-- { // backward scan is faster
		if rf.log[i].Command == command {
			index = i + 1
			rf.mu.Unlock()
			return index, term, isLeader
		}
	}
	// not found in the log(range between[0, commitIndex]), append it
	rf.log = append(rf.log, LogEntry{Term: rf.currentTerm, Command: command})
	index = len(rf.log)
	DPrintf("[%d.%d.%d] appends new log entry[%v] at index[%d]\n", rf.me, rf.role, rf.currentTerm, len(rf.log), index)
	rf.mu.Unlock()

	return index, term, isLeader
}
```

### 2. leader如何将log传递给其它svr？如何commit、apply log？

#### leader侧

leader通过 **AppendEntries** RPC将新log传递给其他log，然后等待半数以上的成功响应。

这里我采用了和Lab2A中的 **fireElection** 类似的trick。如下图：

![raft_fireAppendEntries.excalidraw](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/raft_fireAppendEntries.excalidraw.svg)

leader心跳到时后，派生成两类线程，上图右侧的routine表示用于给其他svr发送rpc的线程，左侧collection routine用于接收发送线程的reply。 两者之间通信采用channel。

fireAppendEntires函数如下：

```go
func (rf *Raft) fireAppendEntires(fromHeartBeatTimer bool) {
	rf.mu.Lock()
	if fromHeartBeatTimer && rf.hBStopByCommitOp {
		rf.mu.Unlock()
		return
	}
	pendingCommitIndex := len(rf.log) - 1
	curTerm := rf.currentTerm
	rf.mu.Unlock()
	// use the same trick in `fireElection`
	ch := make(chan AppendEntriesReplyInCh)
	// send requests
	go rf.doSendAppendEntires(ch, curTerm, pendingCommitIndex)
	// receive
	go rf.doReceiveAppendEntries(ch, curTerm, pendingCommitIndex)
}

```

忽略参数和hBStopByCommitOp相关逻辑，后文会提到。

额外注意这里的 **pendingCommitIndex**。我们必须记录当前发起心跳时，当时需要commit的log长度，因为在进行心跳的过程中，log的长度是可能被修改的。

##### 发送端

发送端的工作流程为：

1. 打包rpc参数
2. 执行发送
3. 最后一个发送rpc的routine，负责关闭channel

```go
func (rf *Raft) doSendAppendEntires(replyCh chan AppendEntriesReplyInCh, curTerm, pendingCommitIndex int) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	var sendRpcNum int = 0 // use this to determine when to close channel
	// for debug
	// curTerm := rf.currentTerm
	// curRole := rf.role

	for i := 0; i < len(rf.peers) && rf.role == LEADER; i++ {
		if i == rf.me {
			continue
		}
		if rf.currentTerm != curTerm {
			sendRpcNum++ // plus 1 so that we can close channel
			if sendRpcNum == len(rf.peers)-1 {
				close(replyCh)
			}
			continue
		}

		prevLogTerm := -1
		prevLogIndex := -1
		var entries []LogEntry
		if rf.nextIndex[i] > 0 {
			DPrintf("[%d] log len:%d, nextIndex[%d]=%d\n", rf.me, len(rf.log), i, rf.nextIndex[i])
			prevLogTerm = rf.log[rf.nextIndex[i]-1].Term
			prevLogIndex = rf.nextIndex[i] - 1
		} else if rf.nextIndex[i] < 0 && pendingCommitIndex >= 0 { // NOTE: Fix bug: if svr becomes leader, set all nextIndex to -1, then accepted new logs, and then doSendAppendEntries
			rf.nextIndex[i] = 0
		}

		if rf.nextIndex[i] >= 0 {
			// for debug
			if rf.nextIndex[i] > pendingCommitIndex+1 {
				log.Panicf("rf.nextIndex[i]=%d is large than pendingCommitIndex=%d", rf.nextIndex[i], pendingCommitIndex)
			}
			entries = make([]LogEntry, 0) // do a copy
			entries = append(entries, rf.log[rf.nextIndex[i]:pendingCommitIndex+1]...)
		}

		args := &AppendEntriesArg{
			Term:         rf.currentTerm,
			LeaderId:     rf.me,
			PrevLogIndex: prevLogIndex,
			PrevLogTerm:  prevLogTerm,
			Entries:      entries,
			LeaderCommit: rf.commitIndex,
		}
		DPrintf("[%d.%d.%d] --> [%d] sendAppendEntires, total log len:%d, args:%+v, nextIndex[%d]=%d\n", rf.me, rf.role, rf.currentTerm, i, len(rf.log), args, i, rf.nextIndex[i])
		go func(svrId int) {
			var reply AppendEntriesReply
			rf.sendAppendEntires(svrId, args, &reply)
			// DPrintf("[%d.%d.%d] --> [%d] AppendEntries Done, reply:%+v\n", rf.me, curRole, curTerm, svrId, reply)

			rf.mu.Lock()
			if rf.role == LEADER {
				rf.mu.Unlock()
				replyCh <- AppendEntriesReplyInCh{
					AppendEntriesReply: reply,
					svrId:              svrId,
				}
				rf.mu.Lock()
				sendRpcNum++ // NOTE:sendPrcNum++ here to prevent close opeartion(below) was executed ahead of replyCh channel due to no one accpet reply
			} else {
				sendRpcNum++
			}
			cnt := sendRpcNum
			rf.mu.Unlock()

			if cnt == len(rf.peers)-1 {
				close(replyCh)
				// DPrintf("[%d.%d.%d] SendAppendEntries close reply channel\n", rf.me, curRole, curTerm)
			}
		}(i)
	}
}
```

这里最重要的是如何打包参数，特别是如何确定发送log的长度，以及要注意，**由于本lab所有svr都跑在同一个机器的同一个进程中, 再打包log entries时，最好做一次全拷贝，而不要用引用**

最后就是关闭channel的方式，通过一个计数器，每个发送rpc的线程结束时，需要对计数器加1，当最后一个线程结束时，关闭channel，否则接收端可能永远死等在channel上。

##### 接收端

接收端的工作流程为：

1. 一旦收到rpc响应，更新 nextIndex. 
2. 一旦收到半数以上的成功响应（这个过程有多次，比如有5个svr，获取到3个成功响应即为半数，但实际还可能又第4个和第5个响应），更新commitIndex，并开始apply log（这个过程只能有一次），同时立即进入阶段2，进入阶段2后，立即尝试关闭heartBeatTimer（不一定真的能立即关闭），避免阶段2的AppendEntries和心跳发送AppendEntries同时进行，否则有一个关于RPC字节数统计的单元测试无法通过。

代码如下：

```go
func (rf *Raft) doReceiveAppendEntries(replyCh <-chan AppendEntriesReplyInCh, curTerm, pendingCommitIndex int) {
	var succOne sync.Once
	// var failOne sync.Once
	successCnt := 1
	failCnt := 0

	for {
		reply, ok := <-replyCh
		if !ok {
			return
		}

		rf.mu.Lock()
		if curTerm != rf.currentTerm || rf.role != LEADER || rf.killed() {
			rf.mu.Unlock()
			// return
			continue // if we return now, replyCh will block some routines
		}

		if reply.Term > rf.currentTerm {
			DPrintf("[%d.%d.%d] --> [%d] , remote term %d is large than me \n", rf.me, rf.role, rf.currentTerm, reply.svrId, reply.Term)
			// for debug
			if reply.Success {
				log.Panicf("reply.Term is large	than me, but reply.Success is true")
			}
			rf.setNewTerm(reply.Term)
			rf.changeRoleTo(FOLLOWER) // someone's term is large than me, so give up leader role and change to follower
			rf.mu.Unlock()
			continue
		}
		// update nextIndex and matchIndex
		rf.updateNextIndex(reply, pendingCommitIndex)
		if reply.Success {
			successCnt++
		} else {
			failCnt++
		}

		if 2*successCnt > len(rf.peers) {
			succOne.Do(func() {
				rf.hBStopByCommitOp = false // force to reset preempByCommit so that heartBeatTimer can send msg
				if curTerm != rf.currentTerm || rf.role != LEADER || rf.killed() {
					// return
					return
				}
				if rf.commitIndex != pendingCommitIndex {
					rf.commitIndex = pendingCommitIndex
					DPrintf("[%d.%d.%d] get majority of appendentries rsp, change committedIndex to %d\n", rf.me, rf.role, rf.currentTerm, pendingCommitIndex)
					// notify applier
					rf.committedChangeCond.Broadcast()
					rf.hBStopByCommitOp = true
					rf.mu.Unlock()
					// fire next turn AppendEntries to tell others to commit
					rf.fireAppendEntires(false)
					rf.mu.Lock() // TODO:lock followed unlock immediately, how to fix this? use `go rf.fireAppendEntires` is another way, but launch a new goroutine is costly too
				}
			})
		} else if 2*failCnt > len(rf.peers) && rf.hBStopByCommitOp {
			//
			// unlikely,
			// actually, it's unnecessary to execute code below
			// now that leader `commit opeartion` failed, which means leader is separeted from others.
            // so we can assume that leader will convert to follower in the further(when rejoin to the network), therefore no heartbeat is needed right now
			// comment those code is much better because it reduce the num of useless heartbeat
			//
			// failOne.Do(func() {
			// 	rf.mu.Lock()
			// 	defer rf.mu.Lock()
			// 	rf.preempByCommit = false // force to turn on heartBeatTimer
			// })
			DPrintf("[%d] commit operation failed, can't get majority rsp\n", rf.me)
		}
		rf.mu.Unlock()
	}
}

```

关于如何updateNextIndex和如何apply下文再说。

这里的上半部分逻辑其实很好理解，无非是一些额外检查，然后更新一些index，最难理解的是 `hBStopByCommitOp` 变量，这个变量是用来避免 once.Do中的 `rf.fireAppendEntires(false)` 与心跳中的 `rf.fireAppendEntires(true)` 同时进行（不过不能精确保证，可能最开始还是会存在同时进行，但是最多一轮这样的同时进行）。这个检测在 `fireAppendEntries`中：

```go
func (rf *Raft) fireAppendEntires(fromHeartBeatTimer bool) {
	rf.mu.Lock()
	if fromHeartBeatTimer && rf.hBStopByCommitOp { 
		rf.mu.Unlock()
		return
	}
	...
}
```

另外，最后有一大段的注释所在分支，其实是为了避免 once.Do 中的 `fireAppendEntries` 无法收到大多数的成功响应，这样无法重置`hBStopByCommitOp=false`，导致心跳停止，出现问题。但其实，这一点我通过另一个位置处的重置来解决，那就是当一个svr转变为leader的时候：

```go
func (rf *Raft) changeRoleTo(role RaftRole) {
	...
	if role == LEADER {
		rf.resetNextIndex()
		rf.hBStopByCommitOp = false // force to turn on heartbeatTimer
	}
}
```

#### svr侧

svr收到log消息，通过一定的一致性检验后，即可append log， 并根据leader的commitindex更新自己的commitIndex，然后开始apply log。代码如下， 关于XTerm，XIndex，XLen，可直接忽略，后文再说：

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArg, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	DPrintf("[%d.%d.%d] AppendEntiresRPC start from leader [%d], log len:%d, log info %+v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, len(rf.log), rf.log)
	if rf.role != FOLLOWER {
		rf.changeRoleTo(FOLLOWER)
	}
	rf.votedFor = args.LeaderId
	rf.resetElectionTimer()
	reply.Term = rf.currentTerm
	reply.Success = false
	reply.XTerm = -1
	reply.XIndex = -1
	reply.XLen = -1

	// follow all server rule
	if args.Term > rf.currentTerm {
		rf.setNewTerm(args.Term)
	}

	// 1. Reply false if term < currentTerm (§5.1)
	if args.Term < rf.currentTerm {
		return
	}
	if args.PrevLogIndex != -1 {
		// 2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)
		if len(rf.log)-1 < args.PrevLogIndex {
			// case 3: rf.log is too short
			reply.XLen = len(rf.log)
			return
		}
		if rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
			reply.XTerm = rf.log[args.PrevLogIndex].Term
			reply.XIndex = args.PrevLogIndex
			i := args.PrevLogIndex - 1
			// find the first index of args.PrevLogTerm in rf.log
			for i >= 0 && rf.log[i].Term == rf.log[i+1].Term {
				reply.XIndex = i
				i--
			}
			return
		}
		rf.log = rf.log[:args.PrevLogIndex+1]
	} else {
		// clear log, because leader sent a full log
		rf.log = []LogEntry{}
	}
	// 4. Append any new entries not already in the log
	rf.log = append(rf.log, args.Entries...)
	// 5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
	if args.LeaderCommit > rf.commitIndex {
		if args.LeaderCommit > len(rf.log)-1 {
			rf.commitIndex = len(rf.log) - 1
		} else {
			rf.commitIndex = args.LeaderCommit
		}
		// notify applier
		rf.committedChangeCond.Broadcast()
		// NOTE: persistent, maybe do a batch persistent?
		DPrintf("[%d] change commitIndex to %d\n", rf.me, rf.commitIndex)
	}
	reply.Success = true
	DPrintf("[%d.%d.%d] AppendEntiresRPC end from leader [%d], log len:%d, log info %+v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, len(rf.log), rf.log)
}
```

### 3. 如果网络发生过分区，如何保证log的最终一致？如何加速这个过程？

为了保证log的最终一致，raft提出了 log replication的策略，原理参考 paper的5.3节，或者看课程对应视频（没记错应该时raft的第二课）。

为了解决这个问题的核心变量有四个：

1. prevLogIndex
2. prevLogTerm
3. nextIndex
4. log

leader通过不断的发送心跳rpc，收到响应后，根据响应结果，调整如上四个参数（具体设定为 prevLogIndex = nextIndex-1; prevLogTerm=log[prevLogIndex]; log=rf.log[nextIndex:])，直到leader找到一个log位置，这个位置之前的所有log和rpc对应的svr达成了一致性共识。 然后将这个位置开始及其之后的log，全部发送给rpc对应的svr。最糟糕的情况就是发送全部的log。

原paper中提出的方案是如果leader收到了响应，响应中提到svr和leader的log不一致，那么leader将nextIndex--, 然后从上一个位置开始重新发起rpc。但是这个方案过慢，视频课程中提出了新的一种加速方案，通过如下几个变量即可加速：

- XTerm， 表示在svr的prevLogIndex处的log Entry Term（如果对应处有log Entry的话）
- XIndex，表示在XTerm在svr log中第一次出现的Index位置。
- XLen，表示如果svr的prevLogIndex处如果没有log Entry的话，此时svr的log总长度是多少。

那么当leader收到一次响应，响应中铁道svr和leader的log不一致，leader通过如上三个变量，即可快速调整nextIndex, 而不用一次次调整。

#### svr侧

代码如下：

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArg, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	DPrintf("[%d.%d.%d] AppendEntiresRPC start from leader [%d], log len:%d, log info %+v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, len(rf.log), rf.log)
	if rf.role != FOLLOWER {
		rf.changeRoleTo(FOLLOWER)
	}
	rf.votedFor = args.LeaderId
	rf.resetElectionTimer()
	reply.Term = rf.currentTerm
	reply.Success = false
	reply.XTerm = -1
	reply.XIndex = -1
	reply.XLen = -1

	// follow all server rule
	if args.Term > rf.currentTerm {
		rf.setNewTerm(args.Term)
	}

	// 1. Reply false if term < currentTerm (§5.1)
	if args.Term < rf.currentTerm {
		return
	}
	if args.PrevLogIndex != -1 {
		// 2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)
		if len(rf.log)-1 < args.PrevLogIndex {
			// case 3: rf.log is too short
			reply.XLen = len(rf.log)
			return
		}
		if rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
			reply.XTerm = rf.log[args.PrevLogIndex].Term
			reply.XIndex = args.PrevLogIndex
			i := args.PrevLogIndex - 1
			// find the first index of args.PrevLogTerm in rf.log
			for i >= 0 && rf.log[i].Term == rf.log[i+1].Term {
				reply.XIndex = i
				i--
			}
			return
		}
		rf.log = rf.log[:args.PrevLogIndex+1]
	} else {
		// clear log, because leader sent a full log
		rf.log = []LogEntry{}
	}
	// 4. Append any new entries not already in the log
	rf.log = append(rf.log, args.Entries...)
	// 5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
	if args.LeaderCommit > rf.commitIndex {
		if args.LeaderCommit > len(rf.log)-1 {
			rf.commitIndex = len(rf.log) - 1
		} else {
			rf.commitIndex = args.LeaderCommit
		}
		// notify applier
		rf.committedChangeCond.Broadcast()
		// NOTE: persistent, maybe do a batch persistent?
		DPrintf("[%d] change commitIndex to %d\n", rf.me, rf.commitIndex)
	}
	reply.Success = true
	DPrintf("[%d.%d.%d] AppendEntiresRPC end from leader [%d], log len:%d, log info %+v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, len(rf.log), rf.log)
}
```

#### leader侧

```go
func (rf *Raft) updateNextIndex(reply AppendEntriesReplyInCh, pendingCommitIndex int) {
	if !reply.Success {
		if reply.XTerm == 0 && reply.XLen == 0 { // success = false, and both equal to 0, assume this is a timeout
			DPrintf("[%d] --> [%d], reply unsuccess timeout\n", rf.me, reply.svrId)
			return
		}
		if reply.XTerm == -1 {
			rf.nextIndex[reply.svrId] = reply.XLen
			DPrintf("[%d.%d.%d] --> [%d] reply unsuccess(remote log is too short), set nextIdx=%d\n", rf.me, rf.role, rf.currentTerm, reply.svrId, rf.nextIndex[reply.svrId])
		} else {
			i := reply.XIndex
			// for debug
			if i < 0 || i >= len(rf.log) {
				log.Panicf("[%d] reply.XIndex is %d, len(rf.log)=%d", rf.me, i, len(rf.log))
			}
			// update
			if rf.log[i].Term == reply.XTerm {
				oldNextIndex := rf.nextIndex[reply.svrId]
				rf.nextIndex[reply.svrId] = i + 1
				i++
				for i < oldNextIndex && rf.log[i].Term == reply.XTerm {
					rf.nextIndex[reply.svrId] = i
					i++
				}
				DPrintf("[%d.%d.%d] --> [%d] reply unsuccess(case 2, XTerm == log.Term), set nextIdx=%d\n", rf.me, rf.role, rf.currentTerm, reply.svrId, rf.nextIndex[reply.svrId])
			} else {
				rf.nextIndex[reply.svrId] = i
				DPrintf("[%d.%d.%d] --> [%d] reply unsuccess(case 1, XTerm != log.Term), set nextIdx=%d\n", rf.me, rf.role, rf.currentTerm, reply.svrId, rf.nextIndex[reply.svrId])
			}
		}
		// for debug
		// if rf.nextIndex[reply.svrId] <= rf.matchIndex[reply.svrId] {
		// 	log.Panicf("[%d] nextIndex[%d]=%d, matchIndex[%d]=%d", rf.me, reply.svrId, rf.nextIndex[reply.svrId], reply.svrId, rf.matchIndex[reply.svrId])
		// }
		// rf.nextIndex[reply.svrId]-- // NOTE: old scheme
	} else {
		rf.nextIndex[reply.svrId] = pendingCommitIndex + 1
		rf.matchIndex[reply.svrId] = rf.nextIndex[reply.svrId] - 1 // NOTE: maybe a bug here
		DPrintf("[%d] matchIndex[%d]=%d, nextIndex[%d]=%d\n", rf.me, reply.svrId, rf.matchIndex[reply.svrId], reply.svrId, rf.nextIndex[reply.svrId])
	}
}
```

### 4. 如何apply log， 如何通知service已经commit了log

首先要说的是apply log就是在通知service了。

那么如何apply log？

我采用的方法是，开启专用线程用于apply log，每当commitIndex修改后，通知该线程开始apply log。

```go
func (rf *Raft) applier() {
	DPrintf("[%d] applier start...\n", rf.me)
	for {
		var commitIndex int
		rf.mu.Lock()
		for rf.lastAppliedIndex == rf.commitIndex && !rf.killed() {
			rf.committedChangeCond.Wait()
		}
		commitIndex = rf.commitIndex
		if rf.killed() {
			rf.mu.Unlock()
			break
		}
		rf.lastAppliedIndex++
		DPrintf("[%d] applier try to apply log[%d-%d]\n", rf.me, rf.lastAppliedIndex, commitIndex)
		for rf.lastAppliedIndex <= commitIndex {
			rf.applyCh <- ApplyMsg{
				CommandValid: true, // NOTE: how to use this? maybe need to fix this in the future
				Command:      rf.log[rf.lastAppliedIndex].Command,
				CommandIndex: rf.lastAppliedIndex + 1, // raft's log id starts from 0, but service starts from 1, so fix it by plus 1
			}
			// DPrintf("[%d] appiler feeds command, command idx:%d\n", rf.me, rf.lastAppliedIndex)
			rf.lastAppliedIndex++
		}
		rf.lastAppliedIndex-- // back one step
		rf.mu.Unlock()        // NOTE: another solution is put the lock inside for loop in order to avoid being blocked by applyCh, but we assume service will accept data immediately here
	}
}
```

## 4. 关于调试

lab2B的单元测试更多，也更复杂，特别是涉及到网络分区后，更容易出现问题，debug方式目前我也只想到了DPrintf，那如何帮助自己更好调试，我采用了如下方式：

1. 尽量让log包含更多有效信息，比如下面这条log

   ```go
   DPrintf("[%d.%d.%d] --> [%d] reply unsuccess(case 2), set nextIdx=%d\n", rf.me, rf.role, rf.currentTerm, reply.svrId, rf.nextIndex[reply.svrId])
   ```

   开头三个 %d分别代表，当前svrId， svr的角色（leader？follower？candidate？), 以及当前term。

2. 通过重定向将log导出，然后通过 grep 来分析具体角色，比如我只想看 svrId=0且它变成了leader的log

   ```shell
   grep -E "0\.4\." out  # 这里的4代表leader role。
   ```

3. 将单元测试修改得更容易分析，比如我在实现得过程中，测试 **TestBackup2B** 耗费了非常久， 因为它是随机生成命令，且每次append log数量有50个，非常难比较每个svr的log差别。于是我自己改写了一个简化版：

   ```go
   func TestSimpleBackup2B(t *testing.T) {
   	servers := 5
   	cfg := make_config(t, servers, false)
   	defer cfg.cleanup()
   	cmdNum := 50
   
   	cmd := 0
   
   	cfg.begin("Test (2B): leader backs up quickly over incorrect follower logs")
   
   	cmd++
   	cfg.one(cmd, servers, true)
   
   	// put leader and one follower in a partition
   	leader1 := cfg.checkOneLeader()
   	cfg.disconnect((leader1 + 2) % servers)
   	cfg.disconnect((leader1 + 3) % servers)
   	cfg.disconnect((leader1 + 4) % servers)
   	DPrintf("disconnect: %v %v %v\n", (leader1+2)%servers, (leader1+3)%servers, (leader1+4)%servers)
   
   	// submit lots of commands that won't commit
   	for i := 0; i < cmdNum; i++ {
   		cmd++
   		cfg.rafts[leader1].Start(cmd)
   	}
   
   	time.Sleep(RaftElectionTimeout / 2)
   
   	cfg.disconnect((leader1 + 0) % servers)
   	cfg.disconnect((leader1 + 1) % servers)
   	DPrintf("disconnect: %v %v\n", (leader1+0)%servers, (leader1+1)%servers)
   
   	// allow other partition to recover
   	cfg.connect((leader1 + 2) % servers)
   	cfg.connect((leader1 + 3) % servers)
   	cfg.connect((leader1 + 4) % servers)
   	DPrintf("connect: %v %v %v\n", (leader1+2)%servers, (leader1+3)%servers, (leader1+4)%servers)
   
   	// lots of successful commands to new group.
   	for i := 0; i < cmdNum; i++ {
   		cmd++
   		cfg.one(cmd, 3, true)
   	}
   
   	// now another partitioned leader and one follower
   	leader2 := cfg.checkOneLeader()
   	other := (leader1 + 2) % servers
   	if leader2 == other {
   		other = (leader2 + 1) % servers
   	}
   	cfg.disconnect(other)
   	DPrintf("disconnect: %v \n", other)
   
   	// lots more commands that won't commit
   	for i := 0; i < cmdNum; i++ {
   		cmd++
   		cfg.rafts[leader2].Start(cmd)
   	}
   
   	time.Sleep(RaftElectionTimeout / 2)
   
   	// bring original leader back to life,
   	for i := 0; i < servers; i++ {
   		cfg.disconnect(i)
   	}
   	cfg.connect((leader1 + 0) % servers)
   	cfg.connect((leader1 + 1) % servers)
   	cfg.connect(other)
   	DPrintf("connect: %v %v %v\n", (leader1+0)%servers, (leader1+1)%servers, other)
   
   	// lots of successful commands to new group.
   	for i := 0; i < cmdNum; i++ {
   		cmd++
   		cfg.one(cmd, 3, true)
   	}
   
   	// now everyone
   	for i := 0; i < servers; i++ {
   		cfg.connect(i)
   	}
   	cmd++
   	cfg.one(cmd, servers, true)
   
   	cfg.end()
   }
   ```

   这样分析过简单很多。

## 5. 总结

通过lab2B，能够对log replication有个更底层了解，特别是对如何解决 log不一致问题，以及如何加速这个过程有了深刻理解。

至此，lab2B通过：

![image-20220224194848472](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220224194848472.png)

