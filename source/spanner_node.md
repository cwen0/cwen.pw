title: Spanner 论文笔记
author: cwen
date:  2017-11-05
update:  2017-11-05
tags:
    - 分布式
    - Spanner
    - 数据库
    - 分布式数据库

---
刚到公司的时候，记得奇哥就推荐了几篇基础论文（尴尬，目前我还有完全看完），Spanner 论文就是其中之一，同时 TiKV 的设计架构也是收到了 Spanner 的启发，在阅读 Spanner 论文的同时，记录些东西，方便以后回顾复习...  <!--more-->

## 结构

一套 Spanner 系统称为一个 universe, Spanner 论文上说目前 google 总共部署了三套  Spanner，一个开发，一个测试，一个线上。

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-8d4d85a54036697f.png)

上图是 Spanner 论文上的结构图，可以看出Spanner 是由多个 zone （管理部署的基本单元） 组成，一个数据中心中可以有多个 zone。 一个 zone 包含一个 zonemaster和一百或者几千个 spanserver。zonemaster 把数据分配给 spanserver，spanserver 把数据提供给客户端。客户端使用没个 zone 上的 location proxy 来定位提供服务的 spanserver，spanerver 提供具体的服务。（TiDB 的 PD 与 TiKV 之间的配合还真是有异曲同工之妙啊）。Universe master 主要是一个控制台，它显示了关于 zone 的各种状态信息，可以用于相互之间的调试。Placement driver 会周期性地与 spanserver 进行交互，来发现那些需要被转移的数据，或者是为了满足新的副本约束条件，或者是为了进行负载均衡。

###  spanserver 实现

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-b99c0dffe8361eb9.png)

从图中可以看出，每个 spanserver 负载管理 100 - 1000 个称为 tablet 的数据结构的实例。一个 tablet 就类似于 BigTable 中的 tablet，也实现了下面的映射: `(key:string, timestamp:int64)->string` 。

spanserver 不同于 bigtable 的地方在于，spanserver 其实更像一个完整关系数据库，而不是一个纯粹的键值存储，spanserver 会为每一个数据分配一个时间戳（有点类似 MVCC），tablet 的状态存储在类似于 B-树的文件集合以及 WAL（write-ahead-log)中，并且都存放在 GFS 的升级版 Colossus 文件系统之上。（感觉结构有点复杂啊，也可能是我先了解了 TiKV 的设计，有点先入为主了）。

再说 tablet 的上层，从图中可以看到，是用 Paxos 状态机来做副本管理（Paxos 算饭没有仔细研究过，钥匙又地方描述的有问题，欢迎指正）。论文里面讲到，目前每个 tablet 上实现了一个 Paxos 状态机，每次写操作的时候都需要写两份数据，一份落到 tablet 的日志中，一份是写入，Paxos 的日志中，目前Paxos 的 leader 实现了支持长寿命的 leader ，简单来说就是使用了租约的机制（lease TiKV 中 Raft leader 也是如此）。写操作必须在领导者上初始化 Paxos 协议，读操作可以直接从底层的任何副本的 tablet 中访问状态信息，只要这个副本足够新。副本的集合被称为一个 Paxos group。

对于每个是领导者的副本而言，每个 spanserver 会实现一个锁表来实现并发控制。这个锁表包含了两阶段锁机制的状态:它把键的值域映射到锁状态上面。注意，采用一个长寿命的 Paxos 领导者，对于有效管理锁表而言是非常关键的。在 BigTable 和 Spanner 中，我们都专门为长事务做了设计，比如，对于报表操作，可能要持续几分钟，当存在冲突时，采用乐观并发控制机制会表现出很差的性能。对于那些需要同步的操作，比如事务型的读操作，需要获得锁表中的锁，而其他类型的操作则可以不理会锁表。

对于每个扮演领导者角色的副本，每个 spanserver 也会实施一个事务管理器来支持分布式事务。这个事务管理器被用来实现一个 participant leader，该组内的其他副本则是作为 participant slaves。如果一个事务只包含一个 Paxos 组(对于许多事务而言都是如此)，它就可以绕过事务管理器，因为锁表和 Paxos 二者一起可以保证事务性。如果一个事务包含了多 于一个 Paxos 组，那些组的领导者之间会彼此协调合作完成两阶段提交。其中一个参与者组，会被选为协调者，该组的 participant leader 被称为 coordinator leader，该组的 participant slaves 被称为 coordinator slaves。每个事务管理器的状态，会被保存到底层的 Paxos 组。目前 TiKV 的事务好像都走的是两阶段提交，不知到要是也搞个事务管理器效果会如何？

### dictionary

Spanner 对具有公共前缀的键做了一个抽象，称为 dictionary。目前一个 dictionary 是数据存放的基本单位。。属于一个目录的所有数据，都具有相同的副本配置。 当数据在不同的 Paxos 组之间进行移动时，会一个目录一个目录地转移，如下图所示。Spanner 可能会移动一个目录从而减轻一个 Paxos 组的负担，也可能会把那些被频繁地一起访问的目录都放置到同一个组中，或者会把一个目录转移到距离访问者更近的地方。当客户端操作正在进行时，也可以进行目录的转移。我们可以预期在几秒内转移 50MB 的目录。

dictionary 是数据复制和placement配置的基本单位。spanner中负载均衡的最小单位也是dictionary，同时提供方法MoveDir可以手动将一个dictionary移动到指定的zone

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-192ba58099b34f11.png)


## 数据模型

spanner的行模型是 `(key:string, timestamp:int64) -> row content`，可以看到跟big table的模型最大的不同是这里强化了row的概念，不再突出column；这样spanner的timestamp是赋给整行数据的，是有物理意义的，这使得spanner更像一个实现多版本并发的数据库，而在big table中，timestamp仅仅用于保存多个版本的key-value，跟并发完全无关；我觉得这也是为什么spanner称自己为semi-relational 数据库，而big table只称自己是semi-structure 数据库的原因。

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-c784a00321761682.png)

Spanner 的数据模型不是纯粹关系型的，它的行必须有名称。更准确地说，每个表都需 要有包含一个或多个主键列的排序集合。这种需求，让 Spanner 看起来仍然有点像键值存储: 主键形成了一个行的名称，每个表都定义了从主键列到非主键列的映射。当一个行存在时，必须要求已经给行的一些键定义了一些值(即使是 NULL)。采用这种结构是很有用的，因为这可以让应用通过选择键来控制数据的局部性。

### TrueTime API

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-d1557f77c3323255.png)

rueTime API 是一个非常有创意的东西，可以同步全球的时间。TT.now()可以获得一个绝对时间TTinterval，这个值和UnixTime是相同的，同时还能够得到一个误差e。TT.after(t)和TT.before(t)是基于TT.now()实现的。那这个TrueTime API实现靠的是GFS和原子钟。之所以要用两种技术来处理，是因为导致这两个技术的失败的原因是不同的。GPS会有一个天线，电波干扰会导致其失灵。原子钟很稳定。当GPS失灵的时候，原子钟仍然能保证在相当长的时间内，不会出现偏差。实际部署的时候。每个数据中心需要部署一些Master机器，其他机器上需要有一个slave进程来从Master同步。有的Master用GPS，有的Master用原子钟。

## 并发控制

Spanner使用TrueTime来控制并发，实现外部一致性。支持以下几种事务。

* 读写事务
* 只读事务
* 快照读，客户端提供时间戳
* 快照读，客户端提供时间范围

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-91fa02b85deaedff.png)

上表是Spanner现在支持的事务。单独的写操作都被实现为读写事务 ； 单独的非快照被实现为只读事务。事务总有失败的时候，如果失败，对于这两种操作会自己重试，无需应用自己实现重试循环。

时间戳的设计大大提高了只读事务的性能。事务开始的时候，要声明这个事务里没有写操作，只读事务可不是一个简单的没有写操作的读写事务。它会用一个系统时间戳去读，所以对于同时的其他的写操作是没有Block的。而且只读事务可以在任意一台已经更新过的replica上面读。

对于快照读操作，可以读取以前的数据，需要客户端指定一个时间戳或者一个时间范围。Spanner会找到一个已经充分更新好的replica上读取。

还有一个有趣的特性的是，对于只读事务，如果执行到一半，该replica出现了错误。客户端没有必要在本地缓存刚刚读过的时间，因为是根据时间戳读取的。只要再用刚刚的时间戳读取，就可以获得一样的结果。

### Paxos 领导者租约

Spanner 的 Paxos 实现中使用了时间化的租约，来实现长时间的领导者地位(默认是 10秒)。一个潜在的领导者会发起请求，请求时间化的租约投票，在收到指定数量的投票后，这个领导者就可以确定自己拥有了一个租约。一个副本在成功完成一个写操作后，会隐式地延期自己的租约。对于一个领导者而言，如果它的租约快要到期了，就要显示地请求租约延期。另一个领导者的租约有个时间区间，这个时间区间的起点就是这个领导者获得指定数量的投票那一刻，时间区间的终点就是这个领导者失去指定数量的投票的那一刻(因为有些投票已经过期了)。Spanner 依赖于下面这些“不连贯性”:对于每个 Paxos 组，每个 Paxos 领 导者的租约时间区间，是和其他领导者的时间区间完全隔离的。

Spanner 实现允许一个 Paxos 领导者通过把 slave 从租约投票中释放出来这种方式，实现领导者的退位。为了保持这种彼此隔离的不连贯性，Spanner 会对什么时候退位做出限制。把 smax 定义为一个领导者可以使用的最大的时间戳。在退位之前，一个领导者必须等到 TT.after(smax)是真。

### 为读写事务分配时间戳

事务读和写采用两段锁协议。当所有的锁都已经获得以后，在任何锁被释放之前，就可以给事务分配时间戳。对于一个给定的事务，Spanner 会为事务分配时间戳，这个时间戳是 Paxos 分配给 Paxos 写操作的，它代表了事务提交的时间。

Spanner 依赖下面这些单调性:在每个 Paxos 组内，Spanner 会以单调增加的顺序给每个 Paxos 写操作分配时间戳，即使在跨越多个领导者时也是如此。一个单个的领导者副本，可以很容易地以单调增加的方式分配时间戳。在多个领导者之间就会强制实现彼此隔离的不连 贯:一个领导者必须只能分配属于它自己租约时间区间内的时间戳。要注意到，一旦一个时间戳 s 被分配，smax 就会被增加到 s，从而保证彼此隔离性(不连贯性)。

Spanner 也会实现下面的外部一致性:如果一个事务 T2 在事务 T1 提交以后开始执行， 那么，事务 T2 的时间戳一定比事务 T1 的时间戳大。对于一个事务 Ti 而言，定义开始和提交事件eistart和eicommit，事务提交时间为si。对外部一致性的要求就变成了:
tabs(e1commit )<tabs(e2start ) s1<s2。执行事务的协议和分配时间戳的协议，遵守两条规则，二者一起保证外部一致性。对于一个写操作 Ti 而言，担任协调者的领导者发出的提交请求的事件为eiserver 。

Start. 为一个事务 Ti 担任协调者的领导者分配一个提交时间戳 si，不会小于 TT.now().latest 的值，TT.now().latest的值是在esierver事件之后计算得到的。要注意，担任参与者的领导者， 在这里不起作用。第 4.2.1 节描述了这些担任参与者的领导者是如何参与下一条规则的实现的。

Commit Wait. 担任协调者的领导者，必须确保客户端不能看到任何被 Ti 提交的数据，直到 TT.after(si)为真。提交等待，就是要确保 si 会比 Ti 的绝对提交时间小。证明如下:

![](http://7xnp02.com1.z0.glb.clouddn.com/1752522-fff83c4dbdc512f1.png)

###  读写事务

正如BigTable一样，Spanner的事务是会将所有的写操作先缓存起来，在Commit的时候一次提交。这样的话，就读不出在同一个事务中写的数据了。不过这没有关系，因为Spanner的数据都是有版本的。

在读写事务中使用wound-wait算法来避免死锁。当客户端发起一个读写事务的时候，首先是读操作，他先找到相关数据的leader replica，然后加上读锁，读取最近的数据。在客户端事务存活的时候会不断的向leader发心跳，防止超时。当客户端完成了所有的读操作，并且缓存了所有的写操作，就开始了两阶段提交。客户端闲置一个coordinator group，并给每一个leader发送coordinator的id和缓存的写数据。

leader首先会上一个写锁，他要找一个比现有事务晚的时间戳。通过Paxos记录。每一个相关的都要给coordinator发送他自己准备的那个时间戳。

Coordinatorleader一开始也会上个写锁，当大家发送时间戳给他之后，他就选择一个提交时间戳。这个提交的时间戳，必须比刚刚的所有时间戳晚，而且还要比TT.now()+误差时间 还有晚。这个Coordinator将这个信息记录到Paxos。

在让replica写入数据生效之前，coordinator还有再等一会。需要等两倍时间误差。这段时间也刚好让Paxos来同步。因为等待之后，在任意机器上发起的下一个事务的开始时间，都比如不会比这个事务的结束时间早了。然后coordinator将提交时间戳发送给客户端还有其他的replica。他们记录日志，写入生效，释放锁。



### 只读事务

对于只读事务，Spanner首先要指定一个读事务时间戳。还需要了解在这个读操作中，需要访问的所有的读的Key。Spanner可以自动确定Key的范围。

如果Key的范围在一个Paxos group内。客户端可以发起一个只读请求给group leader。leader选一个时间戳，这个时间戳要比上一个事务的结束时间要大。然后读取相应的数据。这个事务可以满足外部一致性，读出的结果是最后一次写的结果，并且不会有不一致的数据。

如果Key的范围在多个Paxos group内，就相对复杂一些。其中一个比较复杂的例子是，可以遍历所有的group leaders，寻找最近的事务发生的时间，并读取。客户端只要时间戳在TT.now().latest之后就可以满足要求了。

## 参考

1. [Spanner: Google’s Globally-Distributed Database](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/spanner-osdi2012.pdf)
2. [spnner 论文中文版](http://linbingdong.com/2017/02/10/%E5%85%A8%E7%90%83%E5%88%86%E5%B8%83%E5%BC%8F%E6%95%B0%E6%8D%AE%E5%BA%93%EF%BC%9AGoogle%20Spanner%EF%BC%88%E8%AE%BA%E6%96%87%E7%BF%BB%E8%AF%91%EF%BC%89/)
3. [spanner与bigtable](http://www.voidcn.com/article/p-wxiysjtf-ho.html)
4. [Google全球级分布式数据库Spanner原理](http://blog.jobbole.com/28096/)

