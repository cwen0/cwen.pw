title: MapReduce 笔记 - MIT-6.824
author: cwen
date:  2017-08-04
update:  2017-08-04
tags:
    - 分布式
    - MapReduce
    - MIT-6.824

---

MapReduce 由google提出的软件架构，主要用于大规模数据集的并行计算... <!--more-->

## MapReduce 概念

* 背景：在几个小时内处理晚TB级别的数据量，eg: 分析一个爬行网也的图形结构，由非分布式系统专家开发的程序运行成千的机器上会是一件很痛苦的事情， eg：错误处理
* 总体目标：非专业程序员可以轻松的在合理的效率下解决的巨大的数据处理问题。程序员定义Map函数和Reduce函数、顺序代码一般都比较简单。 MR在成千的机器上面运行处理大量的数据输入，隐藏全部分布式的细节。

## MapReduce 抽象

输入是被切分成 M 个分片

```
Input1 -> Map -> a,1   b,1  c,1
Input2 -> Map ->     b,1
Input3 -> Map -> a,1      c,1
                  |   |   |
                  |   |    -> Reduce -> c,2
                  |    ----> Reduce -> b,2
                    ------> Reduce -> a,2
```

MR 在每个输入分片上调用 Map() 函数，产生(k2, v2)这样的中间数据集。  每一个Map()函数的调用就是一个 "task"
MR 收集所有key为k2的所有值, 并且把他们传递给Reduce 调用， 最后Reduce输出是这样<k2, v3>这样的一对数值，并存储到输出文件中。

## Example: 单词计算

输入是成千文本文件

```
 Map(k, v)
    split v into words
    for each word w
      emit(w, "1")
  Reduce(k, v)
    emit(len(v))
```

## MapReduce 隐藏了很多让人痛苦的细节

* 在服务器上启动软件(s/w)
* 跟踪哪些"tasks"已经完成
* 数据传送
* 失败恢复

## MapReduce 易拓展

N 台计算机具有 Nx 的吞吐量，假设 M 和 R 都是 大于等于N (即，大量的输入文件和输出的keys), 因为每个Map() 互不影响，所用Map() 函数可以并发的执行。 Reduce() 同样如此。
所以我们可以通过买更多的电脑来增加吞吐。

## 什么会限制性能？

这是我们关系去优化的地方？ CPU? memory? disk? network?
MepReduce 的作者在 2004 时候，网络带宽是个大问题。关键所有Map()， Reduce() 交互过程中的所有数据都是经过网络的，网络速度是远小于磁盘和内存的速度。 所有当初作者尽量减少网络搬迁数据（如今网络速度相比2004年快了很多)

## 更多细节

* master: 分发"task"给 workers;
* 记录m Map task 的中间输出文件、r Reduce
* 输入文件是被存储在 GFS，每个Map 输出文件都有三份;
* 所有计算机都同时运行这GFS 和 MR workers;
* 输入文件是比 workers 多；
* master 给每个 worker 一个 map task, 只有当老的 task 完成后 master 才会分发新的任务
* map worker 在本地磁盘上使用 hash 算法将 中间 key 分成 R 份
* 直达所有的Map tasks 全部完成，才会调用 Reduce
* master 通知 Reducers 从 Map workers 去回去中间数据分区
* Reduce worker 将最终结果写入GFS(每个 Reduce task 产生一个 文件)

## 如何设计去减少慢网络的影响

* Map worker 的输入是从本地磁盘上的GFS副本读取，不经过网络
* 中间数据只经过一次网络
* Map worker 写数据到本地磁盘，而不是 GFS
* 每个中间数据切分成的文件中都包含许多keys
* 大的网络传输是更加有效率

## 如何更好的负载均衡

让n-1 servers 去等待一个server 的结束，这是非常糟糕的， 但是总有 tasks 是别其他的tasks 运行的时间要久。
解决办法： tasks 的数量要比 workers 多。 master 检测到 workers 的老的tasks 执行结束后， 给他分配新的 task。所以没有比这个 worker自己所能允许的最大时间还长的task存在(希望这样) 。所以运行速度快的 servers 比运行速度慢的servers做更多的工作，但是能在同时完成

## worker 失败恢复的细节(如何做容错 ）

### Map worker 崩溃

* master 检查到 worker 不在相应心跳
* 崩溃的 worker 的 map 中间输出数据丢失，但是可能每一个 Reduce task 都需要这个数据
* master 重新调度，在GFS拥有输入文件的其他副本的计算机上重新启动 task
* 有些 Reduce worker 可能已经拥有了读到了崩溃掉机器上的中间数据，在这里我们依赖 map 函数的功能和确定性
* 如果Reduce 以及获取到所有中间数据，那么master不需要重新运行 Map

### Reduce worker 崩溃

* 已经执行结束的 Tasks 是没有问题的- 别存储到了 GFS，带有多个副本
* master 在其他 workers 上重启崩溃掉的 worker 没有完成的 task

### Reduce worker 在正在写他的输出文件的时候崩溃掉

GFS会自动重命名输出，然后使其保持不可见直到Reduce完成，所以master在其他地方再次运行Reduce worker将会是安全的。

## 其他错误和问题

* 如果 master 给两个 workers 同样的 Map() task 肿么办？

可能 master 错误的认为其中一个 worker 以及死亡， 它只会告诉Reduce worker其中的一个

* 如果 master 给两个 workers 同样的 Reducer() task 肿么办？

它们都会将同一份数据写到GFS上面，GFS的原子重命名操作会触发，先完成的获胜将结果写到GFS.

* 如果某一个单独的 worker 是非常的慢 -- 一个掉队者？

 产生原因可能是非常糟糕的硬件设施。 master会对这些最后的任务创建第二份拷贝任务执行。

* 如果由于硬件或是软件原因造成一个worker计算出一个错误的输出肿么办？

 太糟糕了！MR假设是建立在"fail-stop"的cpu和软件之上。

* 如果 master 崩溃了肿么办？

单独的 master 挂了， 那就挂了

## 关于那些MapReduce不能很好执行的应用

* 并不是所以工作都适合map/shuffle/reduce这种模式
* 小的数据，因为管理成本太高,如非网站后端
* 大数据中的小更新，比如添加一些文件到大的索引
* 不可预知的读(Map 和 Reduce都不能选择输入)
* Multiple shuffles, e.g. page-rank (can use multiple MR but not very efficient)
* 多数灵活的系统允许MR，但是使用非常复杂的r模型

## 总结

MapReduce 的出现使得大数据计算变得流行起来

* 不是最有效或是灵活的
* 拓展性好
* 容易编程 -- 失败和数据迁移被隐藏起来


## 参考

1. [MIT-8.624 nodes](https://pdos.csail.mit.edu/6.824/notes/l01.txt)
2. [MapReduce: Simplified Data Processing on Large Clusters](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)





