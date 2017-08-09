title: RPC in GO - MIT-6.824
author: cwen
date:  2017-08-10
update:  2017-08-10
tags:
    - RPC
    - 网络
    - MIT-6.824
    - go
---

RPC 理想上想把网络通信做的跟函数调用一样 <!--more-->

## 远程调用 (RPC)

* 分布式系统的关键模块
* 目的：
    * 容易编写网络通信程序
    * 隐藏大多数 client/server 之间通信的细节
    * 客户端调用更加像传统的过程调用
    * 服务端处理更加像传统的过程调用
* RPC 已经被广泛应用

## RPC 理想上想把网络通信就当做函数调用一样简单

* Client:

```
    z = fn(x,y)
```

* Server:

```
    fn(x, y) {
        compute
        return z
    }
```

RPC 的目标是这样的水平透明

## Go example [kv.go](https://pdos.csail.mit.edu/6.824)

* client "dial" 向 server 端请求调用就像寻常函数调用一样
* server 并发的处理每一个请求，当然，对于keyvalue要用到锁

## RPC消息流程图：

```
Client             Server
	request--->
   		<---response
```

## 软件架构

```
client app         handlers
	stubs           dispatcher
RPC lib           RPC lib
 　　net  ------------ net
```

## 一些细节

* 应该调用哪个服务器函数（handler）？

去掉用 Call() 中指定的函数

* 序列化数据： 数列换数据到包中
    * 棘手的数组，指针，对象等。
    * Go的RPC库非常强大。
    * 有些东西你不能传递：比如channels和function。

* 绑定：client 如何知道该和谁交互
    * client 也许使用 server host name
    * 也许使用命名服务，将服务名字映射到最好的服务器。

## RPC 问题： 当遇到失败会做一些什么操作？
    eg: 丢包，网络中断, server 响应缓慢，server 端挂掉

## 错误对RPC客户端意味着什么?

* client 永远不会收到 server 的响应
* clinet 不知道 server 是否收到请求(可能在server 发送响应的时候网络中断)

## 简单的方案：“最少一次” 执行

* RPC client 等待响应一定时间，在这段时间内没有收到响应，则重新发送请求，持续这样的操作一定次数后，依然吗没有响应，则向应用汇报错误

## Q: "至少一次"容易被应用程序处理吗？

* 至少一次写的简单问题： 客户端发送"deduct $10 from bank account"

## Q: 客户端可能出现什么错误？

* Put("k",10) -- 一个RPC调用在数据库服务器中设置键值对。
* Put("k",20) -- 客户端对同一个键设置其他值。

## Q: "至少一次" 是否是正确的？

* 如果只是读操作没有问题
* 如果应用对于重复写做了处理，也是OK 的

## 更好的RPC行为："最多一次"

* idea:服务器的RPC代码发现重复的请求，返回之前的回复，而不是重写运行。
* Q：如何发现相同的请求?

client让每一个请求带有唯一标示码XID(unique ID),相同请求使用相同的XID重新发送。
server：

```
if seen[xid]:
    r = old[xid]
else
    r = handler()
    old[xid] = r
    seen[xid] = true
```

## "最多一次" 的复杂度

* 如何确保 XID 是唯一的？
    * 很大的随机数?
    * 将唯一的客户端ID（ip address？）和序列号组合起来？

* server 最后必须丢弃老的 RPC 信息
    * 什么时候的确是安全的？
    * 想法：
        * 唯一的client  IDs
        * 前一个rpc请求的序列号
        * 客户端每个 RPC 请求都包括 "seen all replies <= x"
        * 类似tcp中的seq和ack
    * 或者每次只允许一个RPC调用，到达的是seq+1，那么忽略其他小于seq
    * 客户端最多可以尝试5次，服务器会忽略大于5次的请求
* 当原来的请求还在执行，怎么样处理相同seq的请求？
    * 服务器不想运行两次，也不想回复。
    * 想法：给每个执行的RPC，pending标识；等待或者忽略。

## 如果一个 "最多一次" 的server挂掉了或是重启了肿么办？

* 如果服务器将副本信息保存在内存中，服务器会忘记请求，同时在重启之后接受相同的请求。
* 也许，你应该将副本信息保存到磁盘？
* 也许，副本服务器应该保存副本信息？

## "至少执行一次" 如何？
* 至多一次+无限重试+容错服务

## Go RPC实现的”最多一次“？

* 打开TCP连接
* 向TCP连接写入请求
* TCP也许会重传，但是服务器的TCP协议栈会过滤重复的信息
* 在Go代码里面不会有重试（即：不会创建第二个TCP连接）
* Go RPC代码当没有获取到回复之后将返回错误
    * 也许是TCP连接的超时
    * 也许是服务器没有看到请求
    * 也许服务器处理了请求，但是在返回回复之前服务器的网络故障

## 参考

1. [MIT-8.624 nodes](https://pdos.csail.mit.edu/6.824/notes/l-rpc.txt)
2. [Distributed-Systems](https://github.com/feixiao/Distributed-Systems)

