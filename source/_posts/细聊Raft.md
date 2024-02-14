---
title: 细聊Raft
categories: 分布式
abbrlink:
date: 2023-11-02 13:58:12
---


# 1. 前言

一年多前做过mit 6824, 学习了raft，不过当时更多是从代码层面记录博文，最近正好在系统学习分布式的知识，本文从理论角度全面细谈Raft。

<!--more-->

# 2. Raft

Raft是Paxso的后生，都属于分布式共识算法。由于Paxso在工程落地上缺乏指导，实现复杂，Diego Ongaro等人提出了Raft（Re{eliable|eplicated|edundant} And Fault-Tolerrant的简称），Raft具有更好的可理解性，且在论文中提供了关键点的指引，很好地减少了分布式共识在学术与工业落地之间的gap。

Raft基于的系统模型如下：

1. 服务器可故障宕机，但不存在拜占庭故障。
2. 网络可分区、延迟，乱序重复。(即公平损失链路模型)

## 1. 一些名词说明

Raft采用和Multi-Paxos一样的设计，都是基于Leader的共识算法。系统内的角色包括：

1. Leader: 负责处理来自客户端的请求和日志复制，同一时刻只允许 **一个正常的** leader。 （同一时刻可能有多个，但只有一个有效）
2. Candidate：候选者，处于Follower和Leader之间的暂态
3. Follower：被动处理请求，不会主动发起RPC

和Paxos的提案编号类似，Raft采用 **Term（任期）**一词代码逻辑时间用于解决时序问题，该值单调递增且需要持久化。系统的所有节点都持有自己的Term。

## 2. Leader选举

Raft是基于Leader的共识算法，所以系统上电的第一步即为选举leader，启动时每个节点都是Follower，每个Follower中设有一个定时器，如果在一个定时器时间内，Follower没有收到任何心跳或日志复制RPC，Follower转为Candidate，Candidate首先将自身任期（Term）+1，然后向系统内其他节点发起广播请求(RequestVote RPC), 如果收到半数以上的成功回信，则晋升为Leader，Leader负责广播AppendEntries RPC(日志复制)和定期的心跳来维持自身Leader的角色。系统角色状态切换如下图：

![raft状态转换图](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgraft%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2%E5%9B%BE.svg)

思考：**如果系统内同时出现多个Candidate，如何保证最多只有一个成功晋升为Leader？**

这里要求有两点：

1. 当一个节点收到RequestVote RPC后，**必须保证在一个 Term中最多向一个Candidate投票**。
2. Candidate只有收到 **超过半数以上** 的投票时，才能晋升为Leader。

如上两点，保证系统一个Term里最多有一个Leader。既然最多有一个Leader，那可能出现一个Term内无Leader（比如5节点集群，S1 S2投给S1, S3 S4投给S3, S5不投票，那没有任何节点获取超过半数的票数），根据数学归纳法，可能出现所有Term内也无Leader问题，**即 Liveness 问题**。和Paxos一样，可以通**过随机化选举超时时间**来解决，这样减少了同一时刻出现多个Candidate的几率。 

至此，RequestVote RPC的伪代码可如下（并不完美，后续优化）：

```go
const (
  Follower = iota
  Candidate
  Leader
)

type Raft struct {
	mu sync.Mutex

	state       int // server state, follower, candidate, leader
	currentTerm int
	votedFor    int // vote for which server in currentTerm
	heartBeat   time.Time
}

type RequestVoteArgs struct {
	Term         int
	candidatedId int
}

type RequestVoteReply struct {
	Term         int // returned server's term
	votedGranted bool
}


func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	reply.votedGranted = false
	if args.Term < reply.Term {
		return
	}

	if args.Term > rf.currentTerm {
    rf.currentTerm = args.Term
    rf.state = Follower
    rf.votedFor = -1
	}

	if rf.votedFor == -1 || rf.votedFor == args.candidatedId {
	  rf.votedFor = args.candidatedId 
	  reply.votedGranted = true
    rf.resetTimer()
	}
  // Q: 如果args.Term > rf.currentTerm && rf.votedFor != args.candidatedId，不用resetTimer么?
  // A: 可以在reply的rpc中做处理，recheck一把rf.state是不是还是candidate
	return
}

```

## 3. Raft日志

### 3.1 日志结构 - Term & Index

Raft同paxos，都是通过复制日志状态机来实现分布式共识。Raft的日志包含如下三个：

- 索引 index, 表示日志在整个日志流中的位置
- Term，表示日志被领导者生成的任期号
- Command: 实际的状态机命令

用代码表示为：

```go
type LogEntry struct {
	Index   int
	Term    int
	Command interface{}
}

// 更新raft结构体
type Raft struct {
	mu sync.Mutex

	state       int // server state, follower, candidate, leader
	currentTerm int
	votedFor    int // vote for which server in currentTerm
	heartBeat   time.Time

	log []LogEntry
}
```

在log entry中，Command根据业务场景自定义，对Raft算法本身来说，Command并不重要。**重要的是log的 索引和任期号，这两个值唯一决定了一条Raft log**。 另外需要注意的是，**日志必须持久化**。

> 对Raft来说，每个Server对应的Term更改（包括candidate自增，根据其他server的term更新自身term等）是否需要持久化？
>
> 个人认为是不需要的，只用读取最新的日志，恢复出Term即可。

一次Raft Log复制的流程大体如下：

>  前置要求，Raft集群内Leader选取成功。

1. 客户端向Leader发起请求
2. Leader生成log并持久化到自身的log流中
3. Leader并行地向其余所有（至少半数以上）节点发起AppendEntries RPC，等待rsp
4. Leader收到超过半数以上的成功日志复制响应，即可立即标记刚才的日志**已Commit**（同paxos中的chosen），而不用等到所有rpc响应，记录该日志index为commitedIndex，并在接下来的AppendEntries（下一次日志复制或心跳中），将该commitedIndex发送给其余节点，其余节点收到该请求后，可标记小于等于commitedIndex的日志为commited。
5. 如果有节点因为宕机或者网络问题未收到AppendEntries RPC，没有成功复制日志，Leader会在后台不断重试。

### 3.2 日志一致性校验 + AppendEntries

思考一个问题：follower、candidate一旦收到含有log的AppendEntries就能无脑接收吗？

显然是不行的，比如对一份log，经过网络发送后，若遇网络故障产生重传，follower和candidate收到两次这样的log，不能都接收。

所谓一致性校验，即Raft在AppendEntries RPC中包含了新日志条目之前的一个日志条目索引(prevLogIndex)和任期(prevLogTerm)；跟随者收到请求后，检验自己最后一条日志的索引任期号是否与请求消息中的prevLogIndex和prevLogTerm相等，如果相等则接收，否则则拒接。

![image-20240212105343023](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212105343023.png)



结合raft log仅由索引位置和Term决定唯一性，Raft能够保证：

- 如果两个节点在某个索引位置处的Term相同，则它们从日志开头到该位置的所有日志均相同。
- 如果给定的日志已经提交，则前面的日志也提交。

**这也是Raft和Paxos的不同点之一，raft总是保证日志被连续提交，Paxos则不保证。**

> 这很容易通过数学归纳法证明

下面是AppendEntries的实现:

```go
type AppendEntriesArgs struct {
	Term         int
	LeaderId     int
	LeaderCommit int

	PrevLogIndex int
	PrevLogTerm  int
	Entries      []LogEntry
}

func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	reply.Success = false
	if args.Term < rf.currentTerm {
		return
	}
	if args.Term > rf.currentTerm {
		reply.Term = args.Term
		rf.currentTerm = args.Term
	}

	// reset role
	rf.state = Follower
	// 日志一致性校验
	lastLogIndex := len(rf.log) - 1
	if rf.log[lastLogIndex].Index < args.PrevLogIndex /* log 接不上 */ || rf.log[args.PrevLogIndex].Term != args.PrevLogTerm /* 能接上，但term不相等, 肯定有垃圾log */ {
		return
	}
	reply.Success = true

	// 排除重复log
	index := args.PrevLogIndex
	for i, entry := range args.Entries {
		index++
		if index < len(rf.log)-1 {
			if rf.log[index].Term == entry.Term {
				continue
			}
			// 剔除本身后续的垃圾log
			rf.log = rf.log[:index]
		}
		// 添加后续有效log
		rf.log = append(rf.log, args.Entries[i:]...)
		break
	}

	// 更新commit index
	if rf.commitIndex < args.LeaderCommit {
		lastLogIndex = rf.log[len(rf.log)-1].Index
		if args.LeaderCommit < lastLogIndex {
			rf.commitIndex = args.LeaderCommit
		} else {
			rf.commitIndex = lastLogIndex
		}
	}

	// reset role
	rf.state = Follower
	return
}
```

已上为处理AppendEntries的具体实现。

> 一个潜在问题： 观察到一致性校验处立即返回，如何优化该流程，才能让该server立即补全log？
>
> TODO: 补全

## 4. leader完备性、领导者选举优化 - 选择具有更多日志信息的节点

前文介绍了Raft中的日志，我们知道日志由Term和Index唯一决定。回头再看看领导者优化，**Raft保证日志是被连续提交的，也就是说如果某个Term的日志被提交，在更大Term的未来领导者中，无论领导者切换了多少次，该日志也一定存在，进一步地可推导出，每次领导者选举，选择出来的leader一定含有所有的已提交的log** ，这也被称为**leader完备性**。这意味着，**领导者不能覆盖已经提交的日志**。那么在领导者选举中，**应当选择一个具有更多日志信息，日志更“长”的server作为领导者**。 这便是领导者选举优化。

举个例子：

![image-20240212115100434](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212115100434.png)

假设leader为第三台日志，某个时刻，在index 5处，Term为2的日志实际上已经replicate到两台server上，但此时第三台节点故障不可用。需要重新选择leader，此时我们更倾向选择谁作为leader？如果是第二台节点被选作了leader，那么index 5处可能产生新的日志（Term为3），覆盖了原本的Term 2，这不符合Raft保证一个日志被提交，后续的日志中也一定出现的原则。所以此时应当选择第一台服务器当作leader。那该如何去做？

> 这里我们假设日志被复制到多数节点上就认为已“提交”，但实际情况并不是。可见第5节的进一步说明。

具体流程为：

1. 候选者C在RequestVote信息中包含自己日志中的最后一条日志的Index和Term，记为lastIndex和lastTerm
2. 收到此投票请求的server V将比较自身日志信息和RPC中带来的lastIndex和lastTerm。 如果满足 **(lastTermV > lastTermC) || (lastTermV == lastTermC) && (lastIndexV > lastIndex C)**, 则拒绝为其投票。否则予以投票。

通过如上条件，可以保证投票出来的leader，其日志信息一定是超过半数投票给它的节点。

实例代码如下：

```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	reply.Term = rf.currentTerm
	reply.votedGranted = false
	if args.Term < reply.Term {
		return
	}

	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = Follower
		rf.votedFor = -1
	}

	if rf.votedFor == -1 || rf.votedFor == args.candidatedId {
		// 增强选举
		lastIndex := len(rf.log) - 1
		if rf.log[lastIndex].Term > args.lastLogTerm || (rf.log[lastIndex].Term == args.lastLogTerm && rf.log[lastIndex].Index > args.lastLogIndex) {
			return
		}
		rf.votedFor = args.candidatedId
		reply.votedGranted = true
		// reset timer
		rf.resetTimer()
	}
	// 问题：如果args.Term > rf.currentTerm && rf.votedFor != args.candidatedId，不用resetTimer么?
	return
}
```

再举个例子，说明上述条件的作用：

![image-20240212153201356](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212153201356.png)



任期为2的领导者S1的第4条日志刚刚被复制到服务器S3,并且领导者可以看到第4条日志己复制到超过半数的服务器,那么该日志可以“提交”并且安全地应用到状态机。 此时我们可以知道第4条log一定是会出现在未来的所有leader的log中。假设此时重新发起选举，S5不能成为leader，因为它的最后一条log的任期太小，仅为1； S4也不能成为leader，它的Term足够大，但是log长度太短。 S2 S3是否能成为leader？答案是可以的，S2和S3可以从S2 S3 S4 S5中获取选票，是完全可以的。S1亦然。

## 5. 延迟提交 - Raft日志什么时候可提交？

思考一个问题：leader将日志复制到多数节点后，日志真的可以提交了吗？

考虑如下场景：

![image-20240212191048276](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212191048276.png)

假设Term=2的leader为S1，称为leader后由于网络分区，index=3处的日志仅复制到S2，并且S3 S4 S5中S5成为新的leader，并接收了client的3条日志，Term=3。 之后网络愈合，S1 S2 S3中重新又选择S1作为leader，并生成了index=4，Term=4的日志。称为leader后，S1通过AppendEntries RPC向S3复制日志，日志一致性校验通过，index=3，Term=2的日志成功复制到S3，**此刻，Index=3，Term=2的日志满足复制到多数派， 但其仍不能提交**。因为S1可能在“提交”后立即宕机，S5又再次发起选举，由于其log信息最长（除开S1，其Term最大），S5可称为leader，一旦S5成为leader，S5也可通过AppendEntries复制log，将index=3，Term=3的log复制给其他server。 **这会覆盖S2和S3上的index=3，Term=2的日志**。不符合raft中已经提交的日志，不可能被覆盖的要求。

所以，**仅靠日志复制到多数派就认为已提交是错误的！**

如何解决这个问题，这就要用到本节提到的延迟提交了。所谓延迟提交，即**leader必须要看到超过半数的节点上都还存储着至少一条自己任期内的日志**。

如：

![image-20240212192141234](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212192141234.png)

S1成为leader后，将index=3，Term=2的日志复制给S3后，不能立即提交index=3，Term=2的日志，而是要等index=4，Term=4（即当前任期）也被复制到半数以上节点后，再一起提交。此时如果故障，S5也无法成为leader，因为存在更大的Term，领导者选举处检查不通过。

这里，我们总结下，**Raft中要commit一条log要满足两个规则**：

1. 要提交的日志必须存储在超过半数的节点上
2. 领导者必须看到超过半数的节点上还存储着至少一条自己任期内的日志

> 这里再贴下Raft Pager原文中的描述
>
> <img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212193007108.png" alt="image-20240212193007108" style="zoom:50%;" />

但是问题到这里就结束了吗？并没有，回到这张图：

![image-20240212191048276](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212191048276.png)

思考：如果S1成为Term=4的leader后，一直没有client发起请求，那index=3，Term=2的日志岂不是一直无法提交？

为了解决旧Term log一直不提交的问题，**Raft论文引入了一种no-op的空日志**。no-op空日志只有index和Term，command为空。**当candiate成功选举为leader时，不论是否有client请求，都会立即向本地写入一条no-op日志**，并立即将该日志复制给其他节点。no-op空日志属于领导者任期的日志，多数派达成后立即提交，此时no-op之前的老Term日志也被间接提交了。

## 6. 清理不一致的日志

通过前文的描述，我们或多或少都感受到了分布式系统中会出现日志不一致的情况，要么日志少了（比如该节点宕机后重启），要么日志多了（网络分区，多个leader造成），如：

![image-20240212194512286](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212194512286.png)

在Leader上，为了给每个节点发送正确的日志信息，会维护一个nextIndex[] 数组，nextIndex[i]表示对第i个server下一次AppendEntries时，log entries从整条log流中的第i个index开始复制。nextIndex[i]的初始值为 1 + leader最后一条日志的索引。

现在，来看如下例子：

![image-20240212194843687](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240212194843687.png)

对于跟随者1，其少了部分log。 初始状态是，leader中nextIndex[1]=11, 将prevLogIndex=10, prevLogTerm=6通过AppendEntires发送给跟随者1，由于跟随者1无index=10的log，拒绝接受该日志。于是leader回退一格，重新发送。这样不断重试，直到nextindex[1]=5时，一致性校验通过。此时补齐index 5 ~ index 10之间的log。

对于跟随者2，其多了部分log。一样的道理，leader中nextIndex[2]=11, 将prevLogIndex=10, prevLogTerm=6通过AppendEntires发送给跟随者2， 由于index=10处的Term=3，一致性校验不通过。leader回退不断重试，直到nextIndex[2]=4时，一致性校验通过，跟随者2将 index 4 ~ index 11的log全部丢弃，并补齐 index 5 ~ index 10之间的log。

> 思考：leader重试时是否可加速回退速度，毕竟一格格退的网络开销是巨大的。

## 7. 剔除旧leader

分布式系统中，总会因为各种原因，出现各种问题。多leader是一个常见的场景，Raft中不能容忍多主的存在，总要想办法剔除过期的leader。Raft的做法如下：

1. 每个RPC请求，都包含发送方的Term
2. 如果接收方发现自身的Term小于发送方的Term，自愿转为Follower，并更新任期。如果接收方的Term大于发送方的Term，则拒绝该RPC，并将自己的Term回复给发送方，发送方如果知道自己的Term过旧，自愿退到Follower，并更新Term。

## 8. 客户端协议

客户端的协议包含如下：

1. client起初不知道谁是leader，可随机选择一位server发起请求，如果该server为leader，则正常处理，如果该server不为leader，则将谁是leader告诉client，client自己重试
2. 同一个命令不能被执行多次，和Paxos一样，client的每个请求都要求带有唯一id，leader将请求的id记录下来，下一次收到client的重复请求时，直接丢弃。

## 9. 线性一致性读

介绍至此，我们重点在如何安全的实现写操作，并得到共识。但仍然未能满足线性一致性读，比如系统中可能存在两个leader，一个旧leader，一个新leader，如果client联系的是旧leader，很可能读到的是一个过期的数据。

要做到线性一致性的读（即总是读到最新的数据），可做如下尝试：

**方法一：**将读请求也封装成一个log，让log走完复制，commit，apply，然后回复。这样一定满足线性一致性读，但是性能不行，因为涉及到log的持久化，复制日志带来的开销等。

**方法二：**leader中增加一个readIndex比来优化一致性读，流程为：

1. 读请求发起后， 若leader当前Term内还无log被提交，则等待，等待当前Term至少有一条log（可以通过no-op来强制提交）被提交。一次任期内log提交后，当前leader就能知道所有commit log信息，保证当前读请求读到的数据是最新的。
2. 将readIndex=CommitIndex
3. leader决定处理读请求后，向follower发起一次心跳，当半数以上心跳得到确认，则leader知道整个集群内，自己仍是最新的leader。
4. leader等待状态机apply日志中的命令，至少等待到日志index apply到当前readIndex(commitIndex)
5. leader处理读请求，返回查询结果

这种方式避免了多于的磁盘io和网络携带复制log开销，但网络开销增加了（多一轮心跳）。

现在来看，leader承担的职责是比较重的，不但要完成日志复制，还要处理业务的读请求，为了分担leader负载，可以让follower处理读请求。为了保证follower读也是满足一致性读，follower在收到读请求后，需要向leader询问最新的readIndex, leader执行`1~3`步，确定可用的readIndex给follower，由follower做剩余的`4~5`步。

**方法三：**采用租约机制。方法二是比较推荐的做法，但是方法二带来了额外的网络开销。一种优化方式是，leader使用正常的定时心跳来维护一个租约，心跳开始时记录时间为start，当心跳被多数派所确认，leader将租约扩长至 `start + electionTimeout/clockDriftBound`时间。在租约内，leader可直接回复读请求。这样的优化的原理在于，一次被确认的心跳后，Raft选举处一个新的leader，至少要经过electionTimeout时间。clockDriftBound为时间漂移界限，即在这一段时间内，服务器的时间不会突然漂移超过这个界限（比如因为时间同步，时间发生了跳变），如果时间跳变超过这个界限，系统可能不满足线性一致性读。

> 问题：为什么是除以clockDriftBound?



## 10. 集群配置变更

### 1. 联合一致

> 这部分书中说得异常跳跃，原Paper也不好理解。建议观看：
>
> 1. [B站视频](https://www.bilibili.com/video/BV11u411y7q9/?spm_id_from=333.337.search-card.all.click&vd_source=d23f1827dad64db7a3c4e984a81c19fc)
> 2. [Raft作者视频](https://www.youtube.com/watch?v=YbZ3zDzDnrw&t=361s)

**本节逻辑可能有误，希望指正。**

分布式系统中，节点总会因各种原因故障宕机，或因为性能、容量等问题需要扩容。这意味着集群要支持动态扩容或缩容的功能。一种可行的办法是将集群完全停掉，手动更改配置然后重启，但这意味着集群在整个过程中需要停服，手动操作也会带来潜在的人为误操作风险。所以我们的目标是“不停服支持配置变更”。

Raft论文中提出了一种“联合一致性(joint consensus)"的方法来实现此需求。我们先来看问题：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213100742628.png" alt="image-20240213100742628" style="zoom:50%;" />

设原集群共3个节点，S1 S2 S3, 现在要扩容至5个节点。 每个server感知应用新配置的时刻不同的，在某个时刻，S1 S2还未知道配置已经变更，S3 S4 S5已经感知配置变更。 此时整个集群会出现两种 majority， 一种是 S1 S2 S3组成，另一种由S1 S2 S3 S4 S5组成，故而集群可能在同一个Term选出两个leader，如S1 S2 S3选出S1作为leader， S3 S4 S5选出S5作为leader。 出现经典的脑裂现象。

Raft论文中提出了一种两阶段协议，将整个配置变更分为两个阶段：

- 阶段一：集群先切换到一个过渡的配置，称为联合一致性。 具体做法是leader向集群其余节点复制一个C(old,new)的配置log，所有节点收到该log后立即应用配置（注意，这里不用等到log被commit），之后有C(old,new)配置的节点上的操作（选举，成为leader后复制log，读、心跳等操作）均需要获得两种配置的 majority 才能算是被确认，具体流程见下文。 
- 阶段二：从过渡配置切换到新配置。具体做法是leader发起Cnew日志，并复制到其余节点。其余节点一旦收到log就立即应用配置（注意，这里不用等到log被commit），之后有Cnew配置的节点上的操作只用获得Cnew配置的majority就能算被确认，具体流程见下文。

再次强调，**Raft将配置变更也作为一个log写入到log流中，但是这种log是特殊的，不必等到被提交就可立即应用。**

#### 1. 脑裂分析

现在详细展开，看Raft是如何用联合一致性解决脑裂问题的。

![image-20240213102641926](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213102641926.png)

上图为整个配置切换流程。横轴代表时间，Cold+new,Cnew字母首次出现时，代表他们被发起的时间，虚线代表已创建但是未提交的配置项，实线代表最新的已提交的配置项。将上图拆分为5个时间段，对每个时间段依次分析是否会出现脑裂现象。下面所有的例子，无特殊说明都为集群从3节点扩容到5节点。

![image-20240213103021672](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213103021672.png)

时间段1，仅有Cold配置，S1 S2 S3中S1作为leader，此时不可能出现脑裂，因为都没开始变更。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213103629368.png" alt="image-20240213103629368" style="zoom:50%;" />

时间段2，发起Cold+new，但还未提交Cold+new日志。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213110438303.png" alt="image-20240213110438303" style="zoom:50%;" />

如上仅有leader S1，S4、S5上有Cold+new。分析此处时候否会有脑裂。假设S1将Cold+new复制给S4和S5后未提交立即宕机，现在重新选主。若S1发起选主，必须获取S1 S2 S3中的majority，同时必须获取S1 S2 S3 S4 S5中的majority，符合条件，所以S1可成为主。S2或S3发起选主，只用获取S1 S2 S3中的majority，也可成为主。 S4或S5发起选主，最多可获取到Cnew中的 S1 S4 S5确认，对于Cold中的S1 S2 S3则无法获取确认，因为S2 S3不承认S4和S5, 也就不可能为它们投票。 所以综上，S1、S2、S3可能成为主，但是他们是互斥的，也就不会出现脑裂问题了。 这说明，**在Cold+new已提出但未提交时，新选主仍可在Cold配置中(S2 S3)，所以这个阶段Cold可独立决策**  

> 补充说明，为集群新增节点时，节点并不是立即进入到集群，这些集群应该首先补齐日志，然后才进入到集群。
>
> 如果未补齐日志就进入集群，那么leader要通过AppendEntries补齐这些新节点的日志，同时leader接收到的请求需要符合更大的majority, 由于补齐需要时间，影响了正常请求的提交。

时刻3， Cold+new被提交，但是还未提起Cnew日志。状态为S1已提交Cold+new日志后再宕机。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213110529313.png" alt="image-20240213110529313" style="zoom:50%;" />

一旦提交，下次选主出来的leader，一定含有Cold+new日志。所以这个阶段需要联合一致性，即任何操作的确认，需要在老配置和新配置获得majority。S1 S2 S4 S5选主，可获取老旧配置的大多数确认，可成为leader。S3选主，根据领导者选举优化，应当选择信息更多的leader，所以S3肯定不会成为leader。故此不会有脑裂问题。

同理，读者可分析下图，看是否符合要求。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213110542224.png" alt="image-20240213110542224" style="zoom:50%;" />

S1 S2 S4同构，可获取老旧配置的大多数确认，可成为leader。 S3 S5同构，根据领导者选举优化，S3和S5是不可能成为leader的。

>  补充说明：集群在联合一致性状态下，依然是可以提供服务的，比如:
>
> <img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213110711434.png" alt="image-20240213110711434" style="zoom:50%;" />
>
> client写入Term=2的日志，只不过leader需要将Term=2的日志在两种配置下都获取majority的确认。

时刻4：已提起Cnew，但未提交Cnew。Cold+new被提交后，即可立即提起Cnew日志。

如：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213111744517.png" alt="image-20240213111744517" style="zoom:50%;" />

S1作为leader，发起Cnew配置并复制到S5后立即宕机。分析此刻是否会出现脑裂。 S1 S5同构，他们仅需要获取S1,S2,S3,S4,S5的大多数即可，显然符合要求，可成为leader。S2，S3，S4同构，需要获取Cold配置和Cnew配置的大多数确认，也符合需求。所以S1~S5中的任何节点都可能成为leader，但他们彼此互斥，不会出现脑裂。注意S2，S3，S4成为leader后，也会立即发起Cnew配置，继续Cnew提交流程。在这之后，Cnew可单独决策。

再看看如果是缩减成员，Cnew是否可单独决策：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213122415651.png" alt="image-20240213122415651" style="zoom:50%;" />

上图S1~S5, 现要缩减S4 和 S5， 然后重新选举。S1可获得S1~S3的选票，S2、S3同构，无法获取老配置中的选票，因为S4、S5已经宕机，S1的信息更完备，不会给S2 S3投票。所以仅S1可获取选票，当选leader。S1上可靠Cnew单独决策。

时刻5：已提交Cnew。

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213112120528.png" alt="image-20240213112120528" style="zoom:50%;" />

此刻重新选主。一定选择含有Cnew日志的节点，即S1、S2或S5的节点。之后便全是Cnew的配置。

#### 2. 额外注意的点

1. 新增节点时，这些节点需要补齐log后再进入集群，否则会影响到正常log的提交

2. 缩减节点时，可能当前leader自身就是需要缩减的对象，这个leader需要再Cnew提交后退位。

   旧Leader在发起Cnew后，此时这个Cnew majority计数不包含本身。 **TODO: 未找到合适的例子证明该结论，希望有读者补充。**

3. 在缩减节点场景下，已经下线的节点不会再收到来自leader的心跳，它们会因为选举时间超时，而重新发起选举，增加自身Term，向集群其他节点发起RequestVote RPC，正常leader由于Term比RequestVote args中的Term小而退位，导致集群紊乱，因为下线节点如果没有停止会一直发起RequestVote请求。Paper中给出的解决方案是：

   一个节点如果在最小选举超时时间内收到RequestVote RPC，那么它会拒绝此RPC，不会更新Term，也不会更改角色。所谓最小选举超时时间，即一个节点自收到leader的心跳后算起的一个超时时间，这和前面提到的线性一致性读-租约机制一样。这种机制并不会影响正常的选举，因为只要大家都没收到超时，超过一个选举周期后，新选举可以得到回应。

### 2. 单成员变更

联合一致性变更实在过于复杂，边界条件过多。Raft作者又提出了一种新的变更算法--单成员变更。即每次变更仅允许一个节点的扩或缩。

#### 1. 单成员变更的原理与流程

回到配置变更要解决的核心问题，即脑裂问题。为什么会出现脑裂？是因为**集群在变更过程中出现了多个不重叠的majority**。而单节点变更的原理就是避免出现这种不重叠majority。

如下场景：

![single-server](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgraft-single-server.png)

比如a图中的4节点扩1一个节点。原4节点的majority为3，必定和扩容后5节点的majority 3重叠。b c d图同理。一旦有重叠，那么选主过程必定经过重叠节点，也就不会出现脑裂了。

下面简述单成员变更的流程：

![image-20240214094446463](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240214094446463.png)

假设场景为3节点扩容为4节点。

1. S4加入到集群前，先完成日志同步，这点和联合一致性相同。

> 问题：如何判定日志完成同步？Raft作者提出，可以同步n轮（推荐n=10）日志，之后便认为同步完成。
>
> 问题什么需要同步多轮？因为leader在一边同步日志，一边还会接受client的请求。这和[芝诺悖论](https://baike.baidu.com/item/芝诺悖论/241624?fromModule=lemma_inlink)同理，芝诺悖论的结果是逼近极限，阿基里斯一定可以追上乌龟。 对应到日志同步，只要同步一定轮数，一定能够同步完成。

2. 当前leader(S2)发起配置变更，增加一个节点到集群。

   假设此时宕机，我们来看是否会有脑裂，S1，S2同构可以获取S1、S2的选票，当选leader，他们之一当选后，S3，S4一定当选不了。再看S3，S3可以当选leader（获取S1 S2 S3 S4中的大多数），S3当选后，S1 S2 S4一定无法当选。最后看S4，S4只被S3承认，一定无法当选。所以，不会有脑裂问题。

3. leader复制日志到其他节点，当超过majority后，日志提交，之后整个集群无论是否重新选举leader，都会走新配置。

#### 2. 可用性缺陷

单节点变更看似将联合一致变更简化了，但是也带来了它的问题：

单节点变更无法像联合一致那样同时变更多个节点，如果要替换一台故障机器（这在生产环境非常常见），比如将 a,b,c 变更为 d,b,c。 需要先将d扩容进去，再扩容a，分为两步。看起来没什么大不了，**但是分为两步降低了系统的可用性**。因为原来 a,b,c 的大多数只有2，即时a故障了，只要b，c确认操作即可提交。现在由于扩容了d，集群节点数为 a,b,c,d. 大多数变成了3，由于a故障，这就要求b,c,d全确认，一旦这个过程b，c，d任意一个节点出现问题，系统就停服了。

进一步的，单节点变更必然会经历偶数节点的状态，降低系统可用性。除了上段所说，还有个场景为，原集群为a,b,c，现在扩容为a,b,c,d。此时发生网络分区， (a,b)为一组， (c,d) 为一组，这两组都不能提交日志，因为majority达不到3。在原来集群内，(a,b)这一组还是可以用的。这降低了可用性。

![image-20240214100524905](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240214100524905.png)

TiDB针对上述问题做了一个优化，**更改配置变更中偶数节点的大多数概念**，如3扩4后，Cnew的提交不用等到3个节点都确认，只用2个节点确认即可。老配置中任意两个节点 (a,b), (a,c), (b,c)都可以算作配置变更过程中四节点的大多数，这是因为 (a,b), (a,c), (b,c)是新老配置的最小交集，所以不会脑裂。

> TODO: 不理解后面这个最小交集是什么意思。希望有读者指正。



#### 3. 配置变更的bug -- 配置变更前需要提交一个no-op日志

让我们来看这样一个例子：

集群最开始的配置为 S1,S2,S3,S4，四个节点。S5是新加入的节点，S1是领导者，S1将新配置D写入到本地，D的配置是 S1,S2,S3,S4,S5。领导者还会将旧配置C和新的配置D复制给S5。

![image-20240213152026458](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213152026458.png)

之后网络分区，S1被隔离，于是S2发起选举，获得了S2，S3和S4的选票，成为了新的领导者。此时再次进行配置变更E，E的配置为S2，S3，S4，S2将配置复制给S3。如下：

![image-20240213152215183](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213152215183.png)

由于E的配置只有三个节点，配置E已经复制到其中2个，满足多数派条件。S2认为可以提交该配置。

> 疑问：走联合一致算法，Cold,new配置也要满足Cold配置下的多数派，也就是S1~S4中的多数派确认才行，这里的E只满足了Cnew配置确认，为何就认为已提交？
>
> 实际上联合一致性算法不存在该问题，仅在单成员变更算法中才可能存在。《深入理解分布式系统》书中竟然没提到这一点。。。

接着网络分区恢复，S1重新发起选举，获取来自S1，S4，S5的选票，称为leader，并开始将D的配置复制给其他节点。

![image-20240213152637978](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213152637978.png)

这就出现了矛盾，原来S2作为领导时，认为index=2的E配置已经提交，现在又被S1的D给覆盖了。

解决方法是，**新的领导者必须先提交一条no-op空日志，才能将配置写入到日志**。 这样S2当上leader时，先写no-op到S2~S4，在写D。网络分区愈合后，由于S4在index=2处有no-op且其Term比S1的index=2处的Term大，根据领导者选举优化，S4不会给S1投票，S1不能成为leader。

## 11. etcd中的Prev-Vote和CheckQuorum优化

### 1. Pre-Vote

思考一个问题，如果集群内有一个节点和其他三个网络不太稳定，只能发出请求，不能接收请求。那么它会不断地因为选举超时而发起RequestVote，由于每次发起RequestVote Term都会递增，导致集群本来正常的leader退位，如此情况不断反复，导致集群无法正常工作。

![image-20240213154256912](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213154256912.png)

关于这点，其实在 配置变更-联合一致-2.额外注意的点的第3点其实提到过。不过本节更为正式的介绍解决办法。在etcd中选举过程中新增了一个Pre-Vote阶段，同时将Candidate角色又进一步分为PreCandidate和Candidate。Pre-Vote相当于在正式选举先询问别的节点，自己是否能进行选举，只有获得多数节点的确认，才进行真正的选举。具体来说，PreCandidate角色执行Pre-Vote，在Pre-Vote中不会增加Term，只有通过了Pre-Vote阶段，才正式进入Candidate状态并进行选举（此时才增加Term）。增加Pre-Candidate后的状态图如下：

![image-20240213154806414](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213154806414.png)

其他节点在收到Pre-Vote请求后，**同意重新选举**的条件是：

- 参数中的任期更大，或者任期相同但是日志长度更长。 （这和领导者选举优化小结中的条件一致）
- 至少一次选举超时时间内没有收到领导者心跳。 （这在配置变更的额外注意点提到）

只有半数以上节点回复了确认，PreCandidate转为Candidate执行投票。

有了如上机制，对于本小节提出的问题，如果有一个网络故障的节点，因为一直收不到心跳而发起Prev-Vote，其他节点收到Pre-Vote后，因为有正常leader的心跳，所以上述条件的第2点是不满足的，所以Pre-Vote请求失败，也就不会进入真正的选举，进而不会影响正常leader。

这就是Pre-Vote机制，**Pre-Vote保证了一旦有领导当选，leader不会被误下台。**

> Pre-Vote的引入，实际上就是将选择流程分为了两阶段。

### 2. CheckQuorum

有了Pre-Vote，一切就完美了吗？

很遗憾，还是不行。例如如下场景：

<img src="https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213155527551.png" alt="image-20240213155527551" style="zoom:50%;" />

上述图是一个5节点组成的Raft集群，故障前，节点4为leader。现出现故障，5节点完全宕机，4节点仅和2能正常通信。节点1,2,3正常。由于1和3收不到4的心跳，于是它们发起Pre-Vote请求，由于节点2能收到4的心跳，所以Pre-Vote不通过，1,3不能发起选举。由于4还是领导，但是它所生产的日志不能复制到1,3，也就不能满足Quorum，所以集群日志永远无法提交，也永远不能选出新leader，集群进入了停滞状态。而这就是Pre-Vote带来的活性问题。

为了解决这个问题，Raft算法增加了一种机制，让leadr主动下台。机制很简单，**如果leader没有收到半数以上的AppendEntries响应（注意是没收到，而不是拒绝），就主动下台**。这样1,2,3就可以重新选主。 这一优化也被称为 **CheckQuorum**。 

## 12. 日志压缩

可以预见的，随着系统一直运行，日志不断生成堆积会越来越多，也意味着节点重新要apply的日志量越大，影响重启恢复速度，对于新加入的节点，要补齐log的开销也越来越大。所以需要一种日志压缩的方式，来减少日志量。Raft中，压缩日志被称为拍快照。不同的系统有不同的日志压缩方式，所以日志压缩的大部分职责都落在状态机本身上。其基本原理如下：

![image-20240213161646382](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213161646382.png)

假设到index=5处执行日志压缩，index=5处，状态机的状态为 x=0, y=9。 将这个状态（和一些元信息）保存下来，index=1~5处的日志就可以丢弃了。

在日志压缩实现上，不论如何实现，都有几个核心点：

1. 每个节点独立压缩，而不用集中在leader上，这样leader不必将压缩后的日志分发给各节点，减少leader压力。除非是要给离线已久或新加入的节点补齐log。
2. 执行压缩后，除了快照本身要保存外，快照所包含的最后一条的log index和term也要保存，用于后续的日志一致性检验。

为了给滞后太多的节点发送快照，Raft增加了一个新RPC: InstallSnapShot RPC参数,其定义如下：

```go
type InstallSnapshotArgs struct {
	Term             int
	LeaderId         int // 用于重定向client请求
	LastIncludeIndex int
	LastIncludeTerm  int
	Offset           int    // 这部分快照的字节偏移量
	Data             []byte // 从偏移量开始的快照数据
	Done             bool   // 是否是最后一部分快照数据
}

type InstallSnapshotReply struct {
	Term int // 跟随者的term
}
```

## 13. 性能优化

### 1. 并行处理、批处理和流水线

并行处理：

在前面的实现中，领导者将日志写入到磁盘后，再将日志复制到跟随者，等待跟随者持久化后，再确认给leader，两次磁盘IO，有明显的时延。可优化如下：

![image-20240213164838838](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213164838838.png)

leader不必等待log落盘持久化，可以并行的发起复制log，然后提交。

批处理：

批处理的核心就是攒，攒成一批IO，一起下发，这在各种存储系统中都可见，比如LevelDB，RocksDB。因为落盘的开销太大，为了保证持久化成功，落盘后我们会显示调用fsync。如果每一个io都调用一次，性能会成倍的下降，所以在实际中，leader会将收到的多个日志一起落盘然后复制。

流水线：

流水线也是老生常谈的优化技术了。在Raft中，leader会为所有folloer维护一个nextIndex变量，表示下一次向某个follower发起复制日志的起始index。 由于网络故障和乱序是小概率事件，所以leader再向一个follower发送了一批日志后，不必等待follower回应，可立即更新nextIndex，然后立即重新发起后续日志（是不是像CPU里面的分支预测？）。由于我们有日志一致性校验，所以即使出错，重新调整nextIndex即可。

> 具体的工程实践是怎样的，暂不明确。 可以预见的是，这种优化对参数应景是敏感的，如果立即重发过快，负载过高，性能下降。重新过慢，复制过慢，吞吐量也就低了。

### 2. 目击者节点

和Cheap Paxos思想一致。 为了降低服务器成本，Raft使用了一类成为目击者的特殊节点，目击者没有状态机，正常情况下也不存储日志。仅当一个服务器故障时介入，开始存储日志，并参与选举，直到服务器恢复正常工作并替换目击者。 目击者节点的开销很低，可以降低成本。

### 3. Multi-Raft

对于集群节点数量很大的场景，单leader承担的职责很重，不仅要处理client读写请求（虽然读可由Follower分担，但是readindex还是要由leader算出），还要维持大量的心跳和日志复制。CockroachDB剔除一种优化方式，在系统中维持多个Raft组，每组数据独立（可以理解成一个shard或多个shard），彼此之间各自用raft同步。

## 14. Raft的经典问题

**1.如果日志图，那些日志可以被安全提交？**

![image-20240213171426118](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/oss_imgimage-20240213171426118.png)

答：仅<1,1>和<2,1>可以提交。

根据领导者完备性原则，一个log被提交，一定出现在未来的所有leader中。由于S2中出现了Term=3的日志，而没有Term=2的日志，所有所有Term=2的日志都没有提交，Term=3本身也没有复制到多点所以Term=3的日志也没有提交。现在就仅剩<1,1>和<2,1>。 <1,1>是一定被提交的，因为所有人都有该日志，不可能被覆盖。<2,1>是否有可能被覆盖？也不可能，如果要覆盖，必须由S4覆盖，但S4却无法成为leader，根据领导者优化，选举的leader是日志信息最充足的，S4不符合，所以<2,1>也被提交。

**2.每个跟随者都在磁盘上持久化存储了3个信息，当前任期(currentTerm)、最近的投票(voteFor)，以及所有接受的日志记录(log[])。**

问题1: 假设跟随者崩溃，重启时丢弃了投票信息。如果重新加入到集群，这样是否安全？

不安全，Raft要求一个Term内，每个Follower最多给一个节点投票，如果丢失，意味着可能在同一个Term为多个leader投票，最终可能导致脑裂。

例如：对于3台服务器:

1. S1获得S1和S2的投票并且成为任期2的领导者

2. S2重启丢失了它在任期2中投过的票（votedFor）

3. S3获得S2和S3的选票,并且成为任期2的第二任领导者

4. 现在S1和S3都可以在任期2的同—位置的日志上提交不同的值

问题2：假设跟随者的日志被阶段了，日志丢失了最后一些记录，该跟随者重新加入集群是否安全？

不安全。这将允许已提交的日志不被存储在多数派上，允许同意索引提交其他不同值。

例如：对于3台服务器：

1. S1称为任期2的leader，并且在S1和S2上写入index=1，term=2，设置commitIndex=1，然后返回已提交的值给client。
2. S2重启，丢弃了日志。
3. S3重新成为leader（获取S2和S3的选票，一致性检验通过），重新在index=1，term=2上写上另一个值。然后返回提交的值给client。

# 3. Paxos vs Raft

介绍完Paxos（Multi-Paxos)和Raft，现在来分析下它们的异同。

相同点：

1. 均存在选举算法，选择出一个leader，由leader接收client的请求，并写入到log，虽然将log复制到跟随者
2. 日志复制到多数派后，该日志可提交，随后所有成员通过apply log得到一致的状态机
3. 两者都满足状态机安全性和领导者完备性。状态机安全性是指，如果一个节点上的状态机应用了某个索引上的日志，那么其他节点永远不会在同一个索引应用一个不同的日志。领导者完备性是指，如果一个命令在任期t和索引i处被提交，那么任何任期大于t的领导者在索引i处均会存在一样的命令。

不同点：

1. Raft是按照顺序提交日志，Paxos允许不按顺序提交，Paxos允许日志存在空间，并需要额外的协议来补齐日志。
2. Raft具有更高效的领导者选举算法，其选择的leader总是具有更多日志信息的节点。反观Paxos根据server id的大小来选择leader，可能选择了一个滞后了很久的leader，然后需要花费较多时间来补齐log，这期间将会停服。

# 4. 总结

本文详细分析了Raft共识协议。Raft是Paxos的后生，其设计目标是一种更容易理解的共识算法。本文首先介绍了Raft中的三类角色，进而讲解了Raft中的Leader选举算法；随后介绍了Raft中的log，log由index和term唯一决定，根据log我们介绍了AppendEntries中的一致性检验和领导完备性原则，优化了选举算法，使得选举出来的leader总是日志信息量更多的节点。 接着，详细介绍Raft中的一个核心问题与解决方案，即什么条件下leader才能提交日志，这用到了延迟提交条件。随后，我们还介绍了客户端协议，线性一致性读的实现方式。集群配置变更所采用的联合一致性算法，由于联合一致性算法过于复杂，还介绍了单成员变更算法，以及单成员变更算法带来的额外问题，并介绍了在etcd中更为完善的Raft实现所采取的两种优化。 为了解决日志不断变大而带来的问题，介绍了Raft的日志压缩。 最后，我们简单对比了Paxso和Raft算法。

# 5. 参考

- [《深入理解分布式系统》](https://book.douban.com/subject/35794814/)
- [Raft作者视频](https://www.youtube.com/watch?v=YbZ3zDzDnrw&t=361s)
- [B站视频](https://www.bilibili.com/video/BV11u411y7q9/?spm_id_from=333.337.search-card.all.click&vd_source=d23f1827dad64db7a3c4e984a81c19fc)
- [Raft 成员变更的相关问题](https://www.inlighting.org/archives/raft-membership-change)
- [Raft在线动画模拟器](https://raft.github.io/raftscope/index.html)
- Raft Paper: In Search of an Understandable Consensus Algorithm



