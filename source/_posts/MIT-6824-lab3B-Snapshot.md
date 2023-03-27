---
title: MIT-6824-lab3B-Snapshot/Log Compaction
categories: MIT6824
date: 2022-04-29 22:10:31
tags:
---



## 0. 前言

本篇为mit6824 spring 2020 lab3B部分的完成记录。

实验整体要求和 mit6824 2021 lab2D部分相同。不过需要完成service层的snapshot逻辑。

原本以为完成了raft（lab2D）后此处相对简单，不过依然花费了不少时间。

<!--more-->

## 1. 实现

首先依然是贴出整部分的逻辑架构图：

![image-20220308144908922](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20220308144908922.png)

从上面图可以看出，如果raft已经完成了log compaction部分，现在要完成的工作包括如下几个：

1. raft上传snapshot，service需要替换当前状态机至snapshot。
2. service重启，读取snapshot
3. service在合适的时候需要向raft发起Snapshot

要完成上述三个步骤，至少需要思考以下几个问题：

1. raft上传snapshot，service替换状态机至snapshot，如果service又收到了snapshot之前的command的怎么办？这里的场景是，raft层还在通过applych传递旧的Command时，raft又通过InstallSnapshot收到了Snapshot并开始向service上传snapshot，此时service需要有能力避免apply旧的command。 （也许可以在raft层解决这个问题，不过我目前是在service层做的）
2. 什么时候做一次Snapshot?  StartKVServer时会设置maxraftstate, 只要raft state的size超过该阈值，就应该做一次snapshot。（这里可以通过定时检测，也可以放置在处理的关键路径上）

额外坑点：

1. **raft层Start在收到命令后，需要立即发起心跳(AppendEntires), 不能等待心跳计时器，否则测试一定会超时。** 这里卡了我非常久，因为受到了心跳次数1s内不能超过10次的限制思想。但实际上，是必须立即发起心跳的（也可能batch一些log，然后发起心跳）。所以这里在raft层面，需要实现一个机制，用于立即心跳，而不用等待计时器。（个人目前采用了channel唤醒，即新开一个线程，等待一个专用心跳通道，raft在Start收到新命令后，通过通道唤醒该线程执行AppendEntires RPC，这里也踩了很多坑，主要是容易遇到死锁和资源无法释放的问题）

2. Snapshot死锁问题，这里要提醒的是，发起Snapshot时，kv service不要持有锁，否则会有死锁问题

3. service 收到raft的commitLog消息后，消息回传给service RPC handler问题处理。 我曾经在lab3A文章中说到为每个rpc进来兵团通过raft.Start后的log的index建立一个通道（看下面的处理过程图），然后当service收到该log的commit消息后（raft通过applyChan回传上来的)，再通过该通道回传到rpc handler中。虽然整体思想没问题，但是也有非常多的坑：

   ![kvraft中的问题-lab3B-2](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/kvraft中的问题-lab3B-2.svg)

   1. 不同rpc handler可能在相同的log index上建立通道，如何处理这种情况？ 这种情景发生再网络分区又愈合后，raft的log被愈合后网络的leader替换了。
   2. rpc handler不能无限等待在该通道上，因为这条log很可能不会被commit，所以应该设定一定的超时检测机制。这里又分为两种情况：
      1. 如果未超时，则正常处理并返回
      2. 超时了，对log index上的通道该如何处理？我的做法是rpc handler不做任何处理，但是这会带来一个新的问题，那就是rpc handler虽然超时了，但是这个log可能在未来某个时间被raft commit并上传到service，service此时通过log index通道回传给rpc handler时，已经没有rpc handler在等待，所以service会出现死等发送完成的现象。故而service回传给rpc handler的通道处理，仍然需要超时处理。

### 1. snapshot编解码

要实现snapshot，首先需要将service状态机需要持久化的内容序列化以及反序列化,目前，我的service中需要持久化的字段只有 

```
dataBase map[string]string		// key value map
opRecord map[int64]int 			// client对应最大操作opId，用于预防重复执行某个op
```

所以编解码代码很简单：

```go
// NOTE: must be guared by mutex
func (kv *KVServer) encodeServiceState() (snapshot []byte) {
	DPrintf("[%d] encode current service state,  dataBase%+v, opRecord%+v\n", kv.me, kv.dataBase, kv.opRecord)
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(kv.dataBase)
	e.Encode(kv.opRecord)
	snapshot = w.Bytes()
	return
}

// NOTE: must be guarded by mutex
func (kv *KVServer) decodeServiceState(snapshot []byte) {
	DPrintf("[%d] decode service state from snapshot\n", kv.me)
	r := bytes.NewBuffer(snapshot)
	d := labgob.NewDecoder(r)
	dataBase := make(map[string]string)
	opRecord := make(map[int64]int)
	// TODO: do we need to clear dataBase first?
	if d.Decode(&dataBase) != nil ||
		d.Decode(&opRecord) != nil {
		log.Panicf("[%d] decode service state failed\n", kv.me)
	}
	kv.dataBase = dataBase
	kv.opRecord = opRecord
	DPrintf("[%d] decode service state success\n", kv.me)
}
```

### 2. 发起Snapshot

发起Snapshot的时刻决策我放置在每次raft通过applyChan回传了commit Log时。如下：

```go
func (kv *KVServer) receiveFromApplyCommand(applyMsg *raft.ApplyMsg) {
	...
	kv.mu.Lock()
	if kv.lastAppliedIndex < applyMsg.CommandIndex {
		kv.lastAppliedIndex = applyMsg.CommandIndex
	} else {
		DPrintf("[%d] lastAppliedIndex(%d) is larger than applyMsg.CommandIndex(%d)\n", kv.me, kv.lastAppliedIndex, applyMsg.CommandIndex)
	}
    ...
	switch replyOp.OpType {
	case GET:
		kv.doGet(replyOp, &reply)
		DPrintf("[%d] execute Get done\n", kv.me)
	case PUT, APPEND:
		kv.doPutAppend(replyOp, &reply)
		DPrintf("[%d] execute Put/Append done\n", kv.me)
	default:
		log.Panicf("unknown op type%v\n", replyOp.OpType)
	}
	...
	// check raft state size is approaching maxraftstate or not, if so, do a snapshot
	if kv.maxraftstate != -1 && kv.persister.RaftStateSize() >= kv.maxraftstate {
		snapshot := kv.encodeServiceState()
		// fmt.Println("take a snapshot")
		DPrintf("[%d] take a snapshot start", kv.me)
		kv.mu.Unlock()
		kv.rf.Snapshot(applyMsg.CommandIndex, snapshot)
		kv.mu.Lock()
		DPrintf("[%d] take a snapshot end", kv.me)
	}
	kv.mu.Unlock()
}

```

lastAppliedIndex用于预防apply旧的CommandIndex。（感觉这种逻辑放置到raft更合适，毕竟上层对raft的感知越少越好，不过目前先暂时这样做吧）

### 3. 几个坑点的处理

#### 1. RPC handler死等处理

rpc handler不能死等，因为log是可能不被committed的，以Get操作为例：

```go
...
kv.mu.Lock()
...
// build channel
var replyCh chan InternalReply
if _, ok := kv.doneChanMap[idx]; !ok { // Started a command before
    DPrintf("[%d] create channel at index[%d]\n", kv.me, idx)
    kv.doneChanMap[idx] = make(chan InternalReply)
    replyCh = make(chan InternalReply)
    kv.doneChanMap[idx] = replyCh
    kv.mu.Unlock()
} else {
    DPrintf("[%d] more than 1 command at the same idx(%d)\n", kv.me, idx)
    kv.mu.Unlock()
}

// wait on channel
DPrintf("[%d] <-- [%d] finish `Get` rf.Start, args:%+v\n", kv.me, args.ClientId, args)
var tmpReply InternalReply
select {
    case tmpReply = <-replyCh:
        // NOTE: check like Put operation for Get operation is unnecessary
        curTerm, isLeader := kv.rf.GetState()
        if curTerm != term || !isLeader {
        reply.Err = ErrWrongLeader
        DPrintf("[%d] <-- [%d] Server Get fail end, term(%d), curTerm(%d), isLeader(%v), tmpReply.Op(%v), args(%v)\n", kv.me, args.ClientId, term, curTerm, isLeader, tmpReply.OpIdentify, args.OpIdentify)
        return
		}
        reply.Err = tmpReply.Err
        reply.Value = tmpReply.Value
        DPrintf("[%d] <-- [%d] Server Get end, reply(%v)\n", kv.me, args.ClientId, reply)
    case <-time.After(commitLogTimeout * time.Millisecond):
        DPrintf("[%d] time up, give up idx=%d receiving Get%+v reply\n", kv.me, idx, args)
        reply.Err = ErrTimeout
}
```

这里的逻辑分为两部分：

1. 为Start的log index建立通道，但是可能会遇到在同一个log index上建立相同的通道，这种情况说明旧log index处的log已经被替换掉，无需对它做处理
2. 等待在replyCh上，并添加了超时等待机制，一旦超时，直接返回。这里遗留下了replyChan，如果未来raft commit此条log并上传至service，而service将通过log index log回传给rpc handler，而rpc handler已经因为timeout退出，所以service通过log index channel回传时，也需要超时检测。

#### 2. service log index channel处理

当service收到raft log commit消息（即applyMsg后），需要找到指定reply channel回传给rpc，并在最后删除此log index channel。此部分逻辑如下，唯一需要注意的是，对于通过log index channel的回传，我也添加了超时检测。

```go
kv.mu.Lock()
if kv.lastAppliedIndex < applyMsg.CommandIndex {
    kv.lastAppliedIndex = applyMsg.CommandIndex
} else {
    DPrintf("[%d] lastAppliedIndex(%d) is larger than applyMsg.CommandIndex(%d)\n", kv.me, kv.lastAppliedIndex, applyMsg.CommandIndex)
}
switch replyOp.OpType {
case GET:
    kv.doGet(replyOp, &reply)
    DPrintf("[%d] execute Get done\n", kv.me)
case PUT, APPEND:
    kv.doPutAppend(replyOp, &reply)
    DPrintf("[%d] execute Put/Append done\n", kv.me)
default:
    log.Panicf("unknown op type%v\n", replyOp.OpType)
}

if _, ok := kv.doneChanMap[applyMsg.CommandIndex]; !ok {
} else {
    replyCh = kv.doneChanMap[applyMsg.CommandIndex]
    kv.mu.Unlock()
    DPrintf("[%d] try to send command[%v] through doneChanMap[%d] \n", kv.me, replyOp, applyMsg.CommandIndex)
    select {
    case replyCh <- reply:
    DPrintf("[%d] sended command[%v] through doneChanMap[%d] success\n", kv.me, replyOp, applyMsg.CommandIndex)
    case <-time.After(receiverRspTimeout * time.Millisecond):
    DPrintf("[%d] sended command[%v] through doneChanMap[%d] timeout \n", kv.me, replyOp, applyMsg.CommandIndex)
}
kv.mu.Lock()
delete(kv.doneChanMap, applyMsg.CommandIndex)
DPrintf("[%d] delete doneChanMap at index[%d]\n", kv.me, applyMsg.CommandIndex)
}

...
```

### 4. Raft立即心跳优化

raft层，在通过Start收到log后，需要立即心跳。这里我采用了专用线程来处理：

```go
func (rf *Raft) heartBeatFromChan() {
	SpecialPrintf("[%d] heartBeatFromChan start...\n", rf.me)
	// timerId := time.Now().Unix()
	for {
		val := <-rf.hBChan
		if val == hBChanEnd {
			if !rf.killed() {
				log.Panic("pass 1 to end hBChan, but rf instant is not killed, did you forget to set the dead flag?")
			}
			break
		}
		if val != hBChanContinue {
			log.Panic("only val = 0, we can send AppendEntriesRPC")
		}
		rf.mu.Lock(rf.me, "HeaderBeatTimer")
		rf.oneMoreTicker = true
		if rf.role == LEADER {
			DPrintf("[%d] fireAppendEntriesRPC\n", rf.me)
			rf.fireAppendEntries()
		}
		rf.mu.Unlock(rf.me, "HeaderBeatTimer")
	}
	SpecialPrintf("[%d] heartBeatFromChan end...\n", rf.me)
}
```

即检测rf.hBChan，如果rf.hBChan有hBChanContinue消息，则发起新的一轮AppendEntires。

修改后的Start如下：

```go

//
// the service using Raft (e.g. a k/v server) wants to start
// agreement on the next command to be appended to Raft's log. if this
// server isn't the leader, returns false. otherwise start the
// agreement and return immediately. there is no guarantee that this
// command will ever be committed to the Raft log, since the leader
// may fail or lose an election. even if the Raft instance has been killed,
// this function should return gracefully.
//
// the first return value is the index that the command will appear at
// if it's ever committed. the second return value is the current
// term. the third return value is true if this server believes it is
// the leader.
//
// NOTE: this is a thread-safe function
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	...
	DPrintf("[%d] notify hBChan start", rf.me)
	select {
	case rf.hBChan <- hBChanContinue: // TODO: maybe stuck here?
		DPrintf("[%d] notify hBChan end", rf.me)
	case <-time.After(10 * time.Millisecond):
		DPrintf("[%d] notify hBChan timeout", rf.me)
	}
	return index, term, isLeader
}
```

**除此外，该优化也可用于leader commit log加速，即leader在第一次收到大多数的log确认请求后，leader commit该log，但是还需要一轮心跳才能告知其他svr，此条log已经commit，有了该机制，可以减少一次心跳等待。** 根据测试结果来看，这个优化加速效果非常明显，对于部分测试，加速了超过1倍。

## 2. 总结

lab3本应不算很难，但我依然花费了不少时间，主要在于对于一些死锁和资源释放的处理要分析不少时间，可能是我对go的工具链使用还是不够熟练，除了`-race`检测以外，暂时不知到还有什么其他方式，所以遇到问题，只能分析log。
