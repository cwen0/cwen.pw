title: MVCC In TiKV
author: cwen
date:  2017-11-28
update:  2017-11-28
tags:
    - 分布式
    - TiKV
    - 分布式数据库
    - MVCC
    - 并发控制
    - 2PL
    - Transaction

---


## 乐观锁和悲观锁
### 乐观锁
乐观锁呢，读写事务，在真正的提交之前，不加读/写锁，而是先看一下数据的版本/时间戳，等到真正提交的时候再看一下版本/时间戳，如果两次相同，说明别人期间没有对数据进行过修改，那么就可以放心提交。乐观体现在，访问数据时不提前加锁。在资源冲突不激烈的场合，用乐观锁性能较好。如果资源冲突严重，乐观锁的实现会导致事务提交的时候经常看到别人在他之前已经修改了数据，然后要进行回滚或者重试，还不如一上来就加锁。
### 悲观锁
一个读写事务在运行的过程中在访问数据之前先加读/写锁这种实现叫做悲观锁，悲观体现在，先加锁，独占数据，防止别人加锁。

> 关于乐观锁悲观锁的解释 copy 自：[吴镝大神知乎的回答](https://www.zhihu.com/question/27876575/answer/73330077), 感觉回答的最简洁贴切

## 并发控制

### 可串行化

多个事务的并发执行是正确的，当且仅当其结果与按某一次序串行地执行它们时的结果相同。我们称这种调度策略为可串行化（Serializable）的调度。

可串行性是并发事务正确性的准则。按这个准则规定，一个给定的并发调度，当且仅当它是可串行化的，才认为是正确调度。DBMS的并发控制机制必须提供一定的手段来保证调度是可串行化的，目前DBMS普遍采用封锁方法实现并发操作调度的可串行性，从而保证调度的正确性。

两段锁（Two-Phase Locking，简称2PL）协议就是保证并发调度可串行性的封锁协议。

### 2PL

两段锁（Two-Phase Locking，简称2PL）是指所有事务都必须分为两阶段对数据进行加锁和解锁：

1. 对任何数据进行读、写操作之前，首先要先申请并获得对该数据的封锁
2. 在释放一个封锁以后，事务不再获得任何其他封锁

在“两段”锁协议中，事务分为两个阶段：

* 第一阶段是获得封锁，也称为扩展阶段。这在阶段，事务可以申请获得任何数据项上的任何类型的锁，但是不能释放任何锁。
* 第二阶段是释放封锁，也称为收缩阶段。在这阶段，事务可以释放任何数据项上的任何类型的琐，但是不能再申请任何琐。

如图：

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202017-11-27%20at%2011.44.23%20PM.png)

### 2PL 一些缺点

* 读锁和写锁会相互阻滞（block）。
* 大部分事务都是只读（read-only）的，所以从事务序列（transaction-ordering）的角度来看是无害的。如果使用基于锁的隔离机制，而且如果有一段很长的读事务的话，在这段时间内这个对象就无法被改写，后面的事务就会被阻塞直到这个事务完成。这种机制对于并发性能来说影响很大。



### MVCC

MVCC - 多版本并发控制（Multi-Version Concurrency Control）, 在 MVCC 中，每当想要更改或者删除某个数据对象时，DBMS 不会在原地去删除或这修改这个已有的数据对象本身，而是创建一个该数据对象的新的版本，这样的话同时并发的读取操作仍旧可以读取老版本的数据，而写操作就可以同时进行。这个模式的好处在于，可以让读取操作不再阻塞，事实上根本就不需要锁。这是一种非常诱人的特型，以至于在很多主流的数据库中都采用了 MVCC 的实现，比如说 PostgreSQL，Oracle，Microsoft SQL Server 等。

> 此处 copy 自[TiKV 的 MVCC（Multi-Version Concurrency Control）机制](https://pingcap.com/blog-cn/mvcc-in-tikv/)

## MVCC in TiKV

TiKV 目前的底层存储还是使用了 rocksdb， 也就是目前我们的所有数据也就是存在 rocksdb 内。目前 TiKV 会启动两个 rocksdb 实例，默认 rocksdb 实例将 KV 数据存储在内部的 default、write 和 lock 3 个 CF 内。

* default CF 存储的是真正的数据；
* write CF 存储的是数据的版本信息（MVCC）以及索引相关的数据；
* lock CF 存储的是锁信息。

Raft RocksDB 实例存储 Raft log。

* default CF 主要存储的是 raft log。

rocksdb 的 cf 是一个逻辑划分数据库的能力，也就是说做到了想多隔离的存储，但是 rocksdb 提供跨 cf 的原子写操作，不同的 cf 共用同一个 WAL，但是使用不同 memtable 和 ssl。（具体 rocksdb 相关的知识可以查阅 rocksdb 相关文档）。也就是说会有一个单独的 cf 用来存储 MVCC 的版本信息。 TiKV 的 MVCC 实现是通过在 Key 后面添加 Version 来实现，简单来说，没有 MVCC 之前，可以把 TiKV 看做这样的：

```
                Key1 -> Value
	            Key2 -> Value
	            ……
	            KeyN -> Value
```

有了 MVCC 之后，TiKV 的 Key 排列是这样的：

```
	            Key1-Version3 -> Value
	            Key1-Version2 -> Value
	            Key1-Version1 -> Value
	            ……
	            Key2-Version4 -> Value
	            Key2-Version3 -> Value
	            Key2-Version2 -> Value
	            Key2-Version1 -> Value
	            ……
	            KeyN-Version2 -> Value
	            KeyN-Version1 -> Value
	            ……
```

注意，对于同一个 Key 的多个版本，我们把版本号较大的放在前面，版本号小的放在后面，这样当用户通过一个 Key + Version 来获取 Value 的时候，可以将 Key 和 Version 构造出 MVCC 的 Key，也就是 Key-Version。然后可以直接 Seek(Key-Version)，定位到第一个大于等于这个 Key-Version 的位置。

> 此处 copy 自申栎哥文章[三篇文章了解 TiDB 技术内幕 - 说存储](https://pingcap.com/blog-cn/tidb-internal-1/)

现在从代码看起：

我们来看 tikv 的 Storage pkg，可以看到这个 pkg 里面有个 mvcc pkg，没错具体的 mvcc 操作实现就是定义在 mvcc 这个 pkg 里面。

### 先看 Storage

```
pub struct Storage {
    engine: Box<Engine>,
    sendch: SyncSendCh<Msg>,
    handle: Arc<Mutex<StorageHandle>>,
    ...
}
impl Storage {
 pub fn start(&mut self, config: &Config) -> Result<()> {
        let mut handle = self.handle.lock().unwrap();
        if handle.handle.is_some() {
            return Err(box_err!("scheduler is already running"));
        }

        let engine = self.engine.clone();
        let builder = thread::Builder::new().name(thd_name!("storage-scheduler"));
        let rx = handle.receiver.take().unwrap();
        let sched_concurrency = config.scheduler_concurrency;
        let sched_worker_pool_size = config.scheduler_worker_pool_size;
        let sched_pending_write_threshold = config.scheduler_pending_write_threshold.0 as usize;
        let ch = self.sendch.clone();
        let h = builder.spawn(move || {
            let mut sched = Scheduler::new(
                engine,
                ch,
                sched_concurrency,
                sched_worker_pool_size,
                sched_pending_write_threshold,
            );
            if let Err(e) = sched.run(rx) {
                panic!("scheduler run err:{:?}", e);
            }
            info!("scheduler stopped");
        })?;
        handle.handle = Some(h);

        Ok(())
    }
}
```
其实 Storage 是实际接受外部指令, Storage 内包含三个字段：

* Engine，数据库操作的接口，raftkv 以及 rocksdb 实现了这个接口，具体实现可以看 engine/raftkv.rs, engine/rocksdb.rs
* SyncSendCh, 一个 channel 内部类型是 Msg, 用来存储 scheduler event 的 channel
* StorageHanle, 是处理从sench 接受到指令，通过 mio 来处理 IO

可以看到 Storage 最后启动了调度器，然后不断的接受客户端指令，然后在传给 scheduler, 然后调度器执行相应的过程或者调用相应的异步函数。在调度器中有两种操作类型，读和写。

### MVCC MvccReader

```
pub struct MvccReader {
 ....
}

impl MvccReader{
    pub fn new() {...};
    pub fn get_statistics(&self) -> &Statistics {...}
    pub fn set_key_only(&mut self, key_only: bool) {...}
    pub fn load_data(&mut self, key: &Key, ts: u64) -> Result<Value> {...}
    pub fn load_lock(&mut self, key: &Key) -> Result<Option<Lock>> {...}
    pub fn get(&mut self, key: &Key, mut ts: u64) -> Result<Option<Value>> {...}
    pub fn get_txn_commit_info(
        &mut self,
        key: &Key,
        start_ts: u64,
    ) -> Result<Option<(u64, WriteType)>> {...}
    pub fn seek_ts(&mut self, ts: u64) -> Result<Option<Key>> {...}
    pub fn seek(&mut self, mut key: Key, ts: u64) -> Result<Option<(Key, Value)>> {...}
    pub fn reverse_seek(&mut self, mut key: Key, ts: u64) -> Result<Option<(Key, Value)>> {...}
    ...
}

```

看 MvccReader 结构很容易理解，各种读的操作。


### MVCCTxn

```
pub struct MvccTxn {
    reader: MvccReader,
    start_ts: u64,
    writes: Vec<Modify>,
    write_size: usize,
}
impl MvccTxn {
    pub fn prewrite(
        &mut self,
        mutation: Mutation,
        primary: &[u8],
        options: &Options,
    ) -> Result<()> {...}

    pub fn commit(&mut self, key: &Key, commit_ts: u64) -> Result<()> {...}
    pub fn rollback(&mut self, key: &Key) -> Result<()> {...}
}
```

MVCCTxn 实现了两段提交（2-Phase Commit，2PC），整个 TiKV 事务模型的核心。在一段事务中，由两个阶段组成。

* Prewrite

选择一个 row 作为 primary row， 余下的作为 secondary row。 对primary row 上锁. 在上锁之前，会检查是否有其他同步的锁已经上到了这个 row 上 或者是是否经有在 startTS 之后的提交操作。这两种情况都会导致冲突，一旦都冲突发生，就会回滚（rollback）。 对于 secondary row 重复以上操作。

* Commit

Rollback 在Prewrite 过程中出现冲突的话就会被调用。

* Garbage Collector

很容易发现，如果没有垃圾收集器（Gabage Collector） 来移除无效的版本的话，数据库中就会存有越来越多的 MVCC 版本。但是我们又不能仅仅移除某个 safe point 之前的所有版本。因为对于某个 key 来说，有可能只存在一个版本，那么这个版本就必须被保存下来。在TiKV中，如果在 safe point 前存在Put 或者Delete，那么说明之后所有的 writes 都是可以被移除的，不然的话只有Delete，Rollback和Lock 会被删除。

> 此处部分 copy 自[TiKV 的 MVCC（Multi-Version Concurrency Control）机制](https://pingcap.com/blog-cn/mvcc-in-tikv/)

## 参考

1. [Two-phase locking](https://en.wikipedia.org/wiki/Two-phase_locking)
2. [OCC和MVCC的区别是什么？](https://www.zhihu.com/question/60278698)
3. [三篇文章了解 TiDB 技术内幕 - 说存储](https://pingcap.com/blog-cn/tidb-internal-1/)
4. [TiKV 的 MVCC（Multi-Version Concurrency Control）机制](https://pingcap.com/blog-cn/mvcc-in-tikv/)




