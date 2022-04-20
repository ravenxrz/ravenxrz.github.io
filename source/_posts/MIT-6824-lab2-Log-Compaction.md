---
title: MIT-6824-lab2-Raft-LogCompaction
categories: MIT6824
date: 2022-04-20 22:10:31
tags:
---



## 0. 前言

本文为Raft的补充篇。 我做的实验是mit 6824 spring 2020, 2020年版本Log Compaction放到了lab3中，但是我发现 2021版本Log Compaction放到了lab 2中，且有配到单元测试，所以这部分单独写了2021版本。现在单独写篇文章说下log compaction。

简述要求：

> raft的log随着svr服务时间的延长而逐渐变大，如若不加任何控制，系统内log占用空间将无限增大，另外，svr每次开机也会反序列化log文件。所以需要在恰当的时间对log执行compaction操作，也被称为snapshot操作。

原文要求 ：http://nil.csail.mit.edu/6.824/2021/labs/lab-raft.html

另外，我的方案中未使用到 `CondInstallSnapshot` 接口。

<!--more-->

## 1. 题解思路

首先要把整个过程理清楚，下图贴出了raft原paper对log compaction示例图。

<img src="https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220420154100621.png" alt="image-20220420154100621" style="zoom: 67%;" />

某个时刻，由raft的上层服务向raft发起Log Compaction请求，这个请求会带上要将log压缩到哪个地方（即index和service状态数据），当raft收到该请求后，根据index将log做裁剪，并保留被裁剪部分的最后一个log entry的相关信息，即 `last included index` 和 `last included term`. 另外，将上层service状态（其实也就是snapshot）和raft此时的状态持久化，用于后续的重启恢复。当svr重启后，service课通过读取snapshot恢复至发起Log Compaction请求时的状态，*然后可重放raft log恢复至系统关机前的状态（此部分还未验证，待lab3做完后更新）*。

流程如上，相对简单，现在来思考一些问题（下文中Snapshot和Log Compaction为同义词）：

**1.什么时候需要Snapshot？**

由service层控制，和raft无关。

**2.怎么做Snapshot，需要删除哪些东西，需要更新哪些东西，哪些状态需要持久化？**

这里其实就对应到之前说的snapshot的流程。比如需要新添加last included index/term用于心跳边界条件判定，索引裁剪后的log需要特殊处理等。

**3.对于那些落后leader特别远的follower来说，leader因为Snapshot后，已经没有follower所需要的log，此时该如何处理？**

对于这种情况，需要 将leader的Snapshot整个发送给follower，follower收到Snapshot后 ，执行安装。 这里就涉及到 `InstallSnapshot` RPC接口设计和follower如何安装两个问题。

**4.leader什么时候发起Snapshot RPC？**

可在心跳时判定follower是否落后于自己过远，如果过远，就发送整个Snapshot。

xxx

整体来说， Log Compaction没有之前难度大，但是可能会牵一发而动全身，为实现Log Compaction可能造成之前的代码出现问题。

## 2. 实现

### 1. 虚拟log结构

关注上文描述，其中提到了log结构的索引方式会发生改变。因为log被会截断，但是这个截断操作对log的使用者来说应该是尽量无感的，也就是说 log[index] 这个操作，index不应该因为log被截断而手动计算更改。

为方便其他操作索引log，我的实现中添加了vlog结构。如下：

```go
type VLog struct {
	LastIncludedTerm  int        // last term at last index of log entries in snapshot
	LastIncludedIndex int        // last index of snapshot, default value: -1
	Logs              []LogEntry // log entries following snapshot
}
```

并且设计了如下接口：

```go
// return global log lengths containing snsapshot
func (vlog *VLog) length() int 

//
// check global log is empty or not?
//
func (vlog *VLog) empty() bool 

// get term number of global log at `idx` 
func (vlog *VLog) getTermAt(idx int) int 

//
// return global log entry at index 'idx'
// if vlog doesn't contain the log at index `idx`, `sucess` will be set to false
//
func (vlog *VLog) indexAt(idx int) (entry LogEntry, success bool) 
//
// set local log
//
func (vlog *VLog) setLog(logs []LogEntry)
//
// cut global logs from start to `end` but exclude log at end, i.e. get log[0,end)
// this does change vlog.log content, jsut return a reference of the range slice
//
func (vlog *VLog) cutLog(start int, end int) []LogEntry

// compact log up to `index` inclued `index`
// NOTE: must be guared by mutex
func (vlog *VLog) logCompaction(index int) 
```

上面的接口含有 详细注释，可以看到注释中包含 `local`和`global`两种字样：

1. global操作代表的是，操作的对象为整个vlog长度
2. local操作代表的是，操作的对象为实际的vlog.Logs对象。 目前仅有 `setLog`接口为local类型操作

实现如上接口后，将原lab2中所有和log相关的操作更改，并跑完以前的所有测试。

### 2. Snapshot接口实现

Snapshotshot由Service层调用，向Raft宣告，截至到index（包括）前的所有log状态都保存在了`snapshot`二进制数据中，现在由Raft做log截断，并更新相应的状态变量。

```go
func (rf *Raft) Snapshot(index int, snapshot []byte) {
	// Your code here (2D).
	index = index - 1 // fix index by minus 1
	// Do illegal check
	rf.mu.Lock(rf.me, "Snapshot")
	defer rf.mu.Unlock(rf.me, "Snpahost")
	// for debug
	if rf.lastAppliedIndex < index {
		log.Fatalf("[%d] lastAppliedIndex(%d) is less than index(%d)\n", rf.me, rf.lastAppliedIndex, index)
	}

	DPrintf("[%d.%d.%d] start Snapshot, last include index=%d, current log real len(%d), virual len(%d)\n", rf.me, rf.role, rf.currentTerm, index, len(rf.vlog.Logs), rf.vlog.length())
	if index < 0 || index <= rf.vlog.LastIncludedIndex || index >= rf.vlog.length() {
		log.Fatalf("[%d.%d.%d] Snapshot failed: index(%d) is illegal, current global log length(%d)\n", rf.me, rf.role, rf.currentTerm, index, rf.vlog.length())
	}
	// save new snapshot
	rf.snapshot = snapshot
  rf.vlog.logCompaction(index)
	DPrintf("[%d.%d.%d] take snapshot success, real log len(%v), virtual log len(%v)\n", rf.me, rf.role, rf.currentTerm, len(rf.vlog.Logs), rf.vlog.length())
	rf.persist()
}
```

首先执行一些合并检测，比如传入的index是否合法，然后就是修改相应的变量了。其实也就是更新vlog结构。最后持久化。

### 3. 修改持久化接口

由于修改了log结构，并添加了LastIncludedTerm/Index变量，所以持久化实现也需要修改。包括读取和存储了两部分。

```
// save Raft's persistent state to stable storage,
// where it can later be retrieved after a crash and restart.
// see paper's Figure 2 for a description of what should be persistent.
//
// NOTE: must be guarded by rf.mu
func (rf *Raft) persist() {
	DPrintf("[%d] persist state start, curTerm %d, voteFor %d, log %+v\n", rf.me, rf.currentTerm, rf.votedFor, rf.vlog.Logs)
	// Your code here (2C).
	// Example:
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.vlog)
	data := w.Bytes()
	rf.persister.SaveStateAndSnapshot(data, rf.snapshot)
	DPrintf("[%d] persist state end, curTerm %d, voteFor %d, log %+v\n", rf.me, rf.currentTerm, rf.votedFor, rf.vlog.Logs)
}

//
// restore previously persisted state.
//
func (rf *Raft) readPersist(data []byte) {
	// NOTE: no lock need actually
	// rf.mu.Lock(rf.me, "readPersist")
	// defer rf.mu.Unlock(rf.me, "readPersist")
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
	var vlog VLog
	if d.Decode(&currentTerm) != nil ||
		d.Decode(&voteFor) != nil ||
		d.Decode(&vlog) != nil {
		DPrintf("[%d] [Error]: read state failed", rf.me)
	} else {
		rf.currentTerm = currentTerm
		rf.votedFor = voteFor
		rf.vlog = vlog
		rf.lastAppliedIndex = rf.vlog.LastIncludedIndex		// !!
		rf.commitIndex = rf.vlog.LastIncludedIndex
		DPrintf("[%d] read state from storage, %+v\n", rf.me, rf)
	}
}


```

值得注意的是，读取持久化状态时，除了恢复vlog之后，还要恢复lastAppliedIndex和commitIndex为 lastIncluedIndex, 否则如果从头apply log，由于已经snapshot，是找不到开始段的log的。

### 4. InstallSnapshot操作

leader在做完Snapshot后，在某个时机还需要向落后很多的follower发送完整的Snapshot。这个过程由InstallSnapshot RPC完成。

```go
func (rf *Raft) InstallSnapshot(args *InstallSnapshotArg, reply *InstallSnapshotReply) {
	rf.mu.Lock(rf.me, "InstallSnapshot")
	defer rf.mu.Unlock(rf.me, "InstallSnapshot")
	defer func() {
		if rf.pendingPersist {
			rf.persist()
			rf.pendingPersist = false
		}
	}()
	DPrintf("[%d.%d.%d] InstallSnapshot, args%+v\n", rf.me, rf.role, rf.currentTerm, args)
	reply.Term = rf.currentTerm
	reply.Installed = false
	DPrintf("[%d.%d.%d] InstallSnapshot from leader(%d) starts , args%+v\n", rf.me, rf.role, args.LeaderId, rf.currentTerm, args)
	if rf.currentTerm < args.Term {
		return
	} else if rf.currentTerm > args.Term {
		rf.turnOnPendingPersist(rf.setNewTerm, args.Term)
	}
	// check do I have newer logs or not
	if args.LastIncludedIndex < rf.vlog.length()-1 {
		return
	}
	// assert args.lastAppliedIndex >= rfv.log.length() - 1
	// notify service
	DPrintf("[%d.%d.%d] InstallSnapshot apply snapshot to servie starts\n", rf.me, rf.role, rf.currentTerm)
	// We can't release lock. because we need guarantee service and raft state are consistent, which means this is a atomic ooperatoin
	rf.applyCh <- ApplyMsg{
		CommandValid:  false,
		SnapshotValid: true,
		Snapshot:      args.Snapshot,
		SnapshotTerm:  args.LastIncludedTerm,
		SnapshotIndex: args.LastIncludedIndex + 1,
	}
	// set state machine
	rf.commitIndex = args.LastIncludedIndex
	rf.lastAppliedIndex = args.LastIncludedIndex
	rf.votedFor = args.LeaderId
	rf.snapshot = args.Snapshot
	rf.changeRoleTo(FOLLOWER) // not neccessary, but recommend

	DPrintf("[%d.%d.%d] InstallSnapshot apply snapshot to servie end\n", rf.me, rf.role, rf.currentTerm)
	// use snapshot to update state machine
	logs := make([]LogEntry, 0)
	rf.vlog.setLog(logs)
	rf.vlog.LastIncludedTerm = args.LastIncludedTerm
	rf.vlog.LastIncludedIndex = args.LastIncludedIndex
	rf.pendingPersist = true // wait defer function to persist state
	DPrintf("[%d.%d.%d] InstallSnapshot end, my real log len:%d, virtual log len:%d, log content:%+v\n", rf.me, rf.role, rf.currentTerm, len(rf.vlog.Logs), rf.vlog.length(), rf.vlog.Logs)

	reply.Installed = true
}
```

这个函数基本流程：

1. 做一些合法性检验，其中最重要的是 ` args.LastIncludedIndex < rf.vlog.length()-1 ` 判定，因为follower在刚接收到多个InstallSnapshot请求，那么对于旧的Snapshot需要丢弃。
2. 接着需要应用Snapshot，这次应用不应该使用单独的线程，因为保证整个raft状态和service状态一致，也就是说apply操作和raft状态修改操作应该原子的。
3. 最后修改raft的状态即可。

### 6. 何时发起InstallSnapshot

我的做法是在发起AppendEntries RPC中做检测，也可以单独开启一个检测线程。看个人实现。

只要发送nextIndex[svrId] 小于当前最小 incluedIndex, 则发起InstallSnapshot请求。

```go
// send InstallSnapshot rpc if rf.nextIndex[i] is less or equal than lastIncludeIndex
if rf.nextIndex[i] <= rf.vlog.LastIncludedIndex {
    go func(svrId int) {
   		 rf.doSendSnpahost(svrId)
    }(i)
}
```

## 3. 其他优化记录

1. 修改心跳计时器线程唤醒逻辑。如下：

   ```go
   func (rf *Raft) heartBeatTimer() {
   	DPrintf("[%d] heartBeat start...\n", rf.me)
   	// timerId := time.Now().Unix()
   	ticker := time.NewTicker(heartBeatTimeout * time.Millisecond)
   	for {
   		rf.mu.Lock(rf.me, "heartBeatTimer")
   		for rf.role != LEADER && !rf.killed() {
   			rf.roleChangeCond.Wait()
   		}
   		rf.mu.Unlock(rf.me, "heartBeatTimer")
   		if rf.killed() {
   			break
   		}
   
   		select {
   		case <-ticker.C:
   			DPrintf("heartBeatTimer is wakeed up by ticker")
   		case <-rf.hBChan:
   			DPrintf("heartBeatTimer is wakeed up by rf.hBChan")
   		}
   
   		// time.Sleep(heartBeatTimeout * time.Millisecond)
   		// fire
   		DPrintf("[%d] fireAppendEntriesRPC\n", rf.me)
   		rf.fireAppendEntries()
   	}
   	// DPrintf("[hb id %d] exits\n", timerId)
   }
   ```

   目前可通过ticker或者通过通道强制唤醒。 添加通道强制唤醒的目的是为了加速心跳过程。比如leader收到了大多数follower发来成功接收新的log entry，此时需要再向各follower发起请求来更新commitIndex。如果采用通道唤醒可以做到立即发送，而不需要等到一轮心跳时间。

2. 针对nextIndex的快速回退优化，之前的系列文章中，采用了nextIndex快速回退的方法，但是在更新nextIndex后需要再等待一轮心跳时间，现在改为，一旦nextIndex被更新，立即发起AppendEntries，而不用等到一轮心跳时间。

   ```go
   rf.updateNextMatchIndex(replyInChan, pendingCommitIndex)
   
   // fire a new trun AppendEntriesRPC for svrId, AppendEntriesRPC may be executed many times
   go rf.reSendAppendEntries(replyInChan.svrId, pendingCommitIndex)
   
   func (rf *Raft) reSendAppendEntries(svrId, pendingCommitIndex int) {
   	retryTimes := retryHBWhenTimeout
   	rf.mu.Lock(rf.me, "reSendAppendEntries")
   	// Rebuild args
   	prevLogTerm := -1
   	prevLogIndex := -1
   	var entries []LogEntry
   
   	// install snapshot
   	if rf.nextIndex[svrId] <= rf.vlog.LastIncludedIndex {
   		rf.mu.Unlock(rf.me, "reSendAppendEntries")
   		DPrintf("[%d] -> [%d] installsnap in Resend AppendEntries, updateNextIndex[%d]=%d\n", rf.me, svrId, svrId, rf.nextIndex[svrId])
   		rf.doSendSnpahost(svrId)
   		return
   	}
   
   	// normal append log entries
   	if rf.nextIndex[svrId] > 0 {
   		DPrintf("[%d] build nextIndex args: log len:%d, nextIndex[%d]=%d\n", rf.me, rf.vlog.length(), svrId, rf.nextIndex[svrId])
   		prevLogTerm = rf.vlog.getTermAt(rf.nextIndex[svrId] - 1)
   		prevLogIndex = rf.nextIndex[svrId] - 1
   	}
   	// build log entries that need to be send
   	if rf.nextIndex[svrId] <= pendingCommitIndex && pendingCommitIndex >= 0 {
   		entries = make([]LogEntry, 0) // do a copy
   		entries = append(entries, rf.vlog.cutLog(rf.nextIndex[svrId], pendingCommitIndex+1)...)
   		// entries = append(entries, rf.log[rf.nextIndex[i]:pendingCommitIndex+1]...)
   	}
   	args := &AppendEntriesArg{
   		Term:         rf.currentTerm,
   		LeaderId:     rf.me,
   		PrevLogIndex: prevLogIndex,
   		PrevLogTerm:  prevLogTerm,
   		Entries:      entries,
   		LeaderCommit: rf.commitIndex,
   	}
   	rf.mu.Unlock(rf.me, "reSendAppendEntries")
   	DPrintf("[%d] -> [%d] resend AppendEntries args%+v", rf.me, svrId, args)
   
   	// start to send
   	for retryTimes != 0 {
   		if rf.killed() {
   			break
   		}
   		DPrintf("[%d] -> [%d] resend AppendEntries start, cnt:%d\n", rf.me, svrId, retryHBWhenTimeout-retryTimes+1)
   		rf.mu.Lock(rf.me, "reSendAppendEntries")
   		if rf.currentTerm != args.Term || args.PrevLogIndex != rf.nextIndex[svrId]-1 || rf.role != LEADER {
   			DPrintf("resend exit: rf.currentTerm(%d) != args.Term(%d) || args.PrevLogIndex(%d) != rf.nextIndex[svrId]-1(%d) || rf.role(%d) != LEADER\n", rf.currentTerm, args.Term, args.PrevLogIndex, rf.nextIndex[svrId]-1, rf.role)
   			rf.mu.Unlock(rf.me, "reSendAppendEntries")
   			break
   		}
   		rf.mu.Unlock(rf.me, "reSendAppendEntries")
   
   		var reply AppendEntriesReply
   		ok := rf.sendAppendEntries(svrId, args, &reply)
   		if ok {
   			DPrintf("[%d] -> [%d] resend AppendEntries success, cnt:%d\n", rf.me, svrId, retryHBWhenTimeout-retryTimes+1)
   			// NOTE:update nextIndex and matchIndex even though the reply is stale data
   			rf.mu.Lock(rf.me, "reSendAppendEntries")
   			rf.updateNextMatchIndex(AppendEntriesReplyInCh{
   				AppendEntriesReply: reply,
   				svrId:              svrId,
   			}, pendingCommitIndex)
   			rf.mu.Unlock(rf.me, "reSendAppendEntries")
   			if !reply.Success {
   				DPrintf("[%d] -> [%d] resend AppendEntries agian, cnt:%d\n", rf.me, svrId, retryHBWhenTimeout-retryTimes+1)
   				rf.reSendAppendEntries(svrId, pendingCommitIndex)
   			}
   			return
   		} else {
   			DPrintf("[%d] -> [%d] resend AppendEntries timeout again, cnt:%d\n", rf.me, svrId, retryHBWhenTimeout-retryTimes+1)
   		}
   		retryTimes--
   	}
   	DPrintf("[%d] -> [%d] resend AppendEntries failed, maybe target svr had been shut down\n", rf.me, svrId)
   }
   ```

   如上reSendAppendEntries是重发AppendEntries函数。**另外针对超时的情况，也做了重发，只是重发的次数做了限制。**

3. 更改了关闭log channel的方式，原来采用atomic计数器，每个执行发起AppendEntries的线程在完成后，对计数器减一，当减到0时，关闭channel。现在改用waitGroup方式。 维护更简单。
4. 之前apply操作采用了 lastAppliedIndex 和commitIndex来表示当前需要apply的log片段范围。但考虑到可能存在多个需要apply的片段，所以做了一个apply log片段边界队列，队列里面的每一个元素代表一组需要apply的log的边界范围。然后使用applier线程执行apply操作。
5. *之前还做了一个超时立即重发优化，用于应对网络不稳定的情况，但是老是出bug，暂时放弃了。逻辑在判定 `sendAppendEntries` 调用是否为True，如果为False，认为发生了timeout，那么立即重发。*

## 4. 总结

没想到raft又写了一篇文章，分布式的难度确实比单机的开发难上不少。
