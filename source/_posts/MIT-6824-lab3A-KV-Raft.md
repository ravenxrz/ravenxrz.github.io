---
title: MIT-6824-lab3A-KvRaft
categories: MIT6824
date: 2022-03-8 22:10:31
tags:
---

## 0. 要求

本次lab为KvRaft PartA, 原文要求[在这里](http://nil.csail.mit.edu/6.824/2020/labs/lab-kvraft.html)

简述要求:

> 实现基于raft的kv存储service，client发起GET/PUT/APPEND请求，server做相应处理，并且基于raft达到fault-tolerate。整个系统要求满足 **线性一致性**

**在做本实验前，请一定一定确保lab2的单元测试通过100次以上**

<!--more-->

## 1. 实现

**本文代码不代表lab3的最终实现，随着lab3的后续实现，本文所出现的代码可能会更改**

让我们不要一下陷入各种细节，思考下整个系统的的架构，下图是官网给出的架构图：

![image-20220308144908922](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/image-20220308144908922.png)

对于lab3A来说，不用关注任何和持久化相关的问题。

现在来思考要实现整个流程，需要解决些什么问题？

1. Clerk发起请求时，向谁发起？
2. 如果Clerk发送到的svr不是leader，Clerk该如何反应？
3. leader服务器收到Clerk的请求后，提交给raft后，就死等该命令被committed吗？如果此时网络分区了该怎么办？
4. leader收到请求，提交给raft后，如果刚才提交的命令丢失了（因为网络分区又愈合，被其他leader将刚才提交的给raft的log覆盖了），该如何处理？
5. 对于Put/Append等写相关命令来说，不能重复执行，如何去除重复执行？
6. raft commit log后，通过applyChannel反馈给kvraft， kvraft根据log entry执行相应command，执行完后应该通知给rpc handler，再由rpc handler通知clerk。 kvraft执行完command，如何执行通知给那个rpc handler？ 因为在同一时刻中，可能存在多个rpc handler的实体（即多个clerk并发请求的情况）
7. leader收到apply响应后，立即向clerk回复，但是如果这个回复包在网络中丢失了，该如何处理？
8. ...

现在让我们先从client(clerk)开始，因为它相对简单。

### 2.1 客户端实现

打开client.go，里面的布局非常简单，仅仅需要我们实现两个接口 Get, PutAppend. client中主要有两个问题需要解决：

1. client开始发起rpc请求时，并不知道谁是leader，所以需要逐渐尝试，这里我采用的是round-robin的方式，从svr 0开始逐渐尝试，如果失败，则重试下一个svr。（其实是有可能所有svr都失败的，这种情况我在代码中并未处理，因为官网的假设是网络最终都会愈合的，所以最终请求一定会成功）。 当找到正确的leader id后，需要记录下该id，避免下一次请求又从头开始寻找。
2. client的一次请求失败后，并不代表该请求并未传达到svr端，有可能只是因为网络超时的原因，导致了client重复发起请求，所以在client的每条请求中都需要有一个对该请求的唯一id标记。 在我个人的实现中，这个标记采用 ClientId + OpId来表示。

有了这两点，实现代码就相对简单了。先看和rpc相关的数据结构：

```go

type OpIdentify struct {
	OpId     int
	ClientId int64
}

// Put or Append
type PutAppendArgs struct {
	Key   string
	Value string
	Op    string // "Put" or "Append"
	// You'll have to add definitions here.
	// Field names must start with capital letters,
	// otherwise RPC will break.
	OpIdentify
}

type PutAppendReply struct {
	Err Err
}

type GetArgs struct {
	Key string
	// You'll have to add definitions here.
	OpIdentify
}

type GetReply struct {
	Err   Err
	Value string
}
```

Clerk的数据结构：

```go
type Clerk struct {
	servers []*labrpc.ClientEnd
	// You will have to modify this struct.
	curLeader int32 // NOTE:atomic operation on this. -1 means no leader found. current Leader id, maybe be modified as leader changes

	opId int32 // NOTE: atomic operation on this. Monotonic increasing
	clientId int64
}
```



#### 2.1.1 Get操作

只需在代码中的实现中包含前面提到的两个问题的解决方案即可。

```go
func (ck *Clerk) Get(key string) string {
	var (
		value   string
		svr     int32
		opId    int32
		sendCnt int = 1	// for debug
	)
	opId = atomic.AddInt32(&ck.opId, 1) // get operation doesn't need this
	args := GetArgs{
		Key: key,
		OpIdentify: OpIdentify{
			ClientId: ck.clientId,
			OpId:     int(opId),
		},
	}
	svr = atomic.LoadInt32(&ck.curLeader)
	if svr == -1 {
		svr = ck.changeLeader()
	}

	for {
		// for debugging
		DPrintf("Client [%v] cnt:%d, send to [%d] Get, args:%+v\n", ck.clientId, sendCnt, svr, args)
		sendCnt++

		// send RPC to server
		var reply GetReply // NOTE: we have to define reply here, if define out of `for loop`, labgorpc will complain
		ok := ck.servers[svr].Call("KVServer.Get", &args, &reply)
		if !ok || (reply.Err != OK && reply.Err != ErrNoKey) { // try another server
			DPrintf("Client [%v] Get failed: Svr[%d]err:%v\n", ck.clientId, svr, reply.Err)
			svr = ck.changeLeader()
			// time.Sleep(RpcRateLimitTimeout * time.Millisecond)
			continue
		}
		if reply.Err == OK {
			value = reply.Value
			DPrintf("Client [%v] found  key[%v] value=[%v]\n", ck.clientId, key, value)
		} else if reply.Err == ErrNoKey {
			DPrintf("Client [%v] no such key[%v] found\n", ck.clientId, key)
		} else {
			log.Panicf("Client [%v] unknown Err:%v", ck.clientId, reply.Err)
		}
		break
	}
	return value
}
```

*实际上这里的OpIdentify是多余的，但是为了和后面的PutAppend操作保持一致，还是加上*

说说changeLeader()如何实现的：

```go
func (ck *Clerk) changeLeader() int32 {
	var expect int32
	for {
		oldValue := atomic.LoadInt32(&ck.curLeader)
		expect = (oldValue + 1) % int32(len(ck.servers))
		if atomic.CompareAndSwapInt32(&ck.curLeader, oldValue, expect) {
			break
		}
	}
	return expect
}
```

为了保证 curLeader 的原子操作，且内部为轮询的方式改变curLeader，这里用到了一个CAS操作。

在代码中，当Clerk收到了rpc的返回响应时，如果rpc返回!ok,或者Err表示为WrongLeader（代码中为不等于OK并且也不等于ErrNoKey），则改变leader重新请求。

#### 2.1.2 PutAppend操作

PutAppend操作和Get操作的处理基本完全一致性，不再赘述。

```go
func (ck *Clerk) PutAppend(key string, value string, op string) {
	// You will have to modify this function.
	var (
		svr     int32
		opId    int32
		sendCnt = 1
		args    PutAppendArgs
	)

	svr = atomic.LoadInt32(&ck.curLeader)
	if svr == -1 {
		svr = ck.changeLeader()
	}
	opId = atomic.AddInt32(&ck.opId, 1)
	args = PutAppendArgs{
		Key:   key,
		Value: value,
		Op:    op,
		OpIdentify: OpIdentify{
			ClientId: ck.clientId,
			OpId:     int(opId),
		},
	}

	DPrintf("Client [%v] PutAppend, args:%+v\n", ck.clientId, args)
	for {
		// for debugging
		DPrintf("Client [%v] cnt:%d, send to [%d] PutAppend, args:%+v\n", ck.clientId, sendCnt, svr, args)
		sendCnt++
		// send RPC to server
		var reply PutAppendReply // NOTE: we have to define reply here, if define out of `for loop`, labgorpc will complain
		ok := ck.servers[svr].Call("KVServer.PutAppend", &args, &reply)
		if !ok || reply.Err == ErrWrongLeader { // try another server
			DPrintf("Client [%v] PutAppend failed: Svr[%d]err:%v\n", ck.clientId, svr, reply.Err)
			svr = ck.changeLeader()
			// time.Sleep(RpcRateLimitTimeout * time.Millisecond)
			continue
		}
		DPrintf("Client [%v] success, reply %v\n", ck.clientId, reply)
		break
	}
}
```

#### 2.1.3 操作码唯一

前文说过，每个Client发起的请求需要有一个唯一标识，我的实现是通过 ClientId + OpId 来保证。OpId比较直观，保证其原子单调递增即可。那么ClientId该如何得到？

这里我采用的方式是记录MakeClerk时的时间戳。如下：

```go
// for debugging
var initTime int64
var once sync.Once

func MakeClerk(servers []*labrpc.ClientEnd) *Clerk {
	ck := new(Clerk)
	ck.servers = servers
	ck.curLeader = -1

	// for debugging
	once.Do(func() {
		initTime = time.Now().UnixNano()
	})

	ck.clientId = time.Now().UnixNano() - initTime
	DPrintf("Client [%v] make clerk, servers:%+v\n", ck.clientId, ck.servers)
	// You'll have to add code here.
	return ck
}
```

虽然这样写在真实的分布式环境下肯定是不对的，但我相信在真实的环境下，每个client肯定也可以通过某种方式得到其唯一的id。（或IP，或某种分布式的uniq id算法）。

### 2.2 服务器端实现

相比客户端来说，服务器端要考虑的问题就比较多了。下图描述了svr的工作流，以及需要处理到的问题：

![kvraft中的问题-2](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/kvraft中的问题-2.svg)

**上图中的”收到commit“步骤由单独线程处理**

首先要明确的是，svr收到消息并通过Start将命令传递给raft后，需要等到raft将log成功replica，再执行刚才的这个命令。那**如果raft一直不反馈这个log被成功replica怎么办？** 如果kv service被卡住，clerk也可能被卡住（假设clerk没有超时处理） 。所以我们需要一个**操作定时器**，如果在规定时间内没有收到raft的反馈，则通告client，这个请求没有执行成功。

第二个问题，**如何避免重复执行命令（操作）**？可以**记录每个Client曾经成功执行过的最大的OpId**。当Client重复请求一个小的Op，直接可以响应该请求已经执行成功了。当raft反馈一个重复的Op时，也可以直接跳过该Op的执行。

第三个问题，**当收到raft的反馈并执行该命令后，该通知那个rpc handler？**， 可以为每个log index建立一个专用的通道，当Op执行完成后，可以通过这个op所在log的index通道来通知rpc handler，让rpc handler响应给client。

第四个问题，紧接第三个问题，**如果多个rpc handler尝试在同一log index上建立通道，该如何处理**？首先要理解的是，什么样的情况下，多个rpc handler会在同一个log index上建立通道？

![kvraft对同一个log_index建立通道可能会遇到的问题](https://cdn.JsDelivr.net/gh/ravenxrz/PicBed/img/kvraft对同一个log_index建立通道可能会遇到的问题.svg)

假设有svr{0,1,2}，0最开始为leader，但提交了命令[1,2,3]后，网络发生了分区，{0}单独作为一个分区，此时client继续发送了[4,5]两个命令，rpc handler会分别在[4,5]两个log index上建立通道，当网络愈合后，{1,2}分区中的leader会同步自己的log, 此时0的log被截断，只剩[1,2,3]。之后新的client发送新的[4,5]命令。那么刚才已经在旧的[4,5]上建立通道，该如何处理这种冲突？

我的做法是，立即通知旧rpc handler，然后新的[4,5] rpc hanlder命令接管旧通道。

ok，理论说了一大堆，现在看代码：

**Op数据结构：**

```go
type Op struct {
	// Your definitions here.
	// Field names must start with capital letters,
	// otherwise RPC will break.
	Key    string
	Value  string
	OpType string // Get/Put/Append

	OpIdentify
 }
```

KVServer数据结构：

```go
type KVServer struct {
	mu      sync.Mutex
	me      int
	rf      *raft.Raft
	applyCh chan raft.ApplyMsg
	dead    int32 // set by Kill()

	maxraftstate int // snapshot if log grows this big

	// Your definitions here.
	dataBase map[string]string
	opRecord map[int64]int // key:clientid, value:max opid .record what operations have been executed by which client before, to avoid duplicate executing

	doneChanMap   map[int]chan InternalReply
    
    curApplyIndex int // for debug, use this to do consistency check
	debugApplyHistory []InternalReply // for debug
}
```

- dataBase中存放的就是kv数据对
- opRecord，记录每个ClientId(int64)到目前位置，执行成功的最大的OpId(int)
- doneChanMap, 前面问题3和问题4所提到的通道。

这里还有一个InternalReply数据结构，用于接收raft消息的专用线程向rpc handler发送消息用：

```go
type InternalReply struct {
	OpIdentify  // not used for completing this lab, but used for debuging
	Err   Err
	Value string // for Get request
}
```

#### 2.2.1 PutAppend操作

PutAppend handler的工作分为两部分：

1. 检查当前提交的请求命令，是否曾经已经执行过，如果已经执行过，那么直接响应
2. 如果没有执行过，需要提交到raft，然后等待执行，并响应。 注意这里的超时处理。

```go
func (kv *KVServer) PutAppend(args *PutAppendArgs, reply *InternalReply) {
	reply.Err = ErrShutDown
	if kv.killed() {
		DPrintf("[%d] return with Shutdown\n", kv.me)
		return
	}

	DPrintf("[%d] <-- [%d] Server PutAppend start...%+v\n", kv.me, args.ClientId, args)
	// avoid duplicate executing
	kv.mu.Lock()
	if _, ok := kv.opRecord[args.ClientId]; ok {
		// check duplicate
		if args.OpId <= kv.opRecord[args.ClientId] {
			DPrintf("[%d] executed PutAppend before...\n", kv.me)
			kv.mu.Unlock()
			reply.Err = OK
			return
		}
	}
	kv.mu.Unlock()

	// init reply
	reply.Err = ErrWrongLeader
	op := Op{
		Key:    args.Key,
		Value:  args.Value,
		OpType: args.Op, // Put Or Append
		OpIdentify: OpIdentify{
			ClientId: args.ClientId,
			OpId:     args.OpId,
		},
	}

	kv.mu.Lock() // must start before Start, altough Start interface is thread-safe
	idx, term, isLeader := kv.rf.Start(op)
	if !isLeader {
		DPrintf("[%d] I am not leader\n", kv.me)
		kv.mu.Unlock()
		return
	}
	// build channel
	var replyCh chan InternalReply
	if _, ok := kv.doneChanMap[idx]; !ok { 
		DPrintf("[%d] create channel at index[%d]\n", kv.me, idx)
		kv.doneChanMap[idx] = make(chan InternalReply)
		replyCh = kv.doneChanMap[idx]
		kv.mu.Unlock()
	} else { // Started a command before
		DPrintf("[%d] PutAppend, more than 1 command at the same idx(%d)\n", kv.me, idx)
		replyCh := kv.doneChanMap[idx]
		kv.mu.Unlock()
		replyCh <- InternalReply{
			Err: ErrWrongLeader, // send fake msg to old routine
		}
	}
	// wait on channel
	timer := time.NewTicker(commitLogTimeout * time.Millisecond)
	select {
	case tmpReply := <-replyCh:
		// check for  invariant condition
		curTerm, isLeader := kv.rf.GetState()
		if curTerm != term || !isLeader {
			DPrintf("[%d] <-- [%d] Server PutAppend fail end, term(%d), curTerm(%d), isLeader(%v), tmpReply.Op(%v), args.Op(%v)\n", kv.me, args.ClientId, term, curTerm, isLeader, tmpReply.OpIdentify, args.OpIdentify)
			return
		}
		reply.Err = tmpReply.Err // OK
		DPrintf("[%d] <-- [%d] Server PutAppend end\n", kv.me, args.ClientId)
		// for debug
		if reply.Err != OK {
			log.Panicf("[%d] <-- [%d] Server PutAppend end, but Errcode is not OK\n", kv.me, args.ClientId)
		}
	case <-timer.C:
		DPrintf("[%d] time up, give up receive PUT/APPEND reply\n", kv.me)
		reply.Err = ErrWrongLeader
		kv.mu.Lock()
		delete(kv.doneChanMap, idx)
		kv.mu.Unlock()
	}
}
```

这里比较迷惑的是以下几行代码：

```go
DPrintf("[%d] PutAppend, more than 1 command at the same idx(%d)\n", kv.me, idx)
replyCh := kv.doneChanMap[idx]
kv.mu.Unlock()
replyCh <- InternalReply{
Err: ErrWrongLeader, // send fake msg to old routine
```

这几行的目的就是为了解决前文提到的问题4。

另外还需要注意一些不变量检查：

```go
// check for  invariant condition
curTerm, isLeader := kv.rf.GetState()
if curTerm != term || !isLeader {
    DPrintf("[%d] <-- [%d] Server PutAppend fail end, term(%d), curTerm(%d), isLeader(%v), tmpReply.Op(%v), args.Op(%v)\n", kv.me, args.ClientId, term, curTerm, isLeader, tmpReply.OpIdentify, args.OpIdentify)
    return
}
```

如果rpc handler收到专用线程的通知时，此时svr的状态已经变化，这意味着之前提交的命令很可能已经被替换掉为其他命名（网络分区后又愈合，新leader覆盖了这个leader的log命令），此时应该让client重试请求。

#### 2.2.2 接收Raft命令的专用线程

我们需要开启一个专用线程，用于接收raft的反馈。接收响应后，根据命令的类型（Get/Put/Append)执行相应的操作，对于Put和Append操作来说，还需要**做去重处理**。执行命令后，需要**更新当前Client执行过的最大OpId**，按照index位置处的通道向rpc hanlder发送命令，在确保rpc接收命令后，还需要删除index位置处的通道。（这部分看起来有点costly，后期考虑是否可以优化）

```go
// receive data from `kv.applyCh`
func (kv *KVServer) receiver() {
	for {
		applyMsg := <-kv.applyCh
		if kv.killed() {
			break
		}
		DPrintf("[%d] get from applyCh now, applyMsg %+v\n", kv.me, applyMsg)
		if applyMsg.CommandValid {
			kv.curApplyIndex++
			if kv.curApplyIndex != applyMsg.CommandIndex {
				log.Panicf("kv.curApplyIndex{%v} is not equal to applyMsg.CommandIndex{%v}\n", kv.curApplyIndex, applyMsg.CommandIndex)
			}
			replyOp, ok := applyMsg.Command.(Op)
			if !ok {
				log.Panicf("replyOp convert to op failed: replyOp(%+v)\n", replyOp)
			}
			DPrintf("[%d] convert to replyOp: %+v\n", kv.me, replyOp)

			var reply InternalReply
			reply.OpIdentify = replyOp.OpIdentify
			var replyCh chan InternalReply
			switch replyOp.OpType {
			case "Get":
				kv.doGet(replyOp, &reply)
				DPrintf("[%d] execute Get done\n", kv.me)
			case "Put", "Append":
				kv.doPutAppend(replyOp, &reply)
				DPrintf("[%d] execute Put/Append done\n", kv.me)
			default:
				log.Panicf("unknown op type%v\n", replyOp.OpType)
			}

			kv.mu.Lock()
			// for debug
			kv.debugApplyHistory = append(kv.debugApplyHistory, reply)
			DPrintf("[%v] apply history(len:%d) %+v\n", kv.me, len(kv.debugApplyHistory), kv.debugApplyHistory)
			// end
			if _, ok := kv.doneChanMap[applyMsg.CommandIndex]; !ok {
			} else {
				replyCh = kv.doneChanMap[applyMsg.CommandIndex]
				kv.mu.Unlock()

				DPrintf("[%d] try to send command[%v] through doneChanMap \n", kv.me, replyOp)
				replyCh <- reply
				DPrintf("[%d] sended command[%v] through doneChanMap \n", kv.me, replyOp)

				kv.mu.Lock()
				delete(kv.doneChanMap, applyMsg.CommandIndex)
				DPrintf("[%d] delete doneChanMap at index[%d]\n", kv.me, applyMsg.CommandIndex)
			}
			kv.mu.Unlock()

		}
	}
	DPrintf("[%d] receiver exits\n", kv.me)
}


func (kv *KVServer) doGet(op Op, reply *InternalReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	val, ok := kv.dataBase[op.Key]
	DPrintf("[%d] get key(%v) value(%v)\n", kv.me, op.Key, val)
	if !ok {
		reply.Err = ErrNoKey
	} else {
		reply.Err = OK
		reply.Value = val
	}
}

func (kv *KVServer) doPutAppend(op Op, reply *InternalReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	// avoid duplicated execution
	if kv.opRecord[op.ClientId] >= op.OpId {
		reply.Err = OK
		return
	}
	// execute op
	if _, ok := kv.dataBase[op.Key]; !ok {
		// take op as put
		kv.dataBase[op.Key] = op.Value
	} else {
		switch op.OpType {
		case "Put":
			// replace
			kv.dataBase[op.Key] = op.Value
		case "Append":
			// append
			kv.dataBase[op.Key] += op.Value // NOTE：concate string  by '+' is not efficient, do more optimization later
		default:
			log.Panicf("unknown OpType")
		}
	}
	// record opId
	if _, ok := kv.opRecord[op.ClientId]; !ok {
		kv.opRecord[op.ClientId] = op.OpId
		DPrintf("[%d]  opRecord, op.ClientId%d, kv.opRecored:%+v\n", kv.me, op.ClientId, kv.opRecord)
	}
	if kv.opRecord[op.ClientId] < op.OpId {
		kv.opRecord[op.ClientId] = op.OpId
	}
	DPrintf("[%d] kv.dataBase:%+v, opRecord:%+v\n", kv.me, kv.dataBase, kv.opRecord)
	// mark reply ok
	reply.Err = OK
}

```

#### 2.2.3 Get操作

Get操作和PutAppend操作基本一致，而且不需要做去重处理，这里不在赘述。

## 2. 总结

对于这个lab来说，最重要的问题在于，如何让专用线程正确的通知到对应的rpc handler线程。不然会遇到很多奇怪的bug。另外lab2一定要保证其正确性，不然做这个lab会非常痛苦。

目前的问题，效率不高，整个lab3A执行完，花费了270s左右。 而官网只花了240s，原以为是lab2实现效率不高，但我的raft比官网执行得更快，所以应该还是lab3的问题。 后期完成整个lab3后，再做调优处理。

**立个flag，写完毕设前，不再碰lab。** 

