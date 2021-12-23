+++
title = "etcd raft"
description = ""
tags = [
    "raft",
]
date = "2021-12-23"
categories = [
    "raft",
]
menu = "main"
+++

---
etcd 代码分析可以参考：[etcd 注解版](https://github.com/microyahoo/etcd) ，也欢迎大家交流讨论

## Raft message type

Raft集群中节点之间的通信都是通过传递不同的Message来完成的，[Message类型](https://github.com/etcd-io/etcd/blob/7572a61a39d4eaad596ab8d9364f7df9a84ff4a3/raft/raftpb/raft.pb.go#L75-L95) 有很多种，下面介绍部分：
- MsgHup
> 用于选举，对于`follower`或者`candidate`，其`tick`函数对应`tickElection`。如果`follower`或者`candidate`在选举超时时间都没有收到心跳信息，它将发送MsgHup到Step方法，其角色将变为candidate（对于follower节点）并发起新一轮选举。

- MsgBeat
> MsgApp是一个内部使用的类型，对于leader节点，其`tick`函数对应`tickHeartbeat`，周期性地向follower节点发送`MsgHeartbeat`消息。

- MsgProp
> MsgProp提议附加数据到log entries，而且它会重定向propose到leader节点。因此，`send`方法用hardState覆写了Message的term号。-
> - 当MsgProp传递到leader的`step`方法，leader首先会调用`appendEntry`方法将entries附加到它的log，接着会调用`bcastAppend`方法发送entries到peers。
> - 当MsgProp传递到candidate，则直接丢弃。
> - 当传递到follower，MsgProp存储到follower的mailbox，即msgs，它保存了发送者的ID，随后通过`rafthttp`包转发到leader。

- MsgApp
> 复制log entries。leader调用`bcastAppend`方法，其会调用`bcastAppend`方法发送即将要复制的MsgApp类型的logs。当MsgApp被传递到candidate的`Step`方法，candidate将会revert back to follower，表示存在有效的leader在发送MsgApp消息。candidate和follower会回复`MsgAppResp`消息。

- MsgAppResp
> 用于响应log复制请求。当MsgApp发送到candidate和follower的`Step`方法，它们会调用`handleAppendEntries`方法，其会发送MsgAppResp到raft mailbox。

- MsgVote
> 请求投票选举。当MsgHup传递到follower或candidate的`Step`方法，将会调用`campaign`方法去竞选自己成为leader， 一旦`campaign`方法被调用，节点将会变成candidate，然后发送MsgVote到peers去请求投票。
> - 当消息传递到leader或candidate的`Step`方法，如果消息的term小于leader或candidate的term，则消息会被拒绝掉（MsgVoteResp将设置`Reject`为true），如果leader或candidate接收到term更大的MsgVote，他们将会revert back to follower。
> - 当'MsgVote'被传递给follower时，只有当sender的last term大于MsgVote的term或sender的last term等于MsgVote的term但sender的最后提交索引大于或等于follower的时，它才会投票给sender。

- MsgVoteResp
> 投票选举相应消息。当candidate收到MsgVoteResp，它会统计自己的票数，如果票数超过quorum，它将会变成leader，然后调用`bcastAppend`，如果candidate收到大多数的拒绝投票，则revert back to follower。

- MsgPreVote和MsgPreVoteResp
> 两阶段选举协议，是一个可选配置项。当`Config.PreVote`设置为true时，预选举过程与常规选举过程类似，只是不会增加term，除非它在第一个阶段竞选赢得了大多数票。加入这个选项可以将节点分区导致的影响降至最低。

- MsgSnap
> 请求应用快照消息。当一个节点刚变为leader，或者leader接收到MsgProp消息，然后调用`bcastAppend`方法，这个方法会调用`sendAppend`方法到每个follower节点。在`sendAppend`方法中，如果leader获取term或者entries失败，则leader将会发送MsgSnap类型消息来请求快照。

- MsgSnapStatus
> 告知快照消息应用的结果。当follower节点拒绝MsgSnap消息，表明因为网络问题导致网络层不能正常发送snapshot到follower，从而导致快照请求失败，leader将follower的`progress`设置为`probe`。当MsgSnap没有被拒绝，表明快照被正常接收，leader将follower的`progress`设置为`probe`，同时开始log复制。

- MsgHeartbeat
> leader发送心跳信息。
> - 当MsgHeartbeat消息发送到candidate，且消息的term大于candidate的term，则candidate将revert back to follower，同时会更新自己的提交索引号，然后发送消息到mailbox。
> - 当MsgHeartbeat消息发送到follower的`Step`方法，如果消息的term大于follower的term，则follower会更新`leaderID`。

- MsgHeartbeatResp
> 心跳响应消息。MsgHeartbeatResp传递到leader的`Step`方法，leader会知道哪些节点做了回复响应。只有当leader最后提交索引号大于follower的`Match`索引号时leader调用`sendAppend`方法。

- MsgUnreachable
> 告知消息或请求不能被deliver，当MsgUnreachable传递到leader的`Step`方法，leader发现发送MsgUnreachable消息的follower节点不可达，这通常意味着MsgApp丢失了。如果此时follower的progress状态为`replicate`，则leader会将其重新设置回`probe`状态。

- MsgCheckQuorum
> 如果开启`CheckQuorum`，则选举超时后会发送`MsgCheckQuorum`消息，leader节点判断其是否满足quorum，如果不满足则step down to follower节点。

- MsgReadIndex
> 

- MsgReadIndexResp
> 

- MsgTransferLeader
> 对于`MsgTransferLeader`消息，follower节点会将其转发到leader节点，leader节点在收到`MsgTransferLeader`消息后会首先记录lead被转移者，然后判断转移目标的日志是否跟上了。
> - 如果跟上了会向被转移者发送 `MsgTimeoutNow` 消息, 被转移者收到消息后会强制发起新一轮选举。
> - 如果没有跟上则先进行日志同步，等leader收到同步日志的`MsgAppResp`消息后会判断其是否已跟上，步骤同上。

- MsgTimeoutNow
>

### Local message
- MsgHup
- MsgBeat
- MsgUnreachable
- MsgSnapStatus
- MsgCheckQuorum

### Response message
- MsgAppResp 
- MsgVoteResp 
- MsgHeartbeatResp 
- MsgUnreachable 
- MsgPreVoteResp

---
## Q & A
1. 发送如下类型的消息时需要设置term，其他类型的消息都不需要设置term。
- pb.MsgVote
- pb.MsgVoteResp
- pb.MsgPreVote
- pb.MsgPreVoteResp

但是对于消息类型既不是MsgProp，又不是MsgReadIndex类型的，会为其加上raft.Term，具体实现参考`raft/raft.go`中的 `send()`方法。

2. raft的`tracker.ProgressTracker`在什么地方赋值的？
> 初始化集群时，如果没有`WAL`，同时为新集群，同时指定了`--initial-cluster`参数，会将其解析为ServerConfig的`InitialPeerURLsMap`参数，然后初始化RaftCluster，并添加members。接着`bootstrapRaftFromCluster`方法会根据cluster的member ids生成peers。
> Peer的信息如下：
```go
type Peer struct {
    ID      uint64
    Context []byte
}

type Member struct {
    ID types.ID `json:"id"`                                                                         
    RaftAttributes
    Attributes
}
```
> peer中的`Context`为`Member`序列化后的信息，其中包括 ID, RaftAttributes, Attributes信息。接着`raft/node.go`会调用`StartNode`方法，里面的Bootstrap方法会根据peers生成相应的`pb.ConfChange` entries，然后调用`applyConfChange`方法，这里会更新`raft.prs`，返回最新的`tracker.Config`和`tracker.ProgressMap`信息。
```go
(dlv) p cfg.Voters
go.etcd.io/etcd/raft/v3/quorum.JointConfig [
	[
		9372538179322589801: {}, 
		10501334649042878790: {}, 
		18249187646912138824: {}, 
	],
	nil,
]
(dlv) p prs
go.etcd.io/etcd/raft/v3/tracker.ProgressMap [
	9372538179322589801: *{
		Match: 0,
		Next: 3,
		State: StateProbe (0),
		PendingSnapshot: 0,
		RecentActive: true,
		ProbeSent: false,
		Inflights: *(*"go.etcd.io/etcd/raft/v3/tracker.Inflights")(0xc00012f470),
		IsLearner: false,}, 
	10501334649042878790: *{
		Match: 0,
		Next: 3,
		State: StateProbe (0),
		PendingSnapshot: 0,
		RecentActive: true,
		ProbeSent: false,
		Inflights: *(*"go.etcd.io/etcd/raft/v3/tracker.Inflights")(0xc00012f620),
		IsLearner: false,}, 
	18249187646912138824: *{
		Match: 0,
		Next: 3,
		State: StateProbe (0),
		PendingSnapshot: 0,
		RecentActive: true,
		ProbeSent: false,
		Inflights: *(*"go.etcd.io/etcd/raft/v3/tracker.Inflights")(0xc00012f560),
		IsLearner: false,}, 
]
```

3. `apply`的时候什么条件触发快照？ `unstable`中的快照是什么时候赋值的？什么条件触发？
> TODO

4. 为什么`raft.msgs`读写时不需要加锁？
> 其实etcd里很多地方都是采用的单线程模式，比如`apply`也是。

5. 对于 `raftpb.Message`中的`MsgApp`类型，其`LogTerm`，`Index`, `Commit`, `Entries`的含义？
> `LogTerm`通常用于append raft logs到follower节点。
> 
> 例如：对于消息`(type=MsgApp,index=100,logTerm=5)`，表示leader从index=101开始 append entries，而index=100对应的Term值为5。
> 对于消息`(type=MsgAppResp,reject=true,index=100,logTerm=5)`，表示follower拒绝了leader的entries（可能是部分），由于follower节点已经包含index=100，term=5的entry。
>
> `Commit` 为 `raftLog` 的`committed` index。
>
> `Index`和`LogTerm`字段是用于日志匹配的日志（即发送的日志的上一条日志）的index与term（用于日志匹配的term字段为LogTerm，消息的Term字段为该节点当前的term，部分消息需要自己指定，部分消息由`send`方法填充）。`Entries`字段保存了需要复制的日志条目。`Commit`字段为leader提交的最后一条日志的索引。

6. 如何提高读性能，同时避免网络分区后重新选举出新leader出现的`stale read`？
> TODO

7. `CheckQuorum`默认自动开启，同时开启`Check Quorum`会自动开启`Leader Lease`

8. `raft.maybeSendAppend`在发送 Message 时，会对 Message 中的entries大小做限制，`maxSizePerMsg`为1MB，因此entries在超过此大小时会如何处理？ `raftLog.maybeAppend`如何更新commit？
> 如果要发送的entries超过大小限制，则会发送多次

9. `pb.MsgSnapStatus`消息以及`ReportSnapshot`的作用？
> TODO

10. `candidate`节点在赢得选举之后会append一条空的日志条目，其作用是什么？
> candidate在当选leader后会在当前term为自己的日志追加一条空日志条目，并广播，以提交之前term的日志，具体可参考`raft/raft.go`中的`handleAppendEntries()`。

11. 日志复制不匹配时的回退优化算法
> TODO

12. 对于`follower`节点，无论是处理`MsgApp`消息还是处理`MsgSnap`消息，返回的消息都是`MsgAppResp`。

13. 在通过`Transport`发送`Messages`时，会忽略`Message.To == 0`的消息。

14. 由于etcd的模块化设计，raft模块和存储网络模块是分开的，因此`send`方法只是将消息放入`mailbox`，而不是立刻将其发出（etcd/raft也没有通信模块）, 其与外界的交互都是通过`Ready`来进行处理的。因此，当`follower`收到`MsgApp`请求时，执行的操作实际上是（不考虑特殊情况）：
> - 将新日志追加到`unstable`中。
> - 将包含`unstable`的`last index`的`MsgAppResp`消息放入信箱，等待发送。

对于`Ready`的处理，角色不同处理的次序也是有区别的：
> - 对于 `follower` 节点，是先将 `entries`， `hardState`， `snapshot` 保存到稳定的存储后再发送 `Messages`。
> - 对于 `leader` 节点，可以在发送 `Messages` 的同时将 `entries`， `hardState`， `snapshot`持久化。

15. raft中的异常处理，例如 添加成员时leader正在进行`leadTransfer`，如果收到MsgProp的消息，这时会返回 `ErrProposalDropped` 如何处理？
> TODO

16. `applyEntryNormal`时有V2和V3请求，分别对应`pb.Request`和`pb.InternalRaftRequest`，其log对应：
```
167 {"level":"debug","ts":"2021-12-08T10:49:48.188+0800","caller":"etcdserver/server.go:1835","msg":"Applying entry","index":8,"term":2,"type":"EntryNormal"}
168 {"level":"debug","ts":"2021-12-08T10:49:48.189+0800","caller":"etcdserver/server.go:1885","msg":"apply entry normal","consistent-index":7,"entry-index":8,"should-applyV3":true}
169 {"level":"debug","ts":"2021-12-08T10:49:48.189+0800","caller":"etcdserver/server.go:1908","msg":"applyEntryNormal","V2request":"ID:16732981032079369986 Method:\"PUT\" Path:\"/0/members/45d559f8148de837/attributes\" Val:\"{\\\"name\\\":\\\"infra4\\\",\\\"clientURLs\\\":[\\\"http://127.0.0.1:42379\\\"]}\" "}
```
```
206 {"level":"debug","ts":"2021-12-08T10:49:48.206+0800","caller":"etcdserver/server.go:1835","msg":"Applying entry","index":12,"term":2,"type":"EntryNormal"}
207 {"level":"debug","ts":"2021-12-08T10:49:48.206+0800","caller":"etcdserver/server.go:1885","msg":"apply entry normal","consistent-index":11,"entry-index":12,"should-applyV3":true}
208 {"level":"debug","ts":"2021-12-08T10:49:48.206+0800","caller":"etcdserver/server.go:1912","msg":"applyEntryNormal","raftReq":"header:<ID:13926956989250840580 > cluster_version_set:<ver:\"3.6.0\" > "}

261 {"level":"debug","ts":"2021-12-08T10:52:35.450+0800","caller":"etcdserver/server.go:1835","msg":"Applying entry","index":14,"term":2,"type":"EntryNormal"}
262 {"level":"debug","ts":"2021-12-08T10:52:35.450+0800","caller":"etcdserver/server.go:1885","msg":"apply entry normal","consistent-index":13,"entry-index":14,"should-applyV3":true}
263 {"level":"debug","ts":"2021-12-08T10:52:35.450+0800","caller":"etcdserver/server.go:1912","msg":"applyEntryNormal","raftReq":"header:<ID:3632572666012018439 > put:<key:\"\\000\\000\\000\\000\\000\\000\\000\\000\" value:\"\\026T\\316d\\230x\\326x\" > "}
```

17. 新加入的节点或者落后很多的节点，leader 会尝试发送快照数据给follower节点，`maybeSendAppend`方法在处理时会生成快照消息，如下所示：
```go
snapshot, _ := r.raftLog.snapshot()
pb.Message{
    To: to,
    Type: pb.MsgSnap,
    Snapshot: snapshot, // 看起来有点多余
}
```
应用逻辑层通过`Ready`获取`messages`之后会将快照消息单独处理（将其发送到`msgSnapC`），`applyAll` 在收到快照消息后会调用 `createMergedSnapshotMessage` 生成合并的`snap.Message`消息后将其发送到peer端。而 `createMergedSnapshotMessage` 方法会根据当前 `etcd progress` 的 `appliedt`和`appliedi`重新生成新的 `Metadata` 和 `Data` (v2 store序列化后的数据)，所以上面raft层在生成 `MsgSnap` 消息时的 `Snapshot` 是多余的，虽然不影响。

18. `serializable read request` 和 `linearizable read request` ?
> Linearizability is one of the strongest single-object consistency models, and implies that every operation appears to take place atomically, in some order, consistent with the real-time ordering of those operations: e.g., if operation A completes before operation B begins, then B should logically take effect after A.

> etcd 中默认是`linearizable read`，如果需要客户端`serializable read`，可以通过`WithSerializable()`进行设置，`Serializable` 请求适用于低延迟。

以`Range`为例，
```go
if !r.Serializable {
	err = s.linearizableReadNotify(ctx)
	trace.Step("agreement among raft nodes before linearized reading")
	if err != nil {
		return nil, err
	}
}
```

`ReadOnlyOption` 包含两种类型：
- `ReadOnlySafe` 通过与`quorum`个节点进行通信来保证只读请求的线性化，默认选项。
- `ReadOnlyLeaseBased` 依赖领导者租约(`leader lease`)来保证只读请求的线性化，它会受`clock drift`的影响。

如果`ReadOnlyOption` 是 `ReadOnlyLeaseBased` 的时候必须开启`CheckQuorum`。

19. [Read index retry](https://github.com/etcd-io/etcd/pull/12780) 中有个注释还没太理解

20. 


---
codes
```
server/etcdserver/server.go 299 NewServer()
server/etcdserver/bootstrap.go 526 newRaftNode()
server/etcdserver/bootstrap.go 268 bootstrapCluster()
server/etcdserver/api/rafthttp/transport.go 175 Send()
raft/node.go 241 RestartNode()


Breakpoint 1 (enabled) at 0xfc67f5 for go.etcd.io/etcd/server/v3/etcdserver.NewServer() ./server/etcdserver/server.go:300 (1)
Breakpoint 2 (enabled) at 0xfad002 for go.etcd.io/etcd/server/v3/etcdserver.bootstrapCluster() ./server/etcdserver/bootstrap.go:269 (1)
```

### ConfState
```
raft/tracker/tracker.go  146 ConfState()
server/storage/schema/confstate.go 在DB中保存或者读取
raft/confchange/restore.go 
raft/raft.go

server/etcdserver/bootstrap.go 285 bootstrapExistingClusterNoWAL()
```

```
raft/raft.go stepLeader() 1112
raft/raft.go maybeSendAppend() 432
raft/raft.go handleAppendEntries() 1498
```

### snapshot 发送与接收
```
server/etcdserver/api/rafthttp/http.go 199 ServeHTTP()
server/etcdserver/snapshot_merge.go
server/etcdserver/server.go 906 applyAll()
server/etcdserver/api/rafthttp/snapshot_sender.go 68 send()
server/etcdserver/raft.go 246 start()
```

## 参考链接
- [etcd 注解版](https://github.com/microyahoo/etcd)
- [raft doc](https://github.com/etcd-io/etcd/blob/main/raft/doc.go)
- [etcd的raft实现之tracker&quorum](https://blog.csdn.net/weixin_42663840/article/details/100056484)
- [etcd raft 处理流程图系列3-wal的存储和运行](https://www.cnblogs.com/charlieroro/p/15134181.html)
- [深入浅出etcd/raft —— 0x05 Raft成员变更](https://mrcroxx.github.io/posts/code-reading/etcdraft-made-simple/5-confchange/)
- [raft: liveness problems during apply-time configuration change](https://github.com/etcd-io/etcd/issues/12359)
- [etcd learner design](https://etcd.io/docs/v3.5/learning/design-learner/)
- [raft: implement fast log rejection](https://github.com/etcd-io/etcd/pull/11964)
- [Raft Consensus Protocol Made Simpler](https://levelup.gitconnected.com/raft-consensus-protocol-made-simpler-922c38675181)
