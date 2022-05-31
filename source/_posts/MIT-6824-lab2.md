---
title: MIT-6824-lab2-Raft总结
categories: MIT6824
date: 2022-03-7 22:10:31
tags:
---



## 0. 前言

文本为MIT-6824-lab2-Raft的总结篇。

在做lab3时，发现之前做的lab2虽然通过了几次全部测试，但是依然存在一些严重bug。在修复后决定不扰乱前三篇文章，重写本篇总结。

<!--more-->

## 1.  工作流

可以把Raft看作一个用于分布式环境下达到共识的lib，上层服务通过嵌入raft lib，使用raft提供的接口即可达到共识。整个工作流如下：

1. **拥有leader身份**的上层服务通过 **Start** 接口将要执行的命令传入到leader raft 中，Raft收到lib后，将其保存在内部的log中(log entry)
2. leader raft lib将收到的log entry通过网络发送到各个副本中，各个副本 **通过一定的验证** 后，给leader raft应答成功拷贝log的消息回复
3. leader raft在收到 **超过一半（包括自己)**的成功拷贝消息后，可以认为这个log entry已经被“安全”拷贝，于是**向上层应答第1步发来的命令(log entry)已经成功commit**， 与此同时发送commit消息给各个副本
4. 各个副本收到raft发来commit消息，根据commit消息，将第2步收到的log entry标记为committed，同时也**向上层应答该log entry已经成功commit**。

![raft框架](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/raft框架.svg)

以上4个步骤为raft的最基本的工作流，但要进入该工作流，还有非常多的问题要思考并解决，如下几个：

1. 各个svr如何进入leader身份？
2. 进入leader身份后，如何保证自己的leader身份不丢失？
3. 如果网络发生了分区该如何处理，如何应对 split brain 问题？
4. 网络分区后又愈合，如何处理多leader的情况？
5. 如果网络中某些svr crash并重启，如何恢复该svr到crash之前的状态？
6. 如果一个svr掉线过久，导致其log落后其他svr太多，如何快速“追上”其他svr？
7. 如果发生网络分区，在 minority  partition 中的leader接收了过多新log，当网络愈合后，如何快速“同步”log?
8. 如何应对请求乱序问题，即对于网络来说，不保证发送请求的包到达对端的顺序问题？
9. 如何应对log无限增长问题？
10. ....

raft的基本工作流非常简单，但是其底层的细节问题有非常多，需要小心处理

## 2. 实现

现在抛开所有细节问题不谈，谈谈raft的基本框架（对应lab1）。对于一个raft实例来说，它对上提供了 **Start** 接口，对其他raft实例来说，可参与leader选举，心跳同步两个操作。正如raft paper figure 2所示，其实核心无非两个RPC handle.

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220222145423023.png" alt="image-20220222145423023" style="zoom:50%;" />

### 2.1 三个状态

所有的svr都可能有三个状态 {leader, candidate, follower}。 所有svr最开始的状态都是follower，每个svr都有一个选举定时器，当定时器到时，follower可转换为candidate，并开始向其他svr宣告自己想要称为leader，看其他svr是否统一，如果candiate能够收到超过一半的投票，那么它就可以转为leader。 

![raft状态转换图](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/raft状态转换图.svg)

**只有leader可以接收上层传来的命令。**

### 2.2 选举

#### 2.2.1 发送端

选举的起点在于选举定时器到时，然后转化身份并发起 **RequestVote** RPC请求。 另外要注意的是选举定时器是可以被Reset并重启开始计时的，**选举过程也可被其他同时参与选举的candidate或者leader所中断的**。

先说小问题，如何实现一个可被Reset的定时器？通过time.Sleep + 定期检测是否超时实现。如下代码, 设定一个定时总时长，并设置一个打盹时间(小于总时长)，每次仅睡眠一个打盹时间，然后检查当前所剩的总时长（总时长可被在其他地方设定，也就是Reset）。

```go
func (rf *Raft) electionTimer() {
	const napTime = 100
	if napTime >= rf.electionTimeout {
		log.Panicf("electionTimer napTime(%v) shouldn't be less than rf.electionTimeout(%v)", napTime, rf.electionTimeout)
	}
	DPrintf("[%d.%d.%d] electionTimer start...\n", rf.me, rf.role, rf.currentTerm)
	timerId := time.Now().Unix()
	for {
		rf.mu.Lock(rf.me, "electionTimer")
		for rf.role == LEADER && !rf.killed() {
			rf.roleChangeCond.Wait()
		}
		if rf.killed() {
			rf.mu.Unlock(rf.me, "electionTimer")
			break
		}
		// assert rf.role == FOLLOWER || rf.role == CANDIDATE
		if rf.electionTimeout <= 0 {
			// fire election
			DPrintf("[%d], fire election..\n", rf.me)
			rf.fireElection()
			rf.mu.Unlock(rf.me, "electionTimer")
			continue
		}
		rf.electionTimeout -= napTime
		rf.mu.Unlock(rf.me, "electionTimer")

		time.Sleep(napTime * time.Millisecond)
	}
	DPrintf("[electionTimeId %d] exits\n", timerId)
}
```

再说如何发起RequestVote RPC，已经如何接受RPC的响应？

发起RequesetVote很简单，**关键在于对于不同的响应结果该如何处理**。同时还应该关注到，**由于网络延迟的存在，一个candidate可能同时发起了多伦RequestVote**，我们只能对最新一轮RequestVote做出反应，丢弃所有旧的请求。

这部分我所采用的处理逻辑图如下：

![raft_fireElectoin.excalidraw](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/raft_fireElectoin.excalidraw.png)

要向多个svr同时发起RPC请求，那么就开启多个线程（本文中线程和协程相同意义）， 该请求是耗时的，我们不能阻塞raft实例，所以对于接收部分，也开启了专用的collection线程。每个 routine x 收到响应后，通过通道回复 collection routine.

现在来看看最重要的部分，如何应对不同响应结果？

1. 如果能收到半数以上票，且发起rpc时的term是当前rf实例的term，则立即转为leader。
2. 如果不能收到半数以上票（可能因为对端svr投了反对票又或者网络断开），且发起rpc时的temr是当前rf实例的term，则应该放弃candidate身份，转为follower。

关于发起rpc和响应rpc的代码如下：

```go
func (rf *Raft) doSendRequestVote(ch chan<- RequestVoteReply) {
	var chCloseCnt int32 = int32(len(rf.peers) - 1) // use this to close `ch`
	// for debug
	rf.mu.Lock(rf.me, "doSendRequestVote")
	curRole := rf.role
	lastLogTerm := -1
	if len(rf.log) != 0 {
		lastLogTerm = rf.log[len(rf.log)-1].Term
	}
	args := &RequestVoteArgs{
		Term:         rf.currentTerm,
		CandidateId:  rf.me,
		LastLogIndex: len(rf.log) - 1,
		LastLogTerm:  lastLogTerm,
	}
	rf.mu.Unlock(rf.me, "doSendRequestVote")
	// sender
	for i := 0; i < len(rf.peers); i++ { // peers is read-only once created, so no mutex needed
		if i == rf.me { 
			continue
		}
		go func(svrId int) {
			var reply RequestVoteReply
			rf.sendRequestVote(svrId, args, &reply)
			DPrintf("[%d.%d.%d] --> [%d] RequestVote Done, reply:%+v\n", rf.me, curRole, args.Term, svrId, reply)
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

func (rf *Raft) doReceiveRequestVoteReply(curTerm int, replyCh <-chan RequestVoteReply) {
	voteNum := 1
	unVoteNum := 1
	var voteOnce sync.Once
	var unVoteOnce sync.Once
	maxTermFromRsp := 0
	// batch persist
	defer func() {
		rf.mu.Lock(rf.me, "doReceiveRequestVoteReply")
		if rf.pendingPersist {
			rf.persist()
			rf.pendingPersist = false
		}
		rf.mu.Unlock(rf.me, "doReceiveRequestVoteReply")
	}()

	for {
		reply, ok := <-replyCh
		if !ok {
			return
		}
		if reply.Term > maxTermFromRsp {
			maxTermFromRsp = reply.Term
		}
		if reply.VoteGranted { 
			voteNum++
			if 2*voteNum > len(rf.peers) {
				voteOnce.Do(func() { // we can't break loop because we need to receive all data from the channel, otherwise some goroutine will be blocked forever
					// update if need
					rf.mu.Lock(rf.me, "doReceiveRequestVoteReply.once")
					// double check whether curTerm is the same with rf.currentTerm to avoid while executing `RequestVote`, the candidate had started a new election
					if curTerm != rf.currentTerm || rf.role != CANDIDATE || rf.killed() {
						rf.mu.Unlock(rf.me, "doReceiveRequestVoteReply.once")
						return
					}
					if rf.currentTerm < maxTermFromRsp {
						rf.turnOnPendingPersist(rf.setNewTerm, maxTermFromRsp)
						rf.changeRoleTo(FOLLOWER)
						rf.mu.Unlock(rf.me, "doReceiveRequestVoteReply.once")
						return
					}
					rf.changeRoleTo(LEADER)
					DPrintf("[%d.%d.%d] becomes leader, log len:%d log content:%v \n", rf.me, rf.role, rf.currentTerm, len(rf.log), rf.log)
					rf.mu.Unlock(rf.me, "doReceiveRequestVoteReply.once")
				})
			}
		} else {
			unVoteNum++
			if 2*unVoteNum >= len(rf.peers) {
				unVoteOnce.Do(func() {
					rf.mu.Lock(rf.me, "doReceiveRequestVoteReply.unVote")
					if curTerm != rf.currentTerm || rf.role != CANDIDATE || rf.killed() {
						rf.mu.Unlock(rf.me, "doReceiveRequestVoteReply.unVote")
						return
					}
					rf.changeRoleTo(FOLLOWER)
					rf.mu.Unlock(rf.me, "doReceiveRequestVoteReply.unVote")
				})
			}
		}
	}
}
```

值得注意的是，在接收消息时，采用了sync.Once，因为“收到超过半数的票”这种情况是可能发生多次的，而我们仅需要在第一次进入“该条件”时做出反应。

#### 2.2.2 接收端

接收端，也就是RequestVote的handle。我们分别代入不同的身份去看，如果收到该请求该如何处理：

1. follower：收到RequestVote，需要做一定log的校验（log at least up-to-date rule) ，看应该投同意票还是反对票.  一种投反对票的情况是我已经给其它svr投票了，而新来请求的term并不比我的curTerm高，此时我应该投反对票。另一种是对方的term比我的term低，此时也应该是反对票
2. candidate: 收到ReuqestVote，说明在一个短时间内，同时有两个及以上的candidate存在（包括自己），如果对方的term大于我的term，那么我应该放弃继续请求，转变为follower，然后回到 follower 角色的处理逻辑。
3. leader：收到RequestVote，说明我发送的心跳信息，对方并没有收到，所以它的选举计时器到了，这或许是网络不稳定，又或许是网络分区又愈合。对于leader来说，和candidate的处理情况类似，如果对方的term比我高，那么我应该放弃leader角色，转为follower。但有如果对方的term和我相等，且我决定要为它投票，此时我也应该放弃leader角色，转为follower。

这部分的代码如下：

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock(rf.me, "RequestVote")
	defer rf.mu.Unlock(rf.me, "RequestVote") // this procedure isn't expensive, so give it a big latch
	DPrintf("[%d.%d.%d] grant vote for [%d] start... args:%+v \n", rf.me, rf.role, rf.currentTerm, args.CandidateId, args)

	defer func() {
		if rf.pendingPersist {
			rf.persist()
			rf.pendingPersist = false
		}
	}()

	reply.Term = rf.currentTerm
	reply.VoteGranted = false

	if args.Term < rf.currentTerm {
		return
	}
	if args.Term > rf.currentTerm {
		rf.turnOnPendingPersist(rf.setNewTerm, args.Term)
		// convert to follower
		if rf.role != FOLLOWER {
			rf.changeRoleTo(FOLLOWER)
		}
	}

	if rf.votedFor != -1 { // voted before
		DPrintf("[%d.%d.%d] grant vote for [%d] failed: voted for [%d] before \n", rf.me, rf.role, rf.currentTerm, args.CandidateId, rf.votedFor)
		return
	}
	// check log is at least up-to-date or not
	if len(rf.log) != 0 {
		lastLogEntry := rf.log[len(rf.log)-1]
		DPrintf("[%d.%d.%d] vote for [%d] log check start, my lastLogEntry is %+v\n", rf.me, rf.role, rf.currentTerm, args.CandidateId, lastLogEntry)
		if lastLogEntry.Term > args.LastLogTerm ||
			(lastLogEntry.Term == args.LastLogTerm && len(rf.log) > args.LastLogIndex+1) {
			DPrintf("[%d.%d.%d] vote for [%d] log check check failed, my log content %+v\n", rf.me, rf.role, rf.currentTerm, args.CandidateId, rf.log)
			return
		}
	}
	rf.turnOnPendingPersist(rf.setVoteFor, args.CandidateId)
	rf.resetElectionTimer()
	if rf.role == LEADER {
		// now all checks pass, I will vote for other svr, so  give up my leader role
		rf.changeRoleTo(FOLLOWER)
	}
	// ok,  grant vote for it
	reply.VoteGranted = true

	// for debug
	lastLogEntry := LogEntry{}
	if len(rf.log) > 0 {
		lastLogEntry = rf.log[len(rf.log)-1]
	}
	DPrintf("[%d.%d.%d] grant vote for [%d] success, my last log content %+v, arg last term (%d), last log index(%d)\n", rf.me, rf.role, rf.currentTerm, args.CandidateId, lastLogEntry, args.Term, args.LastLogIndex)
}
```

### 2.3 心跳

心跳是raft中最为复杂的部分，因为它承担了两个工作：1. 保证自己的leader身份不丢失；**2. log replica**。

如果只是为了保证leader身份不丢失，那么是比较简单的，定期发送心跳，对方收到心跳后，Reset选举定时器即可。但是如果包含了log replica，相对来说就比较难以实现了。下面重点说说log replica的过程。

#### 2.3 发送端

采用了和选举时相同的trick。

![raft_fireAppendEntries.excalidraw](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/raft_fireAppendEntries.excalidraw.svg)

```go
// replica log entires and used as heart beat
func (rf *Raft) fireAppendEntries(fromHeartBeatTimer bool) {
	rf.mu.Lock(rf.me, "fireAppendEntries")
	if fromHeartBeatTimer && rf.hBStopByCommitOp {
		rf.mu.Unlock(rf.me, "fireAppendEntries")
		return
	}
	pendingCommitIndex := len(rf.log) - 1
	curTerm := rf.currentTerm
	rf.mu.Unlock(rf.me, "fireAppendEntries")
	// use the same trick in `fireElection`
	ch := make(chan AppendEntriesReplyInCh)
	// send requests
	go rf.doSendAppendEntries(ch, curTerm, pendingCommitIndex)
	// receive
	go rf.doReceiveAppendEntries(ch, curTerm, pendingCommitIndex)
}
```

##### 发起RPC

当发起RPC用于log复制时，最重要的是，我们**应该复制哪一段log？**每次都全量发送log显然是不合适的，在raft中有一个 **nextIndex[]** 数组，用于存放给每个svr发送log时的log起点位置。 但需要注意的是，发起rpc这个过程同样是可能同时进行多轮的，当下一轮发起时，解决的log内容可能就和前一轮不同了。所以我们依然需要记录发起rpc时的term，当收到rpc响应时，只有最新的term才可以做处理。除了term外，还应该**保存发起rpc时的log长度，用于后续commitIndex更新。**

**当收到的超时的rpc数量超过总svr数量的一半时，我们有理由相信自己已经被网络隔离开，此时应当退位**。

这部分代码如下：

```go
func (rf *Raft) doSendAppendEntries(replyCh chan AppendEntriesReplyInCh, curTerm, pendingCommitIndex int) {
	rf.mu.Lock(rf.me, "doSendAppendEntries")
	defer rf.mu.Unlock(rf.me, "doSendAppendEntries")
	var sendRpcNum int32 = 0 // use this to determine when to close channel
	var timeoutCnt int32 = 0
	var timeoutOnce sync.Once
	// for debug
	// curTerm := rf.currentTerm
	curRole := rf.role
	nowTime := time.Now().UnixNano()

	for i := 0; i < len(rf.peers) && rf.role == LEADER; i++ {
		if i == rf.me {
			continue
		}
		if rf.currentTerm != curTerm {
			close(replyCh) // close channel to exit Collection(receive) routine
			break
		}

		prevLogTerm := -1
		prevLogIndex := -1
		var entries []LogEntry
		if rf.nextIndex[i] > 0 {
			DPrintf("[%d] build nextIndex args: log len:%d, nextIndex[%d]=%d\n", rf.me, len(rf.log), i, rf.nextIndex[i])
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
		DPrintf("[%d.%d.%d] [%v] --> [%d] sendAppendEntries, total log len:%d, args:%+v, nextIndex[%d]=%d, pendingCommitIndex=%d\n", rf.me, rf.role, rf.currentTerm, nowTime, i, len(rf.log), args, i, rf.nextIndex[i], pendingCommitIndex)
		go func(svrId int) {
			var reply AppendEntriesReply
			ok := rf.sendAppendEntries(svrId, args, &reply)
			DPrintf("[%d.%d.%d] [%v] --> [%d] AppendEntries Done, reply:%+v\n", rf.me, curRole, curTerm, nowTime, svrId, reply)
			if ok {
				rf.mu.Lock(rf.me, "doSendAppendEntries1")
				if !rf.killed() {
					rf.mu.Unlock(rf.me, "doSendAppendEntries1")
					replyCh <- AppendEntriesReplyInCh{
						AppendEntriesReply: reply,
						svrId:              svrId,
					}
					rf.mu.Lock(rf.me, "doSendAppendEntries1")
				}
				rf.mu.Unlock(rf.me, "doSendAppendEntries1")
			} else {
				DPrintf("[%d] timeout\n", rf.me)
				if 2*atomic.AddInt32(&timeoutCnt, 1) > int32(len(rf.peers)) {
					timeoutOnce.Do(func() {
						rf.mu.Lock(rf.me, "doSendAppendEntries1")
						// seems like I lost the leader relationship (maybe network partition, or maybe I am sole now)
						if curTerm == rf.currentTerm && rf.role == LEADER && !rf.killed() {
							DPrintf("[%d] commit operation failed, can't get majority rsp\n", rf.me)
							rf.changeRoleTo(FOLLOWER) // consider to execute only once?
						}
						rf.mu.Unlock(rf.me, "doSendAppendEntries1")
					})
				}
			}
			cnt := atomic.AddInt32(&sendRpcNum, 1) // NOTE: sendPrcNum++ here to prevent close opeartion(below) was executed ahead of replyCh channel due to no one accpet reply
			if int(cnt) == len(rf.peers)-1 {
				close(replyCh)
				// DPrintf("[%d.%d.%d] SendAppendEntries close reply channel\n", rf.me, curRole, curTerm)
			}
		}(i)
	}
	DPrintf("[%d.%d.%d] main routine of SendAppendEntries exists, sub routine unsure \n", rf.me, rf.role, rf.currentTerm)
}
```

另外要注意的是，log截取时**，应该做一次全量拷贝** (其实也可以不用，这里主要是因为lab的raft实现，是模拟的网络，所有svr都在一个进程内，如果不全量拷贝，可能存在DATA RACE问题）。

##### 接收RPC响应

接收RPC响应应该是全篇最难的部分，涉及到 **角色退位**、 **nextIndex快速回退**, **raft paer figure 8解决方案** 三个难点，先贴代码然后再解释：

```go
func (rf *Raft) doReceiveAppendEntries(replyCh <-chan AppendEntriesReplyInCh, curTerm, pendingCommitIndex int) {
	var succOne sync.Once
	// var failOne sync.Once
	successCnt := 1
	// for debug
	nowTime := time.Now().UnixNano()

	defer func() {
		rf.mu.Lock(rf.me, "doReceiveAppendEntries2")
		if rf.pendingPersist {
			rf.persist()
			rf.pendingPersist = false
		}
		rf.mu.Unlock(rf.me, "doReceiveAppendEntries2")
	}()

	for {
		reply, ok := <-replyCh
		if !ok {
			return
		}

		rf.mu.Lock(rf.me, "doReceiveAppendEntries1")
		if reply.Term > rf.currentTerm {
			DPrintf("[%d.%d.%d] --> [%d] , remote term %d is large than me \n", rf.me, rf.role, rf.currentTerm, reply.svrId, reply.Term)
			rf.turnOnPendingPersist(rf.setNewTerm, reply.Term)
			rf.changeRoleTo(FOLLOWER) // someone's term is large than me, so give up leader role and change to follower
			rf.mu.Unlock(rf.me, "doReceiveAppendEntries1")
			continue
		}

		// NOTE:update nextIndex and matchIndex even though the reply is stale data
		rf.updateNextIndex(reply, pendingCommitIndex)

		// discard stale rsp
		if curTerm != rf.currentTerm || rf.role != LEADER || rf.killed() {
			rf.mu.Unlock(rf.me, "doReceiveAppendEntries1")
			continue
		}

		if reply.Success {
			successCnt++
		}
		if 2*successCnt > len(rf.peers) {
			if successCnt == len(rf.peers) { // all svrs hold the logs precedeing(contains) `pendingCommitIndex`
				// NOTE: this branch is used to handle the scenario that receive `rf.log[pendingCommitIndex].Term < curTerm`
				// but no curTerm log entry anymore, which will cause the logs precedeing pendingCommitIndex never be
				// committed and client will  timeout
				if rf.commitIndex < pendingCommitIndex {
					DPrintf("[%d] [%v] all svrs hold the log precedeing(contain) index %d\n", rf.me, nowTime, pendingCommitIndex)
					rf.commitLog(pendingCommitIndex)
				}
				// }
			} else {
				succOne.Do(func() {
					rf.hBStopByCommitOp = false                // force to reset preempByCommit so that heartBeatTimer can send msg
					if rf.commitIndex >= pendingCommitIndex || // exclude later heartbeat commit first
						rf.log[pendingCommitIndex].Term != curTerm { // NOTE:!!!! raft paper figure 8 prevention, we can't commit log entry from previous term!!!!
						return
					}
					rf.commitLog(pendingCommitIndex)
				})
			}
		}
		rf.mu.Unlock(rf.me, "doReceiveAppendEntries1")
	}
}
```

首先是 leader角色退位， 如果**收到的rpc响应的term比自身还高，需要主动退位**：

```go
if reply.Term > rf.currentTerm {
    DPrintf("[%d.%d.%d] --> [%d] , remote term %d is large than me \n", rf.me, rf.role, rf.currentTerm, reply.svrId, reply.Term)
    // for debug
    if reply.Success {
        log.Panicf("reply.Term is large	than me, but reply.Success is true")
    }
    rf.turnOnPendingPersist(rf.setNewTerm, reply.Term)

    rf.changeRoleTo(FOLLOWER) // someone's term is large than me, so give up leader role and change to follower
    rf.mu.Unlock(rf.me, "doReceiveAppendEntries1")
    continue
}
```

接着**可能需要更新nextInde**x, 关于nextIndex的更新，看后文。

然后是最新的term判定，拦截旧RPC响应，*为什么拦截不在updateNextIndex之前？这里其实是一个对网络不稳定时的一个优化。可以忽略。*

```go
if curTerm != rf.currentTerm || rf.role != LEADER || rf.killed() {
    rf.mu.Unlock(rf.me, "doReceiveAppendEntries1")
    // return
    continue // if we return now, replyCh will block some routines
}
```

如果**收到半数以上的成功响应,** 则可尝试commitLog:

```go
	succOne.Do(func() {
					rf.hBStopByCommitOp = false // force to reset preempByCommit so that heartBeatTimer can send msg
					if curTerm != rf.currentTerm || rf.role != LEADER || rf.killed() ||
						rf.commitIndex >= pendingCommitIndex || // exclude later heartbeat commit first
						rf.log[pendingCommitIndex].Term != curTerm { // NOTE:!!!! raft paper figure 8 prevention, we can't commit log entry from previous term!!!!
						// return
						return
					}
					rf.commitLog(pendingCommitIndex)
				})
```

这里的重点在于有两个：

```go
	rf.commitIndex >= pendingCommitIndex || // exclude later heartbeat commit first
    rf.log[pendingCommitIndex].Term != curTerm // NOTE:!!!! raft paper figure 8 prevention, we can't commit log entry from previous term!!!!
```

第一个判断用于**预防旧rpc的pendingCommitIndex影响最新的commitIndex**， 我们应该注意到，对于commitIndex来说，只能增长，不能回退。

第二判定用于解决 raft paper figure 8的所提到的问题，即leader要commit的log entry，这个log entry的Term一定要和当前的curTerm相同，否则不予提交（原因如下图)。

<img src="C:\Users\Raven\AppData\Roaming\Typora\typora-user-images\image-20220308092751973.png" alt="image-20220308092751973" style="zoom:80%;" />

然而正是因为第二个判断**可能导致一些log永远不被提交**，比如当前curTerm一直接收不到命令，也就无法发起rpc，更无法收到rpc的响应。所以才多了如下的代码：

```go
if successCnt == len(rf.peers) { // all svrs hold the logs precedeing(contains) `pendingCommitIndex`
        // NOTE: this branch is used to handle the scenario that receive `rf.log[pendingCommitIndex].Term < curTerm`
        // but no curTerm log entry anymore, which will cause the logs precedeing pendingCommitIndex never be
        // committed and client will  timeout
        if curTerm == rf.currentTerm && rf.role == LEADER && !rf.killed() && rf.commitIndex < pendingCommitIndex {
        DPrintf("[%d] [%v] all svrs hold the log precedeing(contain) index %d\n", rf.me, nowTime, pendingCommitIndex)
        rf.commitLog(pendingCommitIndex)
    }
} 
```

当所有svr都回复了成功，也即所有svr都有了该条log的副本，此时我们可以提交该log。

**提交log的代码如下**

```go
// NOTE: must be guarded by rf.mu
func (rf *Raft) commitLog(pendingCommitIndex int) {
	// for debugging
	if rf.commitIndex >= pendingCommitIndex {
		log.Panicf("[%d] rf.commitIndex(%d) is larger equal than pendingCommitIndex(%d)\n", rf.me, rf.commitIndex, pendingCommitIndex)
	}
	rf.commitIndex = pendingCommitIndex
	DPrintf("[%d.%d.%d] committedIndex to %d\n", rf.me, rf.role, rf.currentTerm, rf.commitIndex)
	DPrintf("[%d.%d.%d] get majority of appendentries rsp, change committedIndex to %d\n", rf.me, rf.role, rf.currentTerm, pendingCommitIndex)
	// notify applier
	rf.committedChangeCond.Broadcast()
	rf.hBStopByCommitOp = true
	rf.mu.Unlock(rf.me, "doReceiveAppendEntries1")
	// fire next turn AppendEntries to tell others to commit
	rf.fireAppendEntries(false)
	rf.mu.Lock(rf.me, "doReceiveAppendEntries3") // TODO:lock followed unlock immediately, how to fix this? use `go rf.fireAppendEntries` is another way, but launch a new goroutine is costly too
}

```

这里的重点有两个：

1. commit，应该立即通知上层服务，即代码中的 `notify applier`注释。关于该部分看后文。
2. 立即发起第二次rpc，通知其他svr可以commit上一轮rpc拷贝的logs。

**现在关于发送端还有两个重点：**

1. 如何apply？
2. 如何快速updateNextIndex?

我们先谈第一个点，第二个需要涉及到接收端。

对于第一个问题，我的做法是开一个单独的线程来做, 这里的重点是如何确定要apply哪一段log？ 这里需要用到rf中的 lastAppliedIndex 字段了。

```go
func (rf *Raft) applier() {
	DPrintf("[%d] applier start...\n", rf.me)
	for {
		rf.mu.Lock(rf.me, "applier")
		for rf.lastAppliedIndex == rf.commitIndex && !rf.killed() {
			rf.committedChangeCond.Wait()
		}
		if rf.killed() {
			rf.mu.Unlock(rf.me, "applier")
			break
		}
		// for debugging
		if rf.lastAppliedIndex > rf.commitIndex {
			log.Panicf("[%d] lastAppeileIndex is larger than commitIndex, raft information:%+v\n", rf.me, rf)
		}

		commitIndex := rf.commitIndex
		lastAppliedIndex := rf.lastAppliedIndex + 1 // advance one step
		pendingApplyLog := make([]LogEntry, 0)      // copy log entries to avoid data race
		pendingApplyLog = append(pendingApplyLog, rf.log[lastAppliedIndex:commitIndex+1]...)
		DPrintf("[%d] applier try to apply log[%d-%d], log info%+v\n", rf.me, lastAppliedIndex, commitIndex, pendingApplyLog)
		rf.mu.Unlock(rf.me, "appiler")

		for i := 0; i < len(pendingApplyLog); i++ {
			applyMsg := ApplyMsg{
				CommandValid: true, // NOTE: how to use this? maybe need to fix this in the future
				Command:      pendingApplyLog[i].Command,
				CommandIndex: lastAppliedIndex + 1 + i, // raft's log id starts from 0, but service starts from 1, so fix it by plus 1
			}
			rf.applyCh <- applyMsg
			DPrintf("[%d] applier feeds command, applymsg:%+v\n", rf.me, applyMsg)
		}

		rf.mu.Lock(rf.me, "applier")
		rf.lastAppliedIndex = commitIndex
		DPrintf("[%d] rf.lastAppliedIndex=%d\n", rf.me, rf.lastAppliedIndex)
		DPrintf("[%d] applier finished\n", rf.me)
		rf.mu.Unlock(rf.me, "applier")
	}
}

```

另外，注意的加锁解锁区间，应该避免在apply channel时还持有锁，在lab3是会造成死锁的。

### 2.4 接收端

接收端的逻辑不算复杂，只用照着 raft paper figure 2写下来即可。重点是关于 **updateNextIndex 快速回退的**部分，不过这部分会后文讲解。下面我们依然代入三种角色来看如何处理这个handle:

1. follower：收到该信息，需要重置自己的选举定时器，同时通过一定的验证后，截取拷贝传递进来的log，根据传递进来的commitIndex，更新自己的commitIndex，并唤醒applier，apply log。
2. candidate：和follower类似，收到信息后，需要放弃自己的candidate角色，然后转入follower角色处理逻辑
3. leader：收到该信息，说明网络曾经发生过分区，现在愈合后，网络中至少存在两个leader，如果对方的term大于我，我需要放弃leader角色，然后传入follower角色，如果对方的term小于我，将我自己的term传递给他，让他放弃leader角色。

下面是整个代码：

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArg, reply *AppendEntriesReply) {
	DPrintf("[%d] EnterAppendEntries, args%+v\n", rf.me, args)
	rf.mu.Lock(rf.me, "AppendEntries")
	defer rf.mu.Unlock(rf.me, "AppendEntries")

	defer func() {
		if rf.pendingPersist {
			rf.persist()
			rf.pendingPersist = false
		}
	}()

	DPrintf("[%d.%d.%d] AppendEntriesRPC start from leader [%d], commitIndex=%d log len:%d, log info %v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, rf.commitIndex, len(rf.log), rf.log)
	// for debug
	// if rf.role == LEADER && rf.currentTerm >= args.Term {
	// 	log.Panicf("[%d.%d.%d] AppendEntriesRPC start from leader [%d], my term is larger than args.Term(%d)\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, args.Term)
	// }
	// init reply
	rf.resetElectionTimer()
	reply.Term = rf.currentTerm
	reply.Success = false
	reply.XTerm = -1
	reply.XIndex = -1
	reply.XLen = -1

	// 1. Reply false if term < currentTerm (§5.1)
	if args.Term < rf.currentTerm {
		DPrintf("[%d.%d.%d] AppendEntriesRPC from leader [%d], args Term less than me %v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, args.Term)
		return
	}

	// follow all server rule
	if args.Term > rf.currentTerm {
		rf.turnOnPendingPersist(rf.setNewTerm, args.Term)
	}
	if rf.role != FOLLOWER {
		rf.changeRoleTo(FOLLOWER)
	}
	if rf.votedFor != args.LeaderId {
		rf.turnOnPendingPersist(rf.setVoteFor, args.LeaderId)
	}

	logChanged := false // use this to avoid unnecessary persist
	if args.PrevLogIndex != -1 {
		// 2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3)
		if len(rf.log)-1 < args.PrevLogIndex {
			// case 3: rf.log is too short
			reply.XLen = len(rf.log)
			DPrintf("[%d.%d.%d] my log len is too short", rf.me, rf.role, rf.currentTerm)
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
			DPrintf("[%d.%d.%d] set XTerm(%d) xIndex(%d)", rf.me, rf.role, rf.currentTerm, reply.XTerm, reply.XIndex)
			return
		}
		// check pass, cut rf.log if len(rf.log) -1 != args.PrevLogIndex
		if len(rf.log)-1 != args.PrevLogIndex {
			logChanged = true
			rf.log = rf.log[:args.PrevLogIndex+1]
		}
	} else {
		// clear log, because leader sent a full log
		if len(rf.log) != 0 {
			logChanged = true
			rf.log = []LogEntry{}
		}
	}
	// 4. Append any new entries not already in the log
	if logChanged || len(args.Entries) != 0 {
		rf.turnOnPendingPersist(rf.setLog, append(rf.log, args.Entries...))
	}
	// 5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)
	if args.LeaderCommit > rf.commitIndex {
		if args.LeaderCommit > len(rf.log)-1 {
			if rf.commitIndex > len(rf.log)-1 {
				log.Panicf("[%d] rf.commitIndex(%d) is less than len of rf.log -1(%d)\n", rf.me, rf.commitIndex, len(rf.log)-1)
			}
			rf.commitIndex = len(rf.log) - 1
		} else {
			rf.commitIndex = args.LeaderCommit
		}
		// notify applier
		rf.committedChangeCond.Broadcast()
		// NOTE: maybe do a batch persist?
		DPrintf("[%d] rf.commitIndex=%d \n", rf.me, rf.commitIndex)
		DPrintf("[%d] take AppendEntries RPC, now change commitIndex to %d\n", rf.me, rf.commitIndex)
	}
	reply.Success = true
	DPrintf("[%d.%d.%d] AppendEntriesRPC end from leader [%d],  commitIndex=%d, log len:%d, log info %v\n", rf.me, rf.role, rf.currentTerm, args.LeaderId, rf.commitIndex, len(rf.log), rf.log)
}
```

#### updateNextIndex

为了实现快速updateNextIndex, 这里采用了课堂中的方案。

Follower在回复Leader的AppendEntries消息中，需要携带3个额外的信息，来加速日志的恢复。这里的三个信息是指：

- XLen: 如果follower的log长度不够长，在PrevLogIndex处没有log，则设置XLen为当前的log长度。
- XTerm, 如果follower在PrevLogIndex处的Term不等于PrevLogTerm，设置该Xterm为follower在PrevLogIndex处的Term。 
- XIndex：如果follower在PrevLogIndex处的Term不等于PrevLogTerm，记录该位置的Term，并向前回溯，找到第一次出现该Term的log Index， 设置该Index = XIndex。

当leader拿到这三个消息后：

- XLen不为空，设置nextIndex=Xlen
- XTerm不为空，且XIndex处的log Term ! = XTerm，设置nextIndex = XIndex 
- XTerm不为空，且Xindex处的log Term == XTerm，从XIndex位置处向后查询，找到最后一个出现XTerm的位置Index，设置nextIndex = Index + 1

代码实现：

```go
// must be guarded by rf.mu
func (rf *Raft) updateNextIndex(reply AppendEntriesReplyInCh, pendingCommitIndex int) {
	if !reply.Success {
		if requestTimeOut(reply) { // success = false, and both equal to 0, assume this is a timeout
			DPrintf("[%d.%d.%d] --> [%d], reply unsuccess timeout\n", rf.me, rf.role, rf.currentTerm, reply.svrId)
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
				tmpNextIndex := rf.nextIndex[reply.svrId]
				rf.nextIndex[reply.svrId] = i + 1
				i++
				for i < tmpNextIndex && rf.log[i].Term == reply.XTerm {
					rf.nextIndex[reply.svrId] = i + 1
					i++
				}
				DPrintf("[%d.%d.%d] --> [%d] reply unsuccess(case 2, XTerm == log.Term), set nextIdx=%d\n", rf.me, rf.role, rf.currentTerm, reply.svrId, rf.nextIndex[reply.svrId])
			} else {
				rf.nextIndex[reply.svrId] = i
				DPrintf("[%d.%d.%d] --> [%d] reply unsuccess(case 1, XTerm != log.Term), set nextIdx=%d\n", rf.me, rf.role, rf.currentTerm, reply.svrId, rf.nextIndex[reply.svrId])
			}
		}
		if rf.nextIndex[reply.svrId] <= rf.matchIndex[reply.svrId] {
			rf.nextIndex[reply.svrId] = rf.matchIndex[reply.svrId] + 1
		}
		// rf.nextIndex[reply.svrId]-- // NOTE: old scheme
	} else {
		rf.nextIndex[reply.svrId] = pendingCommitIndex + 1
		rf.matchIndex[reply.svrId] = rf.nextIndex[reply.svrId] - 1 // NOTE: maybe a bug here
		DPrintf("[%d] update nextIndex: matchIndex[%d]=%d, nextIndex[%d]=%d\n", rf.me, reply.svrId, rf.matchIndex[reply.svrId], reply.svrId, rf.nextIndex[reply.svrId])
	}
}
```

除了基本的加速方案实现外，还应该注意到这几行代码：

```go
if rf.nextIndex[reply.svrId] <= rf.matchIndex[reply.svrId] {
    rf.nextIndex[reply.svrId] = rf.matchIndex[reply.svrId] + 1
}
```

这是为了避免**收到旧的响应消息（但是term 依然等于 curTerm）时**， nextIndex 回退过度（不能低于 matchIndex])。

最后一个没有提到的问题是，**nextIndex的初始化在哪里 ？**

当每个svr变为leader时，都会对nextIndex初始化， 初始化为matchIndex的后一位即可。

```go
// NOTE: must be guarded by rf.mu
func (rf *Raft) resetNextIndex() {
	// debug
	if rf.role != LEADER {
		log.Panicf("[%d.%d.%d] non-leader role raft can't resetNextIndex", rf.me, rf.role, rf.currentTerm)
	}
	for i := 0; i < len(rf.peers); i++ {
		rf.nextIndex[i] = rf.matchIndex[i] + 1
	}
	DPrintf("[%d.%d.%d] reset all nextIndex to %d", rf.me, rf.role, rf.currentTerm, len(rf.log)-1)
}
```

### 2.5 持久化

持久化相对简单，不算是lab2的核心，关于Snapshot会在lab3中讲解。

这里直接看旧博文 [lab2C](https://ravenxrz.github.io/archives/5fad6910.html)

## 3. 结语

到这里，总算完结了raft篇。基本记录下了raft实现过程中的各种问题，以及细节处理。后面在写lab3吧。

