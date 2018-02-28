title: Percolator 论文笔记
author: cwen
date:  2018-02-28
update:  2018-02-28
tags:
    - Transaction
    - Percolator
    - 分布式
    - 分布式数据库

---

TiDB 的事务模型是参考 Google Percolator 事务模型，想去研究 TiDB 的事务模型，学习一下 Google Percolator 论文必不可少... <!--more-->

## Percolator 简介

### 背景
简单来说 Percolator 事务模型的出现是因为之前采用 MapReduce 设计的批量创建索引的系统不支持跨行事务以及不支持随机增量更新。于是乎 Percolator 就诞生了（ Google 还真是想啥有啥啊，实力真不是盖的 ）。

### 特性

Percolator的特点如下

* 为增量处理定制
* 处理结果强一致
* 针对大数据量（小数据量用传统的数据库即可）

Percolator 为可扩展的增量处理提供了两个主要抽象：

* 基于随机存取库的ACID事务
* 观察者(observers)–一种用于处理增量计算的方式

### 延迟问题

由于 Percolator 在设计的时候追求的是大数据量而对延迟没有特定要求，所有基于 Percolator 事物模型设计的系统，延迟和传统 DBMS 等数据库系统相比还是有很大的差距的。主要原因还是锁的问题。

* Percolator 使用一个 lazy 的方法去清理遗留下来的锁，比如一个 transactions 由于运行的机器挂掉了，这个 transactions 失败遗留下来的锁可能不会立刻清理，而是等待下一个使用到该数据的饿事务负责清理
* 缺少全局 deadlock 检查，这样就使交易冲突的情况增多，事务重试的次数就会变多，延迟就会随之增大。

## 设计

### Bigtable 概述

Percolator 是建立在 Bigtable 分布式系统的上层，Bigtable
是一个稀疏的、分布式的、持久化存储的多维度排序 Map。Map 的索引是行关键字、列关键字以及时间戳，Bigtable 以 SSTable 格式存储数据，并且这些 SSTables 是被存储在 GFS 上。

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-27%20at%2012.44.16%20AM.png)

### 事务

Percolator 提供了跨行、跨表的、基于快照隔离的ACID事务。

#### Snapshop isolation

![sanpshop isolation](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-27%20at%209.34.11%20AM.png)

如图，是基于 Snapshop isolation  的三个 transactions，每个 Transaction 都是开始与 a start timestamp, 并且结束与 a commit timestamp。在这个例子中：
* 因为 transaction 2 的 start timestamp 是在transaction 1 commit timestamp 之前的，所以 transaction 2 不能够读到 transaction 1 提交的信息
* transaction 3 可以同时看到 transaction 1 和 transaction 2 的提交信息
* transaction 1 和 transaction 2 是并发的执行，如果他们写入同一项，这两个事务中至少又一个会执行失败

#### Lock

Bigtable 本身并没有提供便捷的冲突解决和锁管理，Percolator 必须自己维护一套独立的锁的管理机制。

要求：

* 容错，当机器出现故障的时候不影响正确性，如果在两阶段提交的过程中锁出现了丢失，可能将两个有冲突的事物都提交成功。
* 高吞吐，上千台机器会同时请求获取锁。
* 低延迟, 每个 Get() 操作都需要读取一次锁。

Percolator 需要做到：

* 多副本，容错
* 分布式以及负载均衡，处理负载
* 数据持久化

BigTable 能够满足以上所有要求。所以 Percolator 在实现时，将实际的数据存于Bigtable中。

### Columns in Bigtable

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-27%20at%2011.32.55%20PM.png)

Percolator 在 Bigtable 上抽象了 5 Columns 去存储数据，如上图，其中有 3 和事务相关：

* c:lock

事务产生的锁，未提交的事务会写本项，会包含 primary lock 的位置。其映射关系为

${key,start_ts}=>${primary_key,lock_type,..etc}

    ${key} 数据的key
    ${start_ts} 事务开始时间
    ${primary} 该事务的primary的引用. primary是在事务执行时，从待修改的 keys中 选择一个作为primary,其余的则作为secondary.

* c:write

已提交的数据信息，存储数据所对应的时间戳。其映射关系为

${key,commit_ts}=>${start_ts}

    ${key} 数据的key
    ${commit_ts} 事务的提交时间
    ${start_ts} 该事务的开始时间,指向该数据在data中的实际存储位置。

* c:data

具体存储数据集，映射关系为

${key,start_ts} => ${value}

    ${key} 真实的key
    ${start_ts} 对应事务的开始时间
    ${value} 真实的数据值

### 简述 Percolator 事务流程

事务提交采用两阶段提交。两阶段提交通过 client 来协调。

#### Prewrite

![prewrite](http://7xnp02.com1.z0.glb.clouddn.com/prewrite.jpg)

> 图片 copy [雪姐 blog: Google Percolator 的事务模型](http://andremouche.github.io/transaction/percolator.html)

在提交的第一个阶段：其从 Oracle 获取代表当前物理时间的全局唯一时间戳作为当前事务的 start_ts，我们首先获取所有需要写入的 cell 的 lock（考虑到 client 故障情况，我们会任意指定一个 lock 为 primary，其余的作为 secondary），每个事务都会读取 meta 数据来检测事务是否存在冲突。

有两种冲突的情况：

1. 如果一个事务 A 看到 cell 的 write 列中已经存在一条记录，记录的时间戳晚于该事务开始的时间戳，那么说明存在写冲突，即说明有一个事务在 A 发起之后更新了 cell 的值，事务 A 需要 aborted。

2. 如果事务看到 cell 的 lock 列中存在任意一条锁记录，不管时间戳为多少，直接 aborted。

如果两种冲突都不存在，向 lock 列中写入上锁信息，并更新 data 列数据。

#### Commit

![commit](http://7xnp02.com1.z0.glb.clouddn.com/commit.jpg)

> 图片 copy [雪姐 blog: Google Percolator 的事务模型](http://andremouche.github.io/transaction/percolator.html)

如果没有 cell 冲突，那么说明事务可以提交，进行下一个阶段 commit：首先 client 会向时间服务器申请一个 timestamp 作为 commit_ts，表示 commit 的时间。然后 client 会释放事务中涉及的所有 cell 的锁（清空 lock 列），释放顺序从 primary lock开始。

释放锁之后便更新 write 列来使新的 read 能够读到本次事务的数据。write 列的数据表示：本次事务的数据已经成功更新至 cell 中，write 列中的数据包含了一个 timestamp，该 timestamp 表示本次事务数据的 timestamp，用户可以通过该 timestamp 来找到数据。一旦 primary 锁对应记录的 write 列数据可见了，代表该事务一定已经 commit 了，reader 此时能看到该事务的数据。

#### Get operation

![get](http://7xnp02.com1.z0.glb.clouddn.com/get.jpg)

> 图片 copy [雪姐 blog: Google Percolator 的事务模型](http://andremouche.github.io/transaction/percolator.html)


* Get 操作首先检查[0,start_ts]时间区间内Lock是否存在，若存在，说明可能有其他 transaction 正在写这个 cell，所以这个读 transaction 需要等待 lock 释放掉。如果不存在有冲突的 lock,则获取在 write 中合法的最新提交记录指向的在 data 中的位置。
* 从 data 中获取到相应的数据并返回。


#### clean lock

![clean lock](http://7xnp02.com1.z0.glb.clouddn.com/rollback.jpg)

> 图片 copy [雪姐 blog: Google Percolator 的事务模型](http://andremouche.github.io/transaction/percolator.html)

若客户端在 Commit 一个事务时，出现了异常，Prepare 时产生的锁会被留下。为避免将新事务 hang 住，Percolator 必须清理这些锁。

Percolator 用 lazy 方式处理这些锁：当事务 A 在执行时，发现事务 B 造成的锁冲突，事务 A 将决定事务 B 是否失败，以及清理事务 B 的那些锁。
对事务 A 而言，能准确地判断事务 B 是否成功是关键。Percolator 为每个事务设计了一个元素cell作为事务是否成功的同步标准，该元素产生的 lock 即为 primary lock。A 和 B 事务都能确定事务 B 的 primary lock（因为这个priarmy lock被写入了B事务其它所有涉及元素的lock里面）。执行一个清理 clean up 或者提交 commit 操作需要修改该primary lock，由于这些修改是基于 Bigtable 去做，所以只有一个清理或提交会成功。注意：

* 在B提交 commit 之前,它会先确保其 primary lock 被 write record 所替代（即往 primary 的 write 写提交数据，并删除对应的 lock）。
* 在 A 清理 B 的锁之前，A 必须检查 B 的 primary 以确保 B 未被提交，如果 B 的 primary 存在，则 B 的锁可以被安全的清理掉。

当客户端在执行两阶段提交的 commit 阶段失败时，事务依旧会留下一个提交点 commit point(至少一条记录会被写入 write 中)，但可能会留下一些 lock 未被处理掉。一个事务能够从其 primary lock 中获取到执行情况：

* 如果 priarmy lock 已被 write 所替代，也就是说该事务已被提交，可以放心的清理 B 事务的所有 lock
* 如果 primary lock 存在，事务被 roll back (因为我们总是最先提交 primary ,所以 primary 未被提交时，可以安全地执行回滚)

### 案例

![1](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-28%20at%2012.59.40%20AM.png)
> 初始状态下，Bob 的帐户下有 $10（首先查询 column write 获取最新时间戳数据,获取到 data@5,然后从 column data 里面获取时间戳为5的数据，即 $10），Joe 的帐户下有 $2。

![2](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-28%20at%2012.59.46%20AM.png)
> 转账开始，使用 stat timestamp=7 作为当前事务的开始时间戳，将 Bob 选为本事务的 primary，通过写入 Column Lock 锁定 Bob 的帐户，同时将数据 7:$3 写入到 Column,data 列。

![3](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-28%20at%2012.59.53%20AM.png)
> 同样的，使用 stat timestamp=7，锁定 Joe 的帐户，并将 Joe 改变后的余额写入到 Column,data ,当前锁作为 secondary 并存储一个指向 primary 的引用（当失败时，能够快速定位到 primary 锁，并根据其状态异步清理）

![4](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-28%20at%201.00.00%20AM.png)
> 事务带着当前时间戳 commit timestamp=8 进入 commit 阶段：删除 primary 所在的 lock，并在 write 列中写入从提交时间戳指向数据存储的一个指针 commit_ts=>data @7。至此，读请求过来时将看到 Bob 的余额为3。

![5](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-02-28%20at%201.00.06%20AM.png)
> 依次在 secondary 项中写入wirte并清理锁，整个事务提交结束。

## 参考

1. [Large-scale Incremental Processing
Using Distributed Transactions and Notifications](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf)
2. [雪姐 blog: Google Percolator 的事务模型](http://andremouche.github.io/transaction/percolator.html) - 很多描述是从雪姐 blog 直接 copy 过来
3. [Percolator简单翻译与个人理解](https://www.jianshu.com/p/05194f4b29dd)





