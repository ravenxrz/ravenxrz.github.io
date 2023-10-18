---
title: MIT-6824-lab2C-Raft
categories: MIT6824
abbrlink: 5fad6910
date: 2022-02-24 22:10:31
tags:
---

## 0. 要求

本次lab为raft的第三分部(PartC), 原文要求[在这里](http://nil.csail.mit.edu/6.824/2020/labs/lab-raft.html)

简述lab2C的要求为：

> 实现 Raft状态的持久化，保证svr crash重启后能恢复到之前的状态。

在做完2B后，2C非常简单。

<!--more-->

## 1. 实现

**本文并非最终实现，最终实现在[这里](https://ravenxrz.github.io/archives/d16f195.html)**

根据raft paper figure 2提到的需要持久化的状态只有3个。如下图：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220225115450978.png)

所以只用在适当的地方做持久化即可。

lab2C的持久化方式是模拟的。并不说真的将数据存入到disk，而是用一个 Persister 类来模拟。总体来说我们要做的工作只有两个：

1. 实现 `persist` 和 `readPersist` 两个接口
2. 在适当地方做persist

先说这两个接口，代码如下：

```go
//
// save Raft's persistent state to stable storage,
// where it can later be retrieved after a crash and restart.
// see paper's Figure 2 for a description of what should be persistent.
//
// NOTE: must be guarded by rf.mu
func (rf *Raft) persist() {
	DPrintf("[%d] persist state, curTerm %d, voteFor %d, log %+v\n", rf.me, rf.currentTerm, rf.votedFor, rf.log)
	// Your code here (2C).
	// Example:
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log) 
	data := w.Bytes()
	rf.persister.SaveRaftState(data)
}

//
// restore previously persisted state.
//
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // bootstrap without any state?
		return
	}
	DPrintf("[%d] read state from storage\n", rf.me)
	// Your code here (2C).
	// Example:
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var currentTerm int
	var voteFor int
	var log []LogEntry
	if d.Decode(&currentTerm) != nil ||
		d.Decode(&voteFor) != nil ||
		d.Decode(&log) != nil {
		DPrintf("[%d] [Error]: read state failed", rf.me)
	} else {
		// NOTE: need latch or not? it depends on the caller
		rf.currentTerm = currentTerm
		rf.votedFor = voteFor
		rf.log = log
		DPrintf("[%d] read state from storage, %+v\n", rf.me, rf)
	}
}

```

再说哪些地方需要做persist.

显然是之前提到的三个变量被修改的地方，只是不一定每次修改都需要立即persist，可以做batch persist。 

简单说一下我个人的实现， 我为Raft结构体添加了 `pendingPersist` 变量，用于表示当前是否需要做persist，然后封装了对以上3个变量的Set方法，接着对pendingPersist的setter也做一层包装，如下：

```go
// NOTE: must be guarded by rf.mu
func (rf *Raft) setNewTerm(arg interface{}) {
	term, ok := arg.(int)
	if !ok {
		log.Panicf("call setNewTerm failed, invalid type: %+v", arg)
	}
	rf.currentTerm = term
	DPrintf("[%d.%d.%d] set new term\n", rf.me, rf.role, rf.currentTerm)
	rf.votedFor = -1
	DPrintf("[%d.%d.%d] reset vote to -1 \n", rf.me, rf.role, rf.currentTerm)
}

// NOTE: must be guarded by rf.mu
func (rf *Raft) setVoteFor(arg interface{}) {
	voteFor, ok := arg.(int)
	if !ok {
		log.Panicf("call setVoteFor failed, invalid type: %+v", arg)
	}
	rf.votedFor = voteFor
}

// NOTE: must be guarded by rf.mu
func (rf *Raft) setLog(arg interface{}) {
	logVal, ok := arg.([]LogEntry)
	if !ok {
		log.Panicf("call setLog failed, invalid type: %+v", arg)
	}
	rf.log = logVal
}

func (rf *Raft) turnOnPendingPersist(setter func(val interface{}), val interface{}) {
	rf.pendingPersist = true
	setter(val)
}
```

接着，只用在适当函数的结尾处判定 `pendingPersist` 然后做 persist即可，之所以在函数结尾处，是为了一定程度上的batch，如 `doReceiveRequestVoteReply` :

```go
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
		// if rf.killed() {
		// 	close(done) // actually we don't need this, now that rf is killed, blocked routines will be reclaimed by GC or OS
		// 	return
		// }
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

这里采用了 defer 来做。

其余地方类似，不再赘述。

至此，整个lab2全部完成。

## 2. 其他

在raft paper figure2中还存在这样一段话：

![](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220225124200638.png)

这里我是采用了一个专用后台线程，每隔一段时间就去扫描一次。代码如下：

```go
func (rf *Raft) leaderCommitIndexChecker() {
	DPrintf("[%d] leaderCommitIndexChecker start...\n", rf.me)
	for {
		time.Sleep(commitCheckerTimeout * time.Millisecond)
		rf.mu.Lock()
		for rf.role != LEADER && !rf.killed() {
			rf.roleChangeCond.Wait()
		}
		if rf.killed() {
			rf.mu.Unlock()
			break
		}
		// do check
		matchIndexCopy := make([]int, 0)
		matchIndexCopy = append(matchIndexCopy, rf.matchIndex...)
		sort.Ints(matchIndexCopy)
		startNidx := len(rf.peers) / 2
		// for debug
		if startNidx < 1 {
			log.Panicf("len of rf.peers is less than 3")
		}
		startN := matchIndexCopy[startNidx]
		endN := matchIndexCopy[startNidx-1]
		for N := startN; N >= endN; N-- {
			if N > rf.commitIndex && rf.log[N].Term == rf.currentTerm {
				DPrintf("[%d] leaderCommitIndexChecker found N=%d, commitIndex=%d\n", rf.me, N, rf.commitIndex)
				rf.commitIndex = N
				rf.committedChangeCond.Broadcast()
				break
			}
		}
		rf.mu.Unlock()
	}
}
```

不过就目前来说，并没有真正触发这个规则。

## 3. 回顾总结

首先是感谢mit6824提供了这样的课程和配套lab，整个做下来，收获良多。

lab2A是整个lab2的骨架，核心包含两个计时器, 一个用于选举，一个用于心跳(后面实验中也包含log replication).  选举中的一个注意点是要保证candidate的log必须和follower保持 at least up-to-date. 除此外，如果发生过网络分区，那旧leader也可能会收到来自新leader的心跳，此时要注意role的转换（目前我的策略是一旦发现有比自己更大的Term，则自动降级为Follower）。

Lab2B要求实现log replication，也是整个lab2中最困难的一部分，主要问题是如何保证log的最终一致性，如何加速log的最终一致这个过程。

Lab2C要求实现persist log，相对简单，不多赘述。

ok，最近要忙忙论文，下一个lab可能要又要鸽一段时间了。
