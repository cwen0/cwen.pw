title: Leader Transfer In TiKV
author: cwen
date:  2017-10-28
update:  2017-10-28
tags:
    - 分布式
    - Raft
    - TiKV
    - TiDB
    - PD

---
在 TiKV 中，PD 当发现 TiKV 实例上 region 出现 leader 不均匀的时候，会尝试将 leader 从数量比较多的地方 transfer 到其地方，具体调度指令由 PD 发出，TiKV 接收到 PD 的 transfer leader 指令，调用 raft 操作执行真正操作...  <!--more-->

## 先看 PD
PD（Placement Driver）在 TiDB 集群里面主要负责 meta 信息存储，以及管理和调度 TiKV 集群， 所有可以想象 Transfer Leader 的命令显然是有 PD 发送到 TiKV。

PD 给 TiKV 发送 Transfer Leader 命令，可以分为一下两类

* 人为干预调度

PD 提供 pd-ctl 命令行工具，或是通过 api 接口显示的将一个 region 的 leader 调度到某一个 store 上。

```
>> operator add transfer-leader 1 2         // 把 region 1 的 leader 调度到 store 2
>> operator add transfer-region 1 2 3 4     // 把 region 1 调度到 store 2,3,4
```

pd-ctl 实际上也是请求 PD 的api，具体请求过程略，有兴趣的同学可以去研究一下 PD 的源码。

```
// pd/server/coordinator.go
func (c *coordinator) sendScheduleCommand(region *core.RegionInfo, step schedule.OperatorStep) {
        log.Infof("[region %v] send schedule command: %s", region.GetId(), step)
        switch s := step.(type) {
        case schedule.TransferLeader:
                cmd := &pdpb.RegionHeartbeatResponse{
                        TransferLeader: &pdpb.TransferLeader{
                                Peer: region.GetStorePeer(s.ToStore),
                        },
                }
        .....
}

```

从上边代码可以看到，其实 PD 的调度命令是通过 heartbeat 来进行传递的，PD 和 TiKV 之间是通过 grpc 通信，当收到到这个操作指令的时候，就会调用 grpc 的send 方法，将请求发送给 TiKV。

* Leader 分布不均匀或是热点过于集中，PD 自身调度

PD 的主要作用就是负责管理和调度 TiKV， 如果 TiKV 各个节点上出现了 leader 分布补均匀或是热点 leader 过于集中在某一个 TiKV 节点上的时候，这时候 PD 就会作出干预，进行 transfer leader 等操作。PD 是根据每个 store 或是 leader peer 发送过来的心跳包，来作统计并决定执行哪些操作，每次收到 Region Leader 发来的心跳包时，PD 都会检查是否有对这个 Region 待进行的操作，通过心跳包的回复消息，将需要进行的操作返回给 Region Leader，并在后面的心跳包中监测执行结果。注意这里的操作只是给 Region Leader 的建议，并不保证一定能得到执行，具体是否会执行以及什么时候执行，由 Region Leader 自己根据当前自身状态来定。（后面两句话直接 copy 自 https://pingcap.com/blog-tidb-internal-3-zh）。

```
// HandleRegionHeartbeat processes RegionInfo reports from client.
func (c *RaftCluster) HandleRegionHeartbeat(region *core.RegionInfo) error {
        if err := c.cachedCluster.handleRegionHeartbeat(region); err != nil {
                return errors.Trace(err)
        }

        // If the region peer count is 0, then we should not handle this.
        if len(region.GetPeers()) == 0 {
                log.Warnf("invalid region, zero region peer count - %v", region)
                return errors.Errorf("invalid region, zero region peer count - %v", region)
        }

        c.coordinator.dispatch(region)
        return nil
}
```

这个函数处理来自 leader peer 的heartbeat， `handleRegionHeartbeat` 主要负责更相关region 的信息， 我们主要还是关系 如何发送操作指令，想当然就是 `c.coordinator.dispatch(region) ` 这个函数干的事情了。

```
func (c *coordinator) dispatch(region *core.RegionInfo) {
       // Check existed operator.
       if op := c.getOperator(region.GetId()); op != nil {
               timeout := op.IsTimeout()
               if step := op.Check(region); step != nil && !timeout {
                       operatorCounter.WithLabelValues(op.Desc(), "check").Inc()
                       c.sendScheduleCommand(region, step)
                       return
               }
               if op.IsFinish() {
                       log.Infof("[region %v] operator finish: %s", region.GetId(), op)
                       operatorCounter.WithLabelValues(op.Desc(), "finish").Inc()
                       c.removeOperator(op)
               } else if timeout {
                       log.Infof("[region %v] operator timeout: %s", region.GetId(), op)
                       operatorCounter.WithLabelValues(op.Desc(), "timeout").Inc()
                       c.removeOperator(op)
               }
       }
    ....
}
```

可以看到这个函数，先检查是否存在已有的操作，这个 operator 可以有好多种，比如 `AddReplica、RemoveReplica、TransferLeader`,
如果存在检查这个操作显示是到哪一步了，如果是没有结束并且没有超时，就会使用 `sendScheduleCommand` 通过 grpc 向这个region 在此发送此次操作。 要是以及完成或是超时分别错处响应的处理并删除这个操作。

在看 PD 如何将 Transfer leader 这个 opertor 加到 `c.operators` 里面的

```
func (l *balanceLeaderScheduler) Schedule(cluster schedule.Cluster) *schedule.Operator {
        schedulerCounter.WithLabelValues(l.GetName(), "schedule").Inc()
        region, newLeader := scheduleTransferLeader(cluster, l.GetName(), l.selector)
        if region == nil {
                return nil
        }

        // Skip hot regions.
        if cluster.IsRegionHot(region.GetId()) {
                schedulerCounter.WithLabelValues(l.GetName(), "region_hot").Inc()
                return nil
        }

        source := cluster.GetStore(region.Leader.GetStoreId())
        target := cluster.GetStore(newLeader.GetStoreId())
        if !shouldBalance(source, target, core.LeaderKind) {
                schedulerCounter.WithLabelValues(l.GetName(), "skip").Inc()
                return nil
        }
        l.limit = adjustBalanceLimit(cluster, core.LeaderKind)
        schedulerCounter.WithLabelValues(l.GetName(), "new_opeartor").Inc()
        step := schedule.TransferLeader{FromStore: region.Leader.GetStoreId(), ToStore: newLeader.GetStoreId()}
        return schedule.NewOperator("balance-leader", region.GetId(), core.LeaderKind, step)
}

...
func (c *coordinator) runScheduler(s *scheduleController) {
        defer c.wg.Done()
        defer s.Cleanup(c.cluster)

        timer := time.NewTimer(s.GetInterval())
        defer timer.Stop()

        for {
                select {
                case <-timer.C:
                        timer.Reset(s.GetInterval())
                        if !s.AllowSchedule() {
                                continue
                        }
                        if op := s.Schedule(c.cluster); op != nil {
                                c.addOperator(op)
                        }

                case <-s.Ctx().Done():
                        log.Infof("%v stopped: %v", s.GetName(), s.Ctx().Err())
                        return
                }
        }
}

```
可以看到`Schedule`这个函数最后返回的是一个 `operator` , `runScheduler` 调用 ` c.addOperator` 将这个`operator` 加到`c.operators`


## 在看 TiKV

接着看 TiKV 从 PD 收到 transfer leader 指令后会做哪些操作。(刚入坑 Rust，看 TiKV 还有点费劲，可能有些地方理解的有问题，还望指出来)

TiKV 首先是要接受到 PD 发送的命令，来看一个函数，这个函数是用来处理 PD 发送的命令

```
fn schedule_heartbeat_receiver(&mut self, handle: &Handle) {
        let ch = self.ch.clone();
        let store_id = self.store_id;
        let f = self.pd_client
            .handle_region_heartbeat_response(self.store_id, move |mut resp| {
                let region_id = resp.get_region_id();
                let epoch = resp.take_region_epoch();
                let peer = resp.take_target_peer();

                if resp.has_change_peer() {
                   // more
                } else if resp.has_transfer_leader() {
                    PD_HEARTBEAT_COUNTER_VEC
                        .with_label_values(&["transfer leader"])
                        .inc();

                    let mut transfer_leader = resp.take_transfer_leader();
                    info!(
                        "[region {}] try to transfer leader from {:?} to {:?}",
                        region_id,
                        peer,
                        transfer_leader.get_peer()
                    );
                    let req = new_transfer_leader_request(transfer_leader.take_peer());
                   send_admin_request(&ch, region_id, epoch, peer, req, None)
                } else {
                    PD_HEARTBEAT_COUNTER_VEC.with_label_values(&["noop"]).inc();
                }
            })
    // more
}
```

这段代码很好理解，就是先从 resp 中读取到 `region_id`、`peer`，然后在判断要执行的操作是什么，当执行的操作是 `transfer_leader`的时候，先是更新一下监控，然后在从 resp 中获取到 `leader` 该 `transfer` 到什么地方, 在然后呢，就是发送这个命令去执行了。

先看启动 启动 `transfer_leader` 函数

```
pub fn transfer_leader(&mut self, transferee: u64) {
     let mut m = Message::new();
     m.set_msg_type(MessageType::MsgTransferLeader);
     m.set_from(transferee);
     self.raft.step(m).is_ok();
 }
```

这个函数其实很简单，先设置一下消息类型，然后获取到目标 `leader`，`transferee` 就是目标`leader`，当然自己就是当前 `leader`。

在看处理过程

```
fn handle_transfer_leader(&mut self, m: &Message) {
    let lead_transferee = m.get_from();
    let last_lead_transferee = self.lead_transferee;
    if last_lead_transferee.is_some() {
        if last_lead_transferee.unwrap() == lead_transferee {
            info!(
                "{} [term {}] transfer leadership to {} is in progress, ignores request \
                 to same node {}",
                self.tag,
                self.term,
                lead_transferee,
                lead_transferee
            );
            return;
        }
        self.abort_leader_transfer();
        info!(
            "{} [term {}] abort previous transferring leadership to {}",
            self.tag,
            self.term,
            last_lead_transferee.unwrap()
        );
    }
    if lead_transferee == self.id {
        debug!(
            "{} is already leader. Ignored transferring leadership to self",
            self.tag
        );
        return;
    }
    // Transfer leadership to third party.
    info!(
        "{} [term {}] starts to transfer leadership to {}",
        self.tag,
        self.term,
        lead_transferee
    );
    // Transfer leadership should be finished in one electionTimeout
    // so reset r.electionElapsed.
    self.election_elapsed = 0;
    self.lead_transferee = Some(lead_transferee);
    if self.prs[&m.get_from()].matched == self.raft_log.last_index() {
        self.send_timeout_now(lead_transferee);
        info!(
            "{} sends MsgTimeoutNow to {} immediately as {} already has up-to-date log",
            self.tag,
            lead_transferee,
            lead_transferee
        );
    } else {
        self.send_append(lead_transferee);
    }
}

```

 这段代码稍有点长，我们可以一步一步的来看， 首先是进行一些检查，第一步是检查是否已经有 `transfer leader` 在执行，如果已经正在  `transfer leader` 并且目标 `leader` 相同的话，就退出这次操作，如果目标不同的话，那就 调用` self.abort_leader_transfer();` 这个函数放弃上一次正在执行的 `transfer leader`  操作。 紧接就是判断 目标 `leader`是不是自己，要是自己那就直接退出就好了，因为不需要`transfer leader`。

 下一步就是将目标leader保存在`leadTransferee`中，标示着有`transfer`正在进行，后续如果有请求`propose`进来，会检查这个`lead_transferee` 是不是存在，如果存在，其他操作就无法成功，也就是无法进行写操作。

 下一步就是检查 `transferee` 和`leader`的`log`是否一样新

 * 如果`log` 一致的话就会给`transferee`发送`MsgTimeoutNow`类型的消息，告诉`transferee`可以立即选主，不需要等到`election timeout`。
 * 如果 `log` 不一致，就会给 `lead_transferee`  发送一个`append` 的请求，追加 `log`。 ，leader在收到响应 `MsgAppResp`后,如果发现目前正处于`transfer leader` 过程中并且 `transferee`已经日志最新，则同样，给`transferee`发送`MsgTimeoutNow`。

## 参考

1. [TiKV 源码解析系列 - Placement Driver](https://pingcap.com/集群调度/blog-placement-driver-zh)
2. [三篇文章了解 TiDB 技术内幕 - 谈调度](https://pingcap.com/blog-tidb-internal-3-zh)
3. [etcd raft如何实现leadership transfer](https://zhuanlan.zhihu.com/p/27895034)
4. [PD](https://github.com/pingcap/pd)
5. [TiKV](https://github.com/pingcap/tikv)
