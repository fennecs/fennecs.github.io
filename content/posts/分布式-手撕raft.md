---
title: '[分布式]手撕raft'
tags:
  - raft
categories:
  - 分布式
slug: 1823228532
date: 2019-10-03 16:27:57
---
# 前言
Raft作为一个简单的一致性算法，实现一下还是挺好玩的。代码基于**6.824** [lab-raft](http://nil.csail.mit.edu/6.824/2018/labs/lab-raft.html)，**6.824**是麻省理工的分布式课程的一个编号，里面有4个lab，第二个就是raft协议的实现，第三个是基于raft协议的kv存储设计，有待实现（oh我居然在做麻省理工的课程设计）。该lab要求使用go实现算法，并提供了一个[具有故障模拟功能的RPC](http://oserror.com/distributed/golang-rpc-with-failure-simulation/)，即通过模拟网络，在单台机器我们就可以运行raft算法。

> 做实验前，你应该熟读raft论文，这里是[中文版](https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md)

# 实现
参照raft论文和lab提示，整体利用channel作为事件驱动、mutex保证线程安全，写出一个raft算法骨架还是比较容易的。不过在跑test的时候，小小的细节不对就会导致`test failed`。
![](/images/20191004005135.png)
**raft-lab**提供了17个test，检验了各种情况下的一致性，模拟了各种奇葩网络变化（网络变成这样还是跑路吧），要求4分钟内pass。

## 数据结构
参照论文，定义几个数据结构
```go
const (
	Follower = iota
	Candidate
	Leader

	HeartbeatInterval = 100 * time.Millisecond
)

type ApplyMsg struct {
	CommandValid bool
	Command      interface{}
	CommandIndex int
}

type LogEntry struct {
	Term    int
	Command interface{}
}

type AppendEntriesArgs struct {
	Term         int
	LeaderId     int
	PrevLogIndex int
	PrevLogTerm  int
	Entries      []LogEntry
	LeaderCommit int
}

type AppendEntriesReply struct {
	Term      int
	Success   bool
	NextIndex int
}

type Raft struct {
	currentTerm     int
	mu              sync.Mutex          // Lock to protect shared access to this peer's state
	peers           []*labrpc.ClientEnd // RPC end points of all peers
	persister       *Persister          // Object to hold this peer's persisted state
	me              int                 // this peer's index into peers[]
	state           int                 // 0:Follower 1:Candidate 2:Leader
	votedFor        int                 // 这个实验用index来代替节点
	voteCount       int
	commitIndex     int
	lastApplied     int
	currentLeaderId int
	log             []LogEntry
	nextIndex       []int
	matchIndex      []int

	applyCh     chan ApplyMsg
	heartbeatCh chan bool
	leaderCh    chan bool
	commitCh    chan bool
}

type RequestVoteArgs struct {
	// Your data here (2A, 2B).
	Term         int
	CandidateId  int // 这个实验用index来代替节点
	LastLogIndex int
	LastLogTerm  int
}

type RequestVoteReply struct {
	// Your data here (2A).
	VoteGranted bool // 是否支持
	Term        int
}
```
小声bb，`AppendEntriesReply`论文是没有返回`nextIndex`的，而是由leader自己去**减一重试**，这其实是比较慢的，在设置了网络故障**unreliable**的test中，单纯的**减一重试**会导致raft集群在一定时间内不能达到一致。让follower过滤掉同一个term的index，并返回应该尝试的`nextIndex`，虽然会导致一次复制的日志变多，不过提高了集群达到一致的速度。

## 一些封装
```go
// 获取锁/释放锁的封装，可以在利用`runtime.Caller`打印获取锁的调用点，虽然性能损失比较大。
func (rf *Raft) Lock() {
	rf.mu.Lock()
}

func (rf *Raft) Unlock() {
	rf.mu.Unlock()
}

// 字如其名
func (rf *Raft) getLastLogTerm() int {
	return rf.log[len(rf.log)-1].Term
}

func (rf *Raft) getLastLogIndex() int {
	return len(rf.log) - 1
}
```

## raft实例初始化
```go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {
	rf := &Raft{}
	rf.peers = peers
	rf.persister = persister
	rf.me = me

	// Your initialization code here (2A, 2B, 2C).
	rf.state = Follower
	rf.currentTerm = 0
	rf.votedFor = -1
	rf.currentLeaderId = -1
	// 初始化空白日志
	rf.log = append(rf.log, LogEntry{Term: 0})
	rf.applyCh = applyCh

	rf.heartbeatCh = make(chan bool, 100)
	rf.leaderCh = make(chan bool, 100)
	rf.commitCh = make(chan bool, 100)

	// initialize from state persisted before a crash
	rf.readPersist(persister.ReadRaftState())

	// 初始化随机数资源库
	rand.Seed(time.Now().UnixNano())

	go func() {
		for {
			switch rf.state {
			case Follower:
				select {
				case <-rf.heartbeatCh:
				// 这是lab要求心跳
				case <-time.After(time.Duration(rand.Int63()%333+550) * time.Millisecond):
					rf.state = Candidate
				}
			case Leader:
				rf.broadcastAppendEntries()
				time.Sleep(HeartbeatInterval)
			case Candidate:
				go rf.broadcastVote()
				select {
				case <-time.After(time.Duration(rand.Int63()%300+500) * time.Millisecond): //随机投票超时是必须的，为了防止票被瓜分完。
				case <-rf.heartbeatCh:
					rf.state = Follower
				case <-rf.leaderCh:
				}
			}

		}
	}()

	go func() {
		for {
			<-rf.commitCh
			rf.applyMsg(applyCh)
		}
	}()
	return rf
}
```
这个方法返回一个raft实例，读取持久化数据，起了两个goroutine。

* goroutine1是raft三种状态的转化，这里的超时时间不宜设的太短（太短指论文里的时间），在lab文档里有指出为了配合test，选举超时时间应该**larger than the paper's 150 to 300 milliseconds**

* goroutine2应用已提交日志。

在初始化channel的时候应该设置缓冲大于1。多余的事件并不会导致系统不一致，但是若由于channel缓冲不够而导致阻塞，就会使raft节点死锁。

## votedFor清空时机
一次rpc，无论是发起端还是接收端，只要收到更大的term，就要调整自己的状态，发生下面变化：
```go
rf.votedFor = -1
rf.state = Follower
rf.currentTerm = remoteTerm
```
可以看到`state`会变成`Follower`。

假设一种情景，ABC三个节点下，A为leader，此时C发生分区，那么C一定会不断循环进行超时选举，C的term会一直增大，当C网络恢复重新加入集群后会继续发投票请求rpc。由于C的投票请求rpc中的`term`较大，集群就会调整`currentTerm`以及`state`，已有leader会废掉。而问题是，C的请求投票是无意义的，却使集群进行了一次选举。针对这个问题有个**preVote**方案，就是在投票前调研一下自己是否有投票必要，如果没必要，就不发起投票。这篇文章暂无涉及**preVote**。

## 投票发起与接收
### broadcastVote() 发起投票
```go
func (rf *Raft) broadcastVote() {
	rf.Lock()
	rf.currentTerm++
	rf.voteCount = 1
	rf.votedFor = rf.me
	vote := &RequestVoteArgs{
		Term:         rf.currentTerm,
		CandidateId:  rf.me,
		LastLogIndex: rf.getLastLogIndex(),
		LastLogTerm:  rf.getLastLogTerm(),
	}

	rf.persist()
	rf.Unlock()

	for i := 0; i < len(rf.peers); i++ {
		if rf.state != Candidate { // 发送
			break
		}
		if vote.CandidateId == i { // 自己的票已经给自己了
			continue
		}
		go func(server int) {
			var reply RequestVoteReply
			ok := rf.sendRequestVote(server, vote, &reply)
			if !ok {
				return
			}

			rf.Lock()
			defer rf.Unlock()

			// 一般来说，reply.Term > rf.currentTerm 的情况下 reply.VoteGranted 不会为true
			if reply.Term > rf.currentTerm {
				rf.votedFor = -1
				rf.state = Follower
				rf.currentTerm = reply.Term
				rf.persist()
			}

			if reply.VoteGranted {
				rf.voteCount++
				if rf.state == Candidate && rf.voteCount > len(rf.peers)/2 {
					rf.becomeLeader()
				}
			}
		}(i)
	}
}

func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) bool {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	return ok
}
```
在接收到投票reply后，查看票根是否过半，如果过半转化为leader。
```go
// 没有加锁，外部调用已经加锁了
func (rf *Raft) becomeLeader() {
	rf.state = Leader
	rf.nextIndex = make([]int, len(rf.peers))
	rf.matchIndex = make([]int, len(rf.peers))
	// 初始化为0
	for i := 0; i < len(rf.peers); i++ {
		rf.nextIndex[i] = rf.getLastLogIndex() + 1
	}
	rf.leaderCh <- true // 结束选举阻塞
}
```
### RequestVote 接收投票
```go
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	rf.Lock()
	defer rf.Unlock()
	defer rf.persist()

	// Your code here (2A, 2B).
	reply.VoteGranted = false
	reply.Term = rf.currentTerm

	// 过期的投票请求
	if rf.currentTerm > args.Term {
		return
	}
	// 如果发起方的term比接收方大
	// adjust current term
	if rf.currentTerm < args.Term {
		rf.currentTerm = args.Term
		rf.state = Follower
		rf.votedFor = -1
	}

	upToDate := false
	if args.LastLogTerm > rf.getLastLogTerm() {
		upToDate = true
	}
	if args.LastLogTerm == rf.getLastLogTerm() && args.LastLogIndex >= rf.getLastLogIndex() {
		upToDate = true
	}
	if (rf.votedFor == -1 || rf.votedFor == args.CandidateId) && // 保证有票
		upToDate {
		reply.VoteGranted = true
		rf.state = Follower
		rf.votedFor = args.CandidateId
		rf.heartbeatCh <- true
		return
	}
}
```
1，进行投票后要发送心跳``rf.heartbeatCh <- true``，不然节点会由`Follower`超时，从而使集群选举循环下去。
2，判断日志是否较新要满足其中一个条件：一，term较大，二，term一样，但日志index比较大
## 日志复制与接收
### broadcastAppendEntries 广播日志/心跳
日志复制lab文档要求一秒不能超过10次。
```go
func (rf *Raft) broadcastAppendEntries() {
	rf.Lock()
	defer rf.Unlock()

	N := rf.commitIndex
	for i := rf.commitIndex + 1; i <= rf.getLastLogIndex(); i++ {
		// 1 是leader本身
		num := 1
		for j := range rf.peers {
			// 只能提交本term的，一旦提交了本term的，旧term也算提交了
			if rf.me != j && rf.matchIndex[j] >= i && rf.log[i].Term == rf.currentTerm {
				num++
			}
		}
		if num > len(rf.peers)/2 {
			N = i
		}
	}
	if N != rf.commitIndex {
		rf.commitIndex = N
		rf.commitCh <- true
	}

	for i := 0; i < len(rf.peers); i++ {
		if rf.state != Leader {
			break
		}
		if i == rf.me { // 不用给自己心跳
			continue
		}
		var args AppendEntriesArgs
		args.Term = rf.currentTerm
		args.LeaderCommit = rf.commitIndex
		args.LeaderId = rf.me
		args.PrevLogIndex = rf.nextIndex[i] - 1
		args.PrevLogTerm = rf.log[args.PrevLogIndex].Term
		args.Entries = make([]LogEntry, len(rf.log[args.PrevLogIndex+1:]))
		// 复制
		copy(args.Entries, rf.log[args.PrevLogIndex+1:])

		go func(i int, args AppendEntriesArgs) {
			var reply AppendEntriesReply
			ok := rf.sendAppendEntries(i, &args, &reply)
			if !ok {
				return
			}
			rf.handleAppendEntriesReply(&args, &reply, i)
		}(i, args)
	}
}

func (rf *Raft) sendAppendEntries(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	return ok
}
```
每次发送日志前，leader从`matchIndex[]`里统计出应该commit的index，如果index前进，发送commit事件。统计时要判断`rf.log[i].Term == rf.currentTerm`，也就是说只能提交自己term的log，一旦提交了自己term的log，之前term未被提交的log也算提交了。这个在论文有提到。

下面是复制日志的响应代码，也很直白。
```go
// reply 为 false， 如果不是任期问题，就是日志不匹配
func (rf *Raft) handleAppendEntriesReply(args *AppendEntriesArgs, reply *AppendEntriesReply, i int) {
	rf.Lock()
    defer rf.Unlock()

	if rf.state != Leader { // 获取锁后校验自己的状态
		return
	}
	if args.Term != rf.currentTerm {
		return
	}

	if reply.Term > rf.currentTerm {
		rf.votedFor = -1
		rf.currentTerm = reply.Term
		rf.state = Follower
		rf.persist()
		return
    }

	if reply.Success {
        // len(args.Entries)  == 0 就是心跳了，不用处理
		if len(args.Entries) > 0 {
			rf.matchIndex[i] = args.PrevLogIndex + len(args.Entries)
			rf.nextIndex[i] = rf.matchIndex[i] + 1
		}
	} else {
		rf.nextIndex[i] = reply.NextIndex // 直接采用follower的建议
	}
}
```

### AppendEntries 接收日志
```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.Lock()
	defer rf.Unlock()
	defer rf.persist()

	reply.Success = false

    // 告诉老term的节点该更新啦
	if rf.currentTerm > args.Term {
		reply.Term = rf.currentTerm
		return
	}

    // 心跳
	rf.heartbeatCh <- true

	// adjust current term
	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.state = Follower
		rf.votedFor = -1
	}

	reply.Term = rf.currentTerm

    // 这坨是在日志不匹配的情况下，对leader的NextIndex建议
	if rf.getLastLogIndex() < args.PrevLogIndex {
		reply.NextIndex = rf.getLastLogIndex() + 1
		return
	} else if rf.log[args.PrevLogIndex].Term != args.PrevLogTerm {
		term := rf.log[args.PrevLogIndex].Term
		if args.PrevLogTerm != term {
			for i := args.PrevLogIndex - 1; i >= 0; i-- {
				if rf.log[i].Term != term {
					reply.NextIndex = i + 1
					break
				}
			}
			return
		}
	}

	reply.Success = true
	reply.NextIndex = rf.getLastLogIndex() + 1
	// 删除已存在日志
	rf.log = rf.log[:args.PrevLogIndex+1]
	// 附加新日志
	rf.log = append(rf.log, args.Entries...)

	if args.LeaderCommit > rf.commitIndex {
		rf.commitIndex = Min(args.LeaderCommit, rf.getLastLogIndex())
		rf.commitCh <- true
	}
	return
}
```
这理主要是`NextIndex`建议值的计算。

## 将提交的日志应用至状态机
```go
func (rf *Raft) applyMsg(applyCh chan ApplyMsg) {
	rf.Lock()
	defer rf.Unlock()
	for i := rf.lastApplied + 1; i <= rf.commitIndex; i++ {
		msg := ApplyMsg{
			true,
			rf.log[i].Command,
			i,
		}
		applyCh <- msg
		rf.lastApplied = i
	}
}
```
应用过程其实是由test去管理的，我们只要负责把需要应用的日志放入`chan ApplyMsg`。

## 持久化
```go
func (rf *Raft) persist() {
	// Your code here (2C).
	// Example:
	w := new(bytes.Buffer)
	e := gob.NewEncoder(w)
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.log)
	data := w.Bytes()
	rf.persister.SaveRaftState(data)
}

func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { // bootstrap without any state?
		return
	}
	r := bytes.NewBuffer(data)
	d := gob.NewDecoder(r)
	d.Decode(&rf.currentTerm)
	d.Decode(&rf.votedFor)
	d.Decode(&rf.log)
}
```
两个持久化函数，持久化了`currentTerm`当前term,`votedFor`得票者,`log`日志数组，当这三个属性变化时，都执行一次`rf.persist()`就没错啦。

# 后记
表面是在贴代码，实际就是在贴代码。

由于实验是并发过程，一旦`test failed`是不容易按线性的过程来分析的。我的方法是多打日志，以及利用`net/http/pprof`包对程序的goroutine、mutex状态进行分析。

实现完以后我感觉又变强了（并没有）。