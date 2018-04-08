title: TiDB 性能优化之玄学调优
author: cwen
date:  2018-04-08
update:  2018-04-08
tags:
    - 分布式
    - 分布式数据库
    - TiDB
    - 性能优化

---
TiDB 玄学调优是我们自黑比较狠的梗之一，当初 TiKV 里大把的参数让我头疼了好久 ( TiKV 的配置参数大多数是和 RocksDB 相关 )。好在现在很多参数现在的默认值已经很科学大大的减少了我们玄学调优的难度。
一下是是我平时在测试 TiDB 时候，进行瓶颈定位的过程... <!--more-->

我们以 sysbench 测试为例，使用的测试脚本：https://github.com/pingcap/tidb-bench/tree/master/sysbench
可用机器公共四台，机器配置：CPU 16 核 / Mem 110 GB  / Disk 1024GB SSD

## 硬件性能测试

测试之前先进行基本的硬件测试，比如磁盘 fio 测试，网络延迟以及网速的测试，这些测试有利于排除一些干扰因素。
这地方就提供几本的测试方法，之前已经测试过了，此处就不实际测试了。

测试磁盘：推荐 fio （fio 使用需谨慎，不要直接写裸设备，笔者连续写坏好几块盘，都是血泪啊），dd，sysbench...
```
// fio
fio -ioengine=libaio -bs=32k -direct=1 -thread -rw=read  -size=10G -filename=test -name="PingCAP max throughput" -iodepth=4 -runtime=60

// dd
dd bs=4k count=250000 oflag=direct if=/dev/zero of=./dd_test.txt

// sysbench
sysbench --test=fileio --file-num=16 --file-block-size=16384 --file-total-size=2G prepare
sysbench --test=fileio --file-num=16 --file-block-size=16384 --file-total-size=2G --num-threads=4 --max-requests=100000000 --max-time=180 --file-test-mode=seqwr --file-extra-flags=direct run
sysbench --test=fileio --file-num=16 --file-block-size=16384 --file-total-size=2G --num-threads=4 --max-requests=100000000 --max-time=180 --file-test-mode=seqwr --file-extra-flags=direct clean
```

测试延迟：直接 ping 查看延迟以及丢包情况

测试网速: 最简单 dd + scp，dd + pv + nc , iperf

```
// scp
dd if=/dev/zero bs=2M count=2048 of=./test // 生成 4 GB 文件
scp ./test ip ip

// dd + pv + nc
dd if=/dev/zero bs=1M count=1024 of=./test
pv test| nc 10.0.1.8 5433 // 发送
nc -vv -l -p 5433 > test  // 接收

// iperf
iperf -s            // server
iperf -c 10.0.1.4   // client
```

## sysbench oltp 测试瓶颈定位
运行 sysbench oltp 测试， sysbench 默认 oltp 一个事务包含 18 条 SQL:

```
10 * point select
1 * simple range select
1 * sum range
1 * order range
1 * distinct range
1 * index update
1 * non index update
1 * insert
1 * delete
```

从事务的组合上来看度占大部分，经验告诉我们最先读多属于 CPU 密集型的操作，TiDB 的 CPU 最有可能成为瓶颈，我们先使用 1 TiDB + 3 TiKV 的架构进行测试，实际观察一下运行情况。

使用 [tidb-ansible](https://github.com/pingcap/tidb-ansible) 部署集群

部署拓扑：
|    ip      | |
| ---------- | --- |
| 10.0.1.6   |  tidb / pd / sysbench |
| 10.0.1.7   |  tikv |
| 10.0.1.8   |  tikv |
| 10.0.1.9   |  tikv |

测试流程：prepare -> run

运行测试脚本，先观察 tidb / tikv 的 cpu 使用情况（一般来说 oltp 测试，主要瓶颈会集中在 tidb 的 cpu 上，要又重点的去排查问题）

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-04-08%20at%2012.16.54%20AM.png)

可以看到 tidb 的机器的 load 已经达到了 20+，这是一台 15 核的机器，感觉这台机器的 cpu 几乎已经被榨干了，我们都不需要去看别的瓶颈，这 tidb 的cpu 已经限制了整体性能。那么我们接下来要做的就是要把这个瓶颈去掉，**最直接的方法就是升级硬件或是加机器，我们尝试增加机器，同时主要注意增加压力**。（没有多余的机器就不做实验了） 当我们把 tidb 机器的数量也增加到 3 台或是更多的时候，我们就会发现瓶颈发生了转移，这个时候瓶颈就会从 tidb 转移到了 tikv，tikv 有个 end-point-concurrency 这样一个东西用来处理 tidb 的请求，我们使用 top -H 来观察一下

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-04-08%20at%2012.31.58%20AM.png)

对应 grafana 监控上的就是

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-04-08%20at%2012.34.02%20AM.png)

end-point-concurrency 什么时候算是出现瓶颈呢？ 默认 end-point-concurrency  是使用 4个线程，那么我们就可以这么认为，当 top -H 的时候发现这个四个线程的 cpu 使用都超过了 90% ，那么就可以认为 end-point-concurrency 限制了整个系统的吞吐，接下来我们要做的就是如何来把这个瓶颈给消除掉，如果目前 tikv 的机器整体 cpu 使用还是很闲，那么我们就**增加这个 end-point-concurrency 的线程数量**，这个就要修改 tikv 的启动参数了,这个可以参考文档：https://github.com/pingcap/docs-cn/blob/master/op-guide/tune-tikv.md 。
（grpc 线程也是会出现瓶颈，改进方法和 end-point-concurrency 类似，直接增加线程数量）

当我们解决掉 end-point-concurrency，我们会发现系统的整体吞吐会有所提高，但是此时又会出现新的瓶颈，以我往常的经验来说，接下来的瓶颈应该会出现在  end-point-work 这个线程上，为嘛会出现在这个线程上呢？其实要是知道这个线程是用来干嘛的就很好理解了，这个现场是用来调度 end-point-concurrency 这个玩意掉，当 end-point-concurrency 线程多了，那么调度就会出现瓶颈，遗憾的是 end-point-work 是单线程，我们要解决这个的瓶颈，**只能提升单核 cpu 的能力，或是增加 tikv 的数量**。
### Summary

* 瓶颈
    * TiDB CPU 一般会优先成为瓶颈
    * TiKV 的 endpoint 线程
    * grpc 线程
    * TiKV 的 end-point-work 线程
* 优化
    * 加 TiDB 节点，可以尝试在 TiKV 机器上添加（ TiKV CPU比较充足）
    * 调整 end-point-concurrency 参数，增大 endpoint 线程数量
    * 调整 grpc-concurrency 参数，增加 grpc 线程数量
    * 调大 racksdb cache


sysbench oltp 性能瓶颈定位就说这些，实际情况中 tidb 的瓶颈并不是一成不变的，需要根据实际情况去优化调整，要不然就不叫玄学了😄。

## sysbench insert 测试瓶颈定位

sysbench insert 脚本很简单，就是顺序写，那么我们先从 tikv 基本概念考虑，tikv 内的最小几本单元是 region, 并且是每个表肯定是对应这个不同的 region，那么加入我们的测试只正对一个表 insert，我们有再多的 tikv 也是没有卵用，（对这个问题，目前有相应的解决办法，具体我也不是太清楚）所以我们测试要保证表的数量一定要大于 tikv 节点的数量（当然越多越好了），这里实验的时候我是使用了 16个表，每个表预先 prepare 了 100w 的数据。

运行 insert 脚本，同样我会先 top 看 tidb ／ tikv ／pd 的 cpu 使用情况。
我们会很容易的发现 insert 的时候我们 TiDB 的 CPU 并不会首先成为瓶颈（并不绝对），但是发现 TiKV 的压力还是很大的，我们就需要去定位 TiKV 的瓶颈，如果 TiKV 压力也是很小，那么我们就要看一下是不是磁盘 IO 存在问题，查看磁盘 IO ，正常来说查看磁盘情况我使用 iostat

![](http://7xnp02.com1.z0.glb.clouddn.com/Screen%20Shot%202018-04-08%20at%209.58.10%20AM.png)

我们可以看到 sde 盘的 %util 在 70，也就是说挺忙的，但是还可以还没有达到瓶颈，正常来说超过了 90 以上，就可以认为这块盘已经被我们压榨的差不多了。

> 划水了，insert 测试的详细定位过程后面在单独写一篇吧

### Summary

* 瓶颈
    * TiKV raftstore 线程
    * 磁盘 IO ，出现 stall 或是出现很多 IO wait
* 优化
    * 如果在磁盘没有达到瓶颈的前提下，TiKV 机器的 CPU 还很富裕，可以单台多部署，一般单个 TiKV 实例对应一块盘
    * 磁盘 IO 出现瓶颈，可以调整压缩方式，用 CPU 资源换取 IO 资源
    * 发现系统的 IO 压力不大，但是 CPU资源已经吃光了，top -H 发现有大量的 bg 开头的线程（RocksDB 的 compaction 线程）在运行，这个时候可以考虑减少压缩用 IO 资源换取 CPU 资源

## 一点小提示

* Sysbench 版本建议使用 1.0 及以上版本
* 当集群 TiKV 节点数量大于 3 的时候，在 prepare 数据完成后，需要等待 TiKV 节点上的 * leader 以及 region 数量均匀分布后，在进行各种基准测试，可以通过pd-ctl 或是 http://hosts:port/pd/api/v1/stores 或是 Grafana 监控查看各个 TiKV 节点上 leader 和 region 的分布情况。
* 当进行多轮测试的时候，当需要清理数据的时候，不要执行 DROP 操作，可以使用 Ansible 进行物理删除，然后在重启集群，重新 prepare 数据进行测试。
* 当 TiDB , TiKV 都没有达到瓶颈的时候, 可以不断增大 Sysbench 线程数量

## 最后说一点

以上只是一个小的示例，在进行实际性能测试的过程中，总是会出现各种各样的情况，出现的瓶颈的地方也不是一成不变的，我所谓的玄学并不是没有根据的胡乱调整，而是分析测试实际情况加上自己对 TiDB 系统的理解，进行合理调整部署拓扑以及配置参数。当然如果对 TiDB / TiKV / PD 程序的优化感兴趣，我们也可以很好的利用压测 + Grafan 监控 + Perf + [FlameGraph](https://github.com/brendangregg/FlameGraph) 定位程序内部的瓶颈。


## 参考

1. [工欲性能调优，必先利其器（1）](https://pingcap.com/blog-cn/tangliu-tool-1/)
2. [工欲性能调优，必先利其器（2）- 火焰图](https://pingcap.com/blog-cn/tangliu-tool-2/)
3. [三篇文章了解 TiDB 技术内幕 - 说存储](https://pingcap.com/blog-cn/tidb-internal-1/)
4. [sysbench 的一点整理](http://int64.me/2017/sysbench%E7%9A%84%E4%B8%80%E7%82%B9%E6%95%B4%E7%90%86.html)
5. [TiKV 性能参数调优](https://github.com/pingcap/docs-cn/blob/master/op-guide/tune-tikv.md)
