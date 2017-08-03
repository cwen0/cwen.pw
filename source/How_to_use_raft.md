title: How to use Raft
author: cwen
date:  2017-07-30
update:  2017-07-30
tags:
    - 分布式
    - Raft
    - 一致性
    - etcd

---

记录如何使用 ETCD raft Library... <!--more-->
## 先说 ETCD Raft library

ETCD Raft library 作为目前使用最为广泛的 raft 库，也可以说是目前最为完善稳定的。以一个使用者的姿态来看看该如何将这个 raft library 用在自己的项目中。

这个库实现了 Raft 算法的核心内容，比如 append log 、选主、snapshot、成员变更，但是用户需要自己实现网络传输和网络IO， 用户必须实现自己的传输层，用来不同node 之间进行消息传输，用户也需要自行实现存储层用来Raft日志和状态。

### ETCD Raft library 实现的功能

* Leader election
* Log replication
* Log compaction
* Membership changes
* Leadership transfer extension
* Efficient linearizable read-only queries served by both the leader and followers
* leader checks with quorum and bypasses Raft log before processing read-only queries
* followers asks leader to get a safe read index before processing read-only queries
* More efficient lease-based linearizable read-only queries served by both the leader and followers
* leader bypasses Raft log and processing read-only queries locally
* followers asks leader to get a safe read index before processing read-only queries
* this approach relies on the clock of the all the machines in raft group

#### More

* Optimistic pipelining to reduce log replication latency
* Flow control for log replication
* Batching Raft messages to reduce synchronized network I/O calls
* Batching log entries to reduce disk synchronized I/O
* Writing to leader's disk in parallel
* Internal proposal redirection from followers to leader
* Automatic stepping down when the leader loses quorum

### Storage 接口

library 定义了一个 Storage 接口，但是需要用户自己实现

```
// Storage is an interface that may be implemented by the application
// to retrieve log entries from storage.
//
// If any Storage method returns an error, the raft instance will
// become inoperable and refuse to participate in elections; the
// application is responsible for cleanup and recovery in this case.
type Storage interface {
    // InitialState returns the saved HardState and ConfState information.
    InitialState() (pb.HardState, pb.ConfState, error)
    // Entries returns a slice of log entries in the range [lo,hi).
    // MaxSize limits the total size of the log entries returned, but
    // Entries returns at least one entry if any.
    Entries(lo, hi, maxSize uint64) ([]pb.Entry, error)
    // Term returns the term of entry i, which must be in the range
    // [FirstIndex()-1, LastIndex()]. The term of the entry before
    // FirstIndex is retained for matching purposes even though the
    // rest of that entry may not be available.
    Term(i uint64) (uint64, error)
    // LastIndex returns the index of the last entry in the log.
    LastIndex() (uint64, error)
    // FirstIndex returns the index of the first log entry that is
    // possibly available via Entries (older entries have been incorporated
    // into the latest Snapshot; if storage only contains the dummy entry the
    // first log entry is not available).
    FirstIndex() (uint64, error)
    // Snapshot returns the most recent snapshot.
    // If snapshot is temporarily unavailable, it should return ErrSnapshotTemporarilyUnavailable,
    // so raft state machine could know that Storage needs some time to prepare
    // snapshot and call Snapshot later.
    Snapshot() (pb.Snapshot, error)
}
```
用户实现 Storage 接口用以实现持久化存储，[raftexample](https://github.com/coreos/etcd/tree/master/contrib/raftexample)  使用library 提供 的 MemoryStorage, 配合使用 etcd 的wal 和sanp 包可是实现持久化， 重启的时候从wal和snap中获取日志恢复MemoryStorage。

### Ready struct

```
// Ready encapsulates the entries and messages that are ready to read,
// be saved to stable storage, committed or sent to other peers.
// All fields in Ready are read-only.
type Ready struct {
    // The current volatile state of a Node.
    // SoftState will be nil if there is no update.
    // It is not required to consume or store SoftState.
    *SoftState

    // The current state of a Node to be saved to stable storage BEFORE
    // Messages are sent.
    // HardState will be equal to empty state if there is no update.
    pb.HardState

    // ReadStates can be used for node to serve linearizable read requests locally
    // when its applied index is greater than the index in ReadState.
    // Note that the readState will be returned when raft receives msgReadIndex.
    // The returned is only valid for the request that requested to read.
    ReadStates []ReadState

    // Entries specifies entries to be saved to stable storage BEFORE
    // Messages are sent.
    Entries []pb.Entry

    // Snapshot specifies the snapshot to be saved to stable storage.
    Snapshot pb.Snapshot

    // CommittedEntries specifies entries to be committed to a
    // store/state-machine. These have previously been committed to stable
    // store.
    CommittedEntries []pb.Entry

    // Messages specifies outbound messages to be sent AFTER Entries are
    // committed to stable storage.
    // If it contains a MsgSnap message, the application MUST report back to raft
    // when the snapshot has been received or has failed by calling ReportSnapshot.
    Messages []pb.Message

    // MustSync indicates whether the HardState and Entries must be synchronously
    // written to disk or if an asynchronous write is permissible.
    MustSync bool
}
```

这个Ready结构体封装了一批更新，这些更新包括：

* pb.HardState: 包含当前节点见过的最大的term，以及在这个term给谁投过票，已经当前节点知道的commit index
* Messages: 需要广播给所有peers的消息
* CommittedEntries:已经commit了，还没有apply到状态机的日志
* Snapshot:需要持久化的快照

用户从 node struct 提供一个 ready channel 中不断的 pop 出 Ready 进行处理，library user 使用如下方法获取Ready Channel

```
func (n *node) Ready() <- chan Ready{ return n.readyc}
```

应用需要对 Ready 的处理如下：

1. 将HardState, Entries, Snapshot持久化到storage。
2. 将Messages(上文提到的msgs)非阻塞的广播给其他peers
3. 将CommittedEntries(已经commit还没有apply)应用到状态机。
4. 如果发现CommittedEntries中有成员变更类型的entry，调用node的ApplyConfChange()方法让node知道(这里和raft论文不一样，论文中只要节点收到了成员变更日志就应用)
5. 调用Node.Advance()告诉raft node，这批状态更新处理完了，状态已经演进了，可以给我下一批Ready让我处理。

应用通过raft.StartNode()来启动raft中的一个副本，函数内部通过启动一个goroutine运行来启动服务

```
func (n *node) run(r *raft)
```

应用通过调用

```
func (n *node) Propose(ctx context.Context, data []byte) error
```
来Propose一个请求给raft，被raft开始处理后返回。

增删节点通过调用

```
func (n *node) ProposeConfChange(ctx context.Context, cc pb.ConfChange) error
```
node结构体包含几个重要的channel:

```
// node is the canonical implementation of the Node interface
type node struct {
    propc      chan pb.Message
    recvc      chan pb.Message
    confc      chan pb.ConfChange
    confstatec chan pb.ConfState
    readyc     chan Ready
    advancec   chan struct{}
    tickc      chan struct{}
    done       chan struct{}
    stop       chan struct{}
    status     chan chan Status

    logger Logger
}
```

* propc: propc是一个没有buffer的channel，应用通过Propose接口写入的请求被封装成Message被push到propc中，node的run方法从propc中pop出Message，append自己的raft log中，并且将Message放入mailbox中(raft结构体中的msgs []pb.Message)，这个msgs会被封装在Ready中，被应用从readyc中取出来，然后通过应用自定义的transport发送出去。
* recvc: 应用自定义的transport在收到Message后需要调用
```
func (n *node) Step(ctx context.Context, m pb.Message) error
```
来把Message放入recvc中，经过一些处理后，同样，会把需要发送的Message放入到对应peers的mailbox中。后续通过自定义transport发送出去。

* readyc／advancec: readyc和advancec都是没有buffer的channel，node.run()内部把相关的一些状态更新打包成Ready结构体(其中一种状态就是上面提到的msgs)放入readyc中。应用从readyc中pop出Ready中，对相应的状态进行处理，处理完成后，调用

```
rc.node.Advance()
```

往advancec中push一个空结构体告诉raft，已经对这批Ready包含的状态进行了相应的处理，node.run()内部从advancec中得到通知后，对内部一些状态进行处理，比如把已经持久化到storage中的entries从内存(对应type unstable struct)中删除等。

* tickc:应用定期往tickc中push空结构体，node.run()会调用tick()函数，对于leader来说，tick()会给其他peers发心跳，对于follower来说，会检查是否需要发起选主操作。

* confc/confstatec:应用从Ready中拿出CommittedEntries，检查其如果含有成员变更类型的日志，则需要调用

```
func (n *node) ApplyConfChange(cc pb.ConfChange) *pb.ConfState
```
这个函数会push ConfChange到confc中，confc同样是个无buffer的channel，node.run()内部会从confc中拿出ConfChange，然后进行真正的增减peers操作，之后将最新的成员组push到confstatec中，而ApplyConfChange函数从confstatec pop出最新的成员组返回给应用。

## Usage

raft library 使用raft.StartNode 开启一个 node ，或是使用raft.RestartNode 从一些初始状态启动一个 node。
启动三节点 node

```
  storage := raft.NewMemoryStorage()
  c := &Config{
    ID:              0x01,
    ElectionTick:    10,
    HeartbeatTick:   1,
    Storage:         storage,
    MaxSizePerMsg:   4096,
    MaxInflightMsgs: 256,
  }
  // Set peer list to the other nodes in the cluster.
  // Note that they need to be started separately as well.
  n := raft.StartNode(, []raft.Peer{{ID: 0x01}, {ID: 0x02}, {ID: 0x03}})
```

启动一个单节点集群

```
// Create storage and config as shown above.
// Set peer list to itself, so this node can become the leader of this single-node cluster.
peers := []raft.Peer{{ID: 0x01}}
n := raft.StartNode(c, peers)
```

加入新节点到集群的时候，新集群不能存在任何 peer 。首先，通过在集群内其他节点上调用ProposeConfChange将节点添加到现有集群。然后，启动新的节点，如下所示：

```
// Create storage and config as shown above.
n := raft.StartNode(c, nil)
```

重启节点

```
  storage := raft.NewMemoryStorage()

  // Recover the in-memory storage from persistent snapshot, state and entries.
  storage.ApplySnapshot(snapshot)
  storage.SetHardState(state)
  storage.Append(entries)

  c := &Config{
    ID:              0x01,
    ElectionTick:    10,
    HeartbeatTick:   1,
    Storage:         storage,
    MaxSizePerMsg:   4096,
    MaxInflightMsgs: 256,
  }

  // Restart raft without peer information.
  // Peer information is already included in the storage.
  n := raft.RestartNode(c)
```

在创建了node 后，用户需要做这些：

First ，从Node.Ready（）通道读取并处理其包含的更新。这些步骤可以并行执行，除了步骤2中所述。

1. 将HardState，Entries和Snapshot写入storage（如果它们不为空）。如果写入 entry 的索引是 i， 那么就得丢弃索引大于 i 的所有log。
2. 将所有消息发送到目的的节点。一直发送消息直到最新的HardState已经持久存储到磁盘，并且以Ready batch 方式写入所用的 entry。为了减少I/O延迟，可以应用优化使领导者与其追随者并行写入磁盘。如果任何消息类型为MsgSnap，则在发送后调用Node.ReportSnapshot()（这些消息可能很大）。注意：marshal message 不是线程安全的;确保 在marshalling 的时候没有新的 entry 。实现这一目标的最简单的方法是直接在在 raft loop 内串行执行。
3. 将快照（如果有）和CommittedEntries应用到状态机。如果任何已提交的Entry具有TypeConfChange类型，则调用Node.ApplyConfChange（）将其应用于该节点。此时可以通过在调用ApplyConfChange之前将NodeID字段设置为零（但是ApplyConfChange必须以某种方式调用，并且取消决定必须完全基于状态机而不是外部信息，例如观察到节点的健康状况）。
4. 调用Node.Advance（）来指示下一批更新的准备状态。这可以在步骤1之后的任何时间完成，尽管所有更新必须按照Ready所返回的顺序进行处理。

Second, 所有持久化的日志条目必须通过Storage接口的实现来提供。可以使用提供的MemoryStorage类型（如果在重新启动时重新填充其状态），或者可以提供自定义的磁盘支持实现。

Third，从另一个节点收到消息后，将其传递给Node.Step：

```
func recvRaftRPC(ctx context.Context, m raftpb.Message) {
	n.Step(ctx, m)
}
```

最后，定期调用Node.Tick（）（可能通过time.Ticker）。raft 有两个重要的超时：心跳和选举超时。但是，在 fraft library 内部抽象 "tick" 代表时间。

状态机处理循环，如下所示：

```
 for {
    select {
    case <-s.Ticker:
      n.Tick()
    case rd := <-s.Node.Ready():
      saveToStorage(rd.State, rd.Entries, rd.Snapshot)
      send(rd.Messages)
      if !raft.IsEmptySnap(rd.Snapshot) {
        processSnapshot(rd.Snapshot)
      }
      for _, entry := range rd.CommittedEntries {
        process(entry)
        if entry.Type == raftpb.EntryConfChange {
          var cc raftpb.ConfChange
          cc.Unmarshal(entry.Data)
          s.Node.ApplyConfChange(cc)
        }
      }
      s.Node.Advance()
    case <-s.done:
      return
    }
  }
```

节点获取应用程序数据并同步到状态机，把数据序列化到slice，并调用：

```
n.Propose(ctx，data)
```

如果proposal 被 commit ，数据将出现在类型为raftpb.EntryNormal的提交的条目中。不能保证提出的命令将被提交;该命令超时后会重试。

要添加或删除集群中的节点，请申请 ConfChange struct'cc'并调用：

```
n.ProposeConfChange(ctx，cc)
```
配置更改提交后，将返回类型为raftpb.EntryConfChange已经被提交的 entry 。这必须通过以下方式应用于节点：

```
var cc raftpb.ConfChange
cc.Unmarshal(data)
n.ApplyConfChange(cc)
```

注意：ID表示集群中所有时间的唯一节点。即使旧节点被删除，给定的ID也只能使用一次。这意味着，例如IP地址可能会导致糟糕的节点ID，因为它们可能被重用。节点ID必须不为零。


## 参考

1. [Etcd Raft](https://github.com/coreos/etcd/tree/master/raft)
2. [etcd raft library设计原理和使用](http://www.cnblogs.com/foxmailed/p/7137431.html)
